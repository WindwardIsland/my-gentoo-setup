# Setup

The steps below should be followed after cloning this repository.

Ideally, this setup should be completed during installation.
## Portage

Copy my Portage configurations in `portage/` to `/etc/portage` with the following command:
```
(chroot) livecd # cp -a portage/ /etc/
```

## UEFI Post-Kernel Configuration Hook
On my UEFI system, I personally use [`efibootmgr`](https://wiki.gentoo.org/wiki/Efibootmgr) instead of secondary bootloaders such as GRUB, systemd-boot, rEFInd, etc. It's less overhead for me to manage, and I don't really need a secondary bootloader anyway since I'm only booting into one entry all the time (that being Gentoo). To avoid having to specify the kernel version part of the initramfs and vmlinuz EFI files when creating a boot entry, I have a hook that automatically renames them to fixed names.

Copy the hook (`99-vmlinuz-initramfs-fixed`) into `/etc/kernel/postinst.d`:
```
(chroot) livecd # cp 99-vmlinuz-initramfs-fixed /etc/kernel/postinst.d
```
Generate the EFI files (depending on the kernel used):
```
(chroot) livecd # emerge --config sys-kernel/gentoo-kernel{-bin}
```
## Packages List
These are the packages that I usually install during the Gentoo installation process. To avoid typing them all out as part of `emerge`, the following command can be run as a shorthand:
```
(chroot) livecd # xargs -d '\n' -a package_list emerge
```

## s6 init system setup
> [!TIP]
> This assumes that you are switching inits *during* the installation process. It is much easier to switch than *after* installation as you would have less packages installed, less service scripts installed, and overall less overhead to deal with than a fully installed system.

### Installation
If not yet done, add my [overlay](https://github.com/WindwardIsland/windwardisland-gentoo-overlay) with the following commands:
```
(chroot) livecd # eselect repository add windwardisland git https://github.com/WindwardIsland/windwardisland-gentoo-overlay.git
(chroot) livecd # emaint sync -r windwardisland
```

This overlay contains the *latest* s6-related packages from skarnet.org that aren't available in the Portage tree yet. There are also custom patches and settings applied to these packages that were taken from Artix Linux's s6 flavor.

First, unmerge and deselect OpenRC and SysVinit from the system:
```
(chroot) livecd # emerge -C net-misc/netifrc sys-apps/openrc sys-apps/sysvinit
```

Then, install the s6 supervision suite:
```
(chroot) livecd # emerge s6 s6-rc s6-linux-init s6-frontend
```

Create a symbolic link to `/sbin/init`:
```
(chroot) livecd # ln -sf /usr/bin/s6-init /sbin/init
```

Be sure to regenerate your initramfs afterwards as it will contain the *old* link to `/sbin/init`! Make sure that when regenerating, there is some indication that the initramfs is aware of the new link. For instance, ugRD would indicate this with `init_target: /usr/bin/s6-init` (notice how the target file and not the link is shown).

### System Services
Setup s6 system services (basic and necessary longruns and oneshots) using the `s6_setup` script inside `s6/`:
```
(chroot) livecd # cd s6
(chroot) livecd # ./s6_setup base
(chroot) livecd # ./s6_setup system
```

The s6 supervision suite uses a [repository](https://skarnet.org/software/s6-rc/repodefs.html) to manage services. This will not be gone into detail here; please consult [official documentation](https://skarnet.org/software) for more information. We can set up the repository and initialize the system services with the following commands:
```
(chroot) livecd # s6 repo init
(chroot) livecd # s6 repo list
```

If the previous command didn't output anything, then we need to create a set in the repository containing all our services and their startup states/prescriptions (for defaults, the set name will be `current`):
```
(chroot) livecd # s6-rc-set-new -r /etc/s6/repo current
```

We can list `current`'s services and their prescriptions, and then enable/disable certain ones of our choosing:
```
(chroot) livecd # s6 set status
(chroot) livecd # s6 set enable/disable <servicenames>
```

Finally, we need to commit the set (i.e. save the changes), and install it. These commands compile a database containing all our services, and copy it over to a directory where it will be read at boot time:
```
(chroot) livecd # s6 set commit
(chroot) livecd # s6 live install --init
```

Note that when editing services in our fully installed system, use `s6 repo sync` to synchronize the repository, and then the following `s6 set`/`live` commands. In addition, remove the `--init` option from `s6 live install`. The option should *only* be used when services aren't managed with s6-rc yet, which is appropriate during installation.

Some system services create logs owned by the `s6log` user and group. We can create this user with the following command:
```
(chroot) livecd # useradd -U -r -s /sbin/nologin -d /dev/null s6log
```

The `s6log` user and group will automatically be created every time due to the `sysusers` system service. This service will look in `/usr/lib/sysusers.d` and source the `s6log.conf` file that was copied over earlier with `s6_setup base`.

> [!IMPORTANT]
> The installed Portage hook (`s6/s6-sv-hook`) will automatically search the Artix repos for an s6 service script corresponding to the *packages specified* when running emerge. This includes any dependencies. It is **your responsibility to check** the service stores (located at `/etc/s6/sv` and `/etc/s6/adminsv`) for any extra services that you do not want! Be sure to remove them as `s6 repo sync` will fail if any services have dependencies that are non-existent (due to the corresponding packages themselves not being installed).

### User Services
Make sure this repository is copied over to a location writable by a non-root user from where it was originally during installation. You may also clone this repository again after installation.

> [!IMPORTANT]
> This will setup user services for an **Xorg** installation. If you are using Wayland, feel free to edit the below script to your liking.

Run the setup script, specifying the appropriate non-root user:
```
$ cd s6
$ ./s6_setup user <non-root user>
```
Go to the location where the user services are installed (default: `~/.config/s6`). Run the following commands to initialize the repository and create the database:
- `$ export S6_CONF="${HOME}/.config/s6/s6-frontend.conf"`
- `$ s6 repo init`
- `$ s6 repo list`
- If the previous command returned nothing: `s6-rc-set-new -r "${HOME}/.config/s6/repo" current`
- `$ s6 set status`
- `$ s6 set enable/disable <servicenames>`
- `$ s6 set commit`
- `$ s6 live install --init`

Log out (or reboot), and all the enabled user services should've started successfully!


## Sources
- [Gentoo Wiki on Portage hooks](https://wiki.gentoo.org/wiki/Handbook:Parts/Portage/Advanced#Hooking_into_the_emerge_process)
- [Gentoo Dev docs on ebuild phase hooks](https://dev.gentoo.org/%7Ezmedico/portage/doc/portage.html#config-bashrc-ebuild-phase-hooks)
- [The available ebuild phases to build hooks around](https://dev.gentoo.org/%7Ezmedico/portage/doc/portage.html#package-ebuild-phases)
- [Gentoo Wiki on `/etc/portage/bashrc`](https://wiki.gentoo.org/wiki//etc/portage/bashrc)
