# Steam Deck - A Permanent Modification Mechanism

This repo documents and provides an example set of extensions that utilize the 
`systemd-sysext` mechanism. This mechanism can be used to create permanent system
modifications that supoort filesystem overlays and automatically enabled systemd unit files.

## systemd-sysext

The mechanism this repo provides is little more than a supplement to systemd's builtin `systemd-sysext` mechanism. The primary addition this mechanism adds is a way to automatically load systemd units from installed `systemd-sysext`extensions, whereas normal extensions require users to manually enable any units they wish to use, which won't survive upgrades.

For documentation on how to build systemd-sysext extensions, please see here: https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html

The rest of this README.md will focus on the specific differences needed to use this extension system wrapper.

## steamos-extension-loader

The only required extension is provided by `steamos-extension-loader`.

To install the loader, place the `steamos-extension-loader.raw` file in `/var/lib/extensions`.

Next, get into a terminal session and run as root:

```
systemctl enable --now systemd-sysext.service
systemctl enable --now steamos-extension-loader-installer.service
```

## How it Works

The `steamos-extension-loader` has two purposes:

1. It makes sure that system updates do not uninstall the service and supporting files used by it.
2. It installs a boot service that loads any other installed extensions by making sure the appropriate unit files are enabled and running.

### Persistence

The `steamos-extension-loader` maintains its persistence in a fairly straightforward way. First, it checks `/etc/steamos-extension-loader`, `/etc/systemd/system/steamos-extension-loader.service` and `/etc/atomic-update.d/steamos-extension-loader.conf`, ensuring they have identical checksums to the files packaged in the extension. If they don't exist or have mismatching checksums, it copies the files it bundles into those locations.

Secondly, it enables and runs the `steamos-extension-loader.service` unit if it is not already enabled and active.

This service, among other things, will start and enable `steamos-extension-loader-installer.service` and `systemd-sysext.service`, thus ensuring that updates to the loader get copied up into `/etc`.

Finally, the file placed in `/etc/atomic-update.d` ensures that none of the installed files are lost after a system update.

If persistence is ever lost, it should be enough to just repeat the installation commands.

### Unit Files

The loader, in addition to persistence, also ensures that services from other extensions are enabled and loaded. This is the advantage of `steamos-extension-loader`, as `systemd-sysext` provides no equivalent mechanism (at least as far as this author was able to determine, *please* correct me if I have overlooked anything here).

The algorithm it uses to enable units is very straightforward:

1. Any system unit file starting with `steamos-extension-` is passed to `systemctl preset`. After that, it checks if the unit is enabled. If it is enabled, and it is not yet running, `steamos-extension-loader.service` starts the unit. This allows extension authors to decide which services and timers should be loaded by providing a correct systemd-preset file.

2. User unit files are treated differently. Systemd does not have an equivalent to systemd preset for user units, thus every single unit is simply passed to `systemctl enable --global`, so that they will be loaded during logon. To control which units are running, you must ensure a correct install target. If you have a service fired by a timer that shouldn't run otherwise, omit the entire `[Install]` section in the unit file.

### System Updates

The steamos update mechanism does not like systemd-sysext to be running, as it creates a read-only overlayfs on `/usr`. To solve this problem, `systemd-sysext.service` unloads itself when `rauc.service` (the update service) is started. Unfortunately, `rauc.service` does not unload itself until reboot, which means all extensions are unloaded until reboot. Updates that occur during boot-up do not conflict with systemd-sysext, as they occur earlier in the boot process.

## Extensions

Installing additional extensions is as easy as placing them in `/var/lib/extensions` and rebooting.

A number of example extensions that I personally use are included in this repo, with explanations of what they do in the following sections.

### steamos-extension-balance-btrfs

This extension ensures that any mounted btrfs filesystem has their `bg_reclaim_threshold` set to non-zero, whoch causes them to automatically balance themselves.

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

### steamos-extension-performance-tuning

This extension applies various performance tuning changes. Additionally, it installs a daemon that will change the CPU governor, NVMe parameters, etc. when the system transitions from on AC power to off AC power and vice-vsersa. When on AC power everything is pinned to a maximum performance setting. When off AC power, swttings are pinned to values that should give a good balance between performance and power savings.

This extension changes some kernel command line parameters and will cause an additional reboot after updates are applied. When used together with disable-mitigations, only one additional reboot will occur, not two.

### steamos-extension-update-btrfs

If you use `steamos-btrfs`, this extension will automatically update it on a schedule.

### steamos-extension-update-decky-loader

If you use decky loader, this extension will automatically update it when an update is available. Be warned, it only supports the stable channel, and can't update plugins. Additionally, whenever an update occurs, the steam client will restart, returning you to the main menu. Your game will still be running and accessible.

### steamos-extension-update-flatpak

This extension repairs and updates all installed user and system flatpaks on a schedule. It also removes any unused dependencies after updates.

### steamos-extension-zram

This extension hijacks the steam deck's zram configuration in an obtuse way. I'm not sure I'd recommend anyone else use it, it is almost a toy.

This extension sets up a zram based swap that uses a third of the systms ram allocation, then it creates a second zram device with ext4 that is mounted in /home/deck/.zram and uses another third. Then, it bind mounts various directories (e.g., ~/.cache, the steam client appcache, the decky loader log direcetories, ...) into that zram device so that things like mesa cache updates get to skip the disk when writing. On shutdown, it synchronizes the ram cache to disk, and on next boot, will only buffer the changed files. I've seen this use around max 600MiB RAM, but after the cache is hot, it generally seems to cap out around 100MiB.


The motivation for this extension was that btrfs seemed to cause hangs under heavy write loads, which would cause games to hitch for a second when other games were being updated. This was an attempt to alleviate that.

This extension significantly slows down shutdowns and system updates, as they have to wait for the ram cache to synchronize to disk first.

### steamos-extension-retain-boot

This extensiom sets SteamOS as the next boot entry after each reboot. This can be useful when dual booting if the other OS likes to mess with the bootorder.
