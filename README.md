# A SteamOS Extension System and a Collection of SteamOS Extensions

This repo documents and provides an example set of extensions that utilize the 
`systemd-sysext` mechanism. This mechanism can be used to create permanent system
modifications that support filesystem overlays and automatically enabled systemd unit files.

## HoloISO Support

Most of these extensions have been tested to function against HoloISO. HoloISO is close enough to SteamOS in implementation that minimal levels of support code is needed to target both.

All of these extensions assume that the player's username is `deck`, and a few won't function on HoloISO installs using a different username. Specific details about this are mentioned in the individual extension's documentation at the end of this README.

## systemd-sysext

The mechanism this repo provides is little more than a supplement to systemd's built-in `systemd-sysext` mechanism. The primary addition this mechanism adds is a way to automatically load systemd units from installed `systemd-sysext` extensions, whereas normal extensions require users to manually enable any units they wish to use, which won't survive upgrades.

For documentation on how to build systemd-sysext extensions, please see here: https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html

The rest of this README.md will focus on the specific differences needed to use this extension system wrapper.

## Source Code

All the `.raw` files are just squashfs images. They can be extracted by 7z or mounted on linux, and are thus self-documenting.

Regardless, their source code can also be interrogated [here](https://github.com/MiningMarsh/steamos-extension-examples).

## steamos-extension-loader

The only required extension is provided by `steamos-extension-loader.raw`.

To install the loader, place the `steamos-extension-loader.raw` file in `/var/lib/extensions`.

Next, get into a terminal session and run as root:

```
systemctl enable --now systemd-sysext.service
systemctl enable --now steamos-extension-loader-installer.service
```

## How it Works

`steamos-extension-loader.service` and `steamos-extension-loader-installer.service` have two purposes:

1. They make sure that system updates do not uninstall their services and supporting files.
2. They install themselves as a boot service that loads any other installed extensions by making sure the appropriate unit files are enabled and running.

### Persistence

`steamos-extension-loader-installer.service` maintains its persistence in a fairly straightforward way. First, it checks `/etc/steamos-extension-loader`, `/etc/systemd/system/steamos-extension-loader.service` and `/etc/atomic-update.d/steamos-extension-loader.conf`, ensuring they have identical checksums to the files packaged in the extension. If they don't exist or have mismatching checksums, it copies the bundled file into those locations.

Secondly, `steamos-extension-loader-installer.service` enables and runs the `steamos-extension-loader.service` unit if it is not already enabled and active.

`steamos-extension-loader.service`, among other things, will start and enable `steamos-extension-loader-installer.service` and `systemd-sysext.service`, thus ensuring that `steamos-extension-loader.raw` updates get copied up into `/etc`.

Finally, the file placed in `/etc/atomic-update.d` ensures that none of the installed files are lost after a system update.

If persistence is ever lost, it should be enough to re-run the installation commands.

### Unit Files

`steamos-extension-loader.service`, in addition to helping with persistence, also ensures that services from other extensions are enabled and loaded. This is the advantage of `steamos-extension-loader.service`, as `systemd-sysext.service` provides no equivalent mechanism (at least as far as this author was able to determine; *please* correct me if I have overlooked anything here).

The algorithm it uses to enable units is very straightforward:

1. Any system unit file starting with `steamos-extension-` is passed to `systemctl preset`. After that, it checks if the unit is enabled. If it is enabled and it is not yet running, `steamos-extension-loader.service` starts the unit. This allows extension authors to decide which services and timers should be loaded by providing a correct systemd-preset file.

2. User unit files are treated differently. Systemd does not have an equivalent to systemd preset for user units; thus, every single unit is simply passed to `systemctl enable --global`, so that they will be loaded during logon. To control which units are running, you must ensure a correct install target. If you have a service fired by a timer that shouldn't run otherwise, omit the entire `[Install]` section in the unit file. User units also need to start with `steamos-extension-` to be considered for loading.

### System Updates

The SteamOS update mechanism does not like `systemd-sysext.service` to be running, as it creates a read-only overlayfs on `/usr`. To solve this problem, `systemd-sysext.service` unloads itself when `rauc.service` (the update service) is started. Unfortunately, `rauc.service` does not unload itself until reboot, which means all extensions are unloaded until reboot. Updates that occur during boot-up do not conflict with `systemd-sysext.service`, as only steam client updates can apply during boot-up.

## Extensions

Installing additional extensions is as easy as placing them in `/var/lib/extensions` and rebooting.

Extensions can be uninstalled by removing their extension file from `/var/lib/extensions`, and rebooting. It is safe to leave the loader installed even if all extensions are uninstalled, it just won't do anything.

Some extensions may change grub boot options in order to add kernel parameters. You can permanently uninstall the changes they make by removing the extension, swapping from stable to beta branch or vice versa, and then switching back. SteamOS will regenerate the boot configuration during upgrades, overwriting the changes the extensions made. Make sure that the extension file is removed before the changes in branch take place.

A number of example extensions that I personally use are included in this repo, with explanations of what they do in the following sections.

### steamos-extension-clean-games

This extension automatically removes any game directory from Steam's common install path if they are not tracked by an installed package (game). It also removes any shadercache for missing packages, preventing size buildup of shadercache.

### steamos-extension-compat-tools

This extension regularly installs and updates Boxtron, Luxtorpeda, Roberta, and Proton GE. It always installs the latest version, and it changes their labels to "DosBox", "Source Ports", "ScummVM", and "Proton GE" respectively. In particular, you can set games to use "Proton GE" by default, and they will always use the latest version.

### steamos-extension-disable-mitigations

This extension adds `mitigations=off` to SteamOS' boot config. It is debatable whether this improves performance, so treat this extension with caution. This also *definitely* makes your installation less secure.

I recommend only using this extension if you understand spectre-like vulnerabilities and can perform your own risk and threat assessment.

This extension will cause an additional reboot after updates are applied. When used together with performance-tuning, only one additional reboot will occur, not two.

### steamos-extension-nohang

This extension installs `nohang` to help minimize system latency in low memory conditions.

### steamos-extension-prelockd

This extension installs `prelockd` to help minimize system latency by preventing executable memory-mapped pages from being swapped.

### steamos-extension-thermal-tdp

This extension bundles a daemon that automatically sets the system TDP limit to 20w, and lowers it slowly back down to 15w based on system temperature.

This should allow bursty games to run at 20w, while keeping sustained loads at 15w to prevent overheating.

This utility should not be used on any hardware except the steam deck! Most likely, it won't do anything, however, there is a small possibility that this daemon could set inappropriate TDP values for some other AMD SoC than the steam deck.

### steamos-extension-performance-tuning

This extension applies various performance tuning changes. Additionally, it installs udev rules that will change the CPU governor, NVMe parameters, etc. when the system transitions from on AC power to off AC power and vice versa. When on AC power, everything is pinned to a maximum performance setting. When off AC power, settings are pinned to values that should give a good balance between performance and power savings.

This extension changes some kernel command line parameters and will cause an additional reboot after updates are applied. When used together with disable-mitigations, only one additional reboot will occur, not two.

This extension does not apply kernel command line performance tweaks on HoloISO if it is detected. The kernel parameters caused boot issues for this author, and have been set to only apply to SteamOS, where they function correctly.

### steamos-extension-update-btrfs

If you use `steamos-btrfs`, this extension will automatically update it on a schedule.

Don't use this extension with HoloISO. `steamos-btrfs` likely doesn't function correctly against HoloISO.

### steamos-extension-update-decky-loader

If you use Decky Loader, this extension will automatically update it when an update is available. Be warned, it only supports the stable channel, and can't update plugins. Additionally, whenever an update occurs, the Steam client will restart, returning you to the main menu. Your game will still be running and accessible.

Note that this extension cannot perform the initial Decky Loader install. The Decky Loader installation scripts do not appear to function for fresh installs when invoked directly from a root context.

This extension only functions on HoloISO of the player's username is `deck`.

### steamos-extension-update-flatpak

This extension repairs and updates all installed user and system flatpaks on a schedule. It also removes any unused dependencies after updates.

### steamos-extension-zram

This extension hijacks the Steam Deck's zram configuration in an obtuse way. I'm not sure I'd recommend anyone else use it; it is almost a toy.

This extension sets up a zram-based swap that uses a third of the system's RAM allocation, then it creates a second zram device with ext4 that is mounted in /home/deck/.zram and uses another third. Then, it bind mounts various directories (e.g., ~/.cache, the Steam client appcache, the Decky Loader log directories, ...) into that zram device so that things like mesa cache updates get to skip the disk when writing. On shutdown, it synchronizes the RAM cache to disk, and on the next boot, will only buffer the changed files. I've seen this use around max 600 MiB RAM, but after the cache is hot, it generally seems to cap out around 100 MiB.


The motivation for this extension was that btrfs seemed to cause hangs under heavy write loads, which would cause games to hitch for a second when other games were being updated. This was an attempt to alleviate that.

This extension significantly slows down shutdowns and system updates, as they have to wait for the RAM cache to synchronize to disk first.

This extension will only function on HoloISO if the player's username is `deck`.

### steamos-extension-retain-boot

This extension sets SteamOS as the next boot entry after each reboot. This can be useful when dual booting if the other OS likes to mess with the boot order.

This extensiom functions on HoloISO, but might choose the boot order incorrectly on systems dual booting HoloISO and SteamOS.

### steamos-extension-irqbalance

This extension installs and runs the `irqbalance` service, which automatically balances interrupts across CPU cores. It is configured to try and minimize the number of running cores in addition to balancing interrupts to strike a better balance between power consumption and performance.

### steamos-extension-preload

This extension  installs the `preload` service. `preload` attempts to minimize system latency by pre-caching commonly loaded files. A state file will be left at `/var/lib/preload.state` if you uninstall this extension. The state file is minimal in size (under a megabyte typically).
