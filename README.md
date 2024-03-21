## Table of contents

- [Table of contents](#table-of-contents)
- [Arch Linux on Asus ROG Zephyrus G14 (G401II)](#arch-linux-on-asus-rog-zephyrus-g14-g401ii)
- [Basic Install](#basic-install)
	- [Prepare and Booting ISO](#prepare-and-booting-iso)
	- [Networking](#networking)
	- [Format Disk](#format-disk)
	- [Create encrypted filesystem](#create-encrypted-filesystem)
	- [Create and Mount btrfs Subvolumes](#create-and-mount-btrfs-subvolumes)
	- [Create a btrfs swapfile and remount subvols](#create-a-btrfs-swapfile-and-remount-subvols)
	- [Install the system using pacstrap](#install-the-system-using-pacstrap)
	- [Chroot into the new system and change language settings](#chroot-into-the-new-system-and-change-language-settings)
	- [Add btrfs and encrypt to Initramfs](#add-btrfs-and-encrypt-to-initramfs)
	- [Install Systemd Bootloader](#install-systemd-bootloader)
	- [Blacklist Nouveau](#blacklist-nouveau)
	- [Leave Chroot and Reboot](#leave-chroot-and-reboot)
- [Finetuning after first Reboot](#finetuning-after-first-reboot)
	- [Enable Networkmanager](#enable-networkmanager)
	- [Install Zsh](#install-zsh)
	- [Create a new user](#create-a-new-user)
	- [Update the system](#update-the-system)
- [Install Desktop Environment](#install-desktop-environment)
	- [Get X.Org and KDE Plasma](#get-xorg-and-kde-plasma)
	- [Oh-My-ZSH](#oh-my-zsh)
	- [Setup Plymouth for nice Password Prompt during Boot](#setup-plymouth-for-nice-password-prompt-during-boot)
- [Customizations](#customizations)
	- [Install asusctl tool](#install-asusctl-tool)
	- [Install ROG Kernel](#install-rog-kernel)
	- [Nvidia](#nvidia)
	- [Switch Profile On Charger Connect](#switch-profile-on-charger-connect)
	- [ROG Key Map](#rog-key-map)
	- [Change Fan Profile](#change-fan-profile)
- [Fixing Audio on Linux](#fixing-audio-on-linux)
	- [Mic Mute Key](#mic-mute-key)
- [Setup Automatic Snapshots for pacman](#setup-automatic-snapshots-for-pacman)
- [Install an AUR Helper](#install-an-aur-helper)
- [Installing Waydroid](#installing-waydroid)
- [Gamma Correction and Color profile](#gamma-correction-and-color-profile)
- [Miscellaneous](#miscellaneous)
	- [Fetch on Terminal Start](#fetch-on-terminal-start)
	- [Key delay](#key-delay)
 	- [QuickShare](#quickshare)


# Arch Linux on Asus ROG Zephyrus G14 (G401II)
Guide to install Arch Linux with btrfs, disc encryption, auto-snapshots, no-noise fan-curves on Asus ROG Zephyrus G14. Credits to [Unim8rix](https://github.com/Unim8trix/G14Arch), this guide is a fork of their guide.

![Screenshot_20240321_020632](https://github.com/k-amin07/G14Arch/assets/28199865/4f619b9e-a792-40ed-be7e-f3e9f17a4b8c)

## Basic Install

### Prepare and Booting ISO

Boot Arch Linux using a prepared USB stick. [Rufus](https://rufus.ie/en/) can be used on windows, [Etcher](https://www.balena.io/etcher/) can be used on Windows or Linux.

### Networking

For Network i use wireless, if you need wired please check the [Arch WiKi](https://wiki.archlinux.org/index.php/Network_configuration). 

Launch `iwctl` and connect to your AP like this:
* `station wlan0 scan`
* `station wlan0 get-networks`
* `station wlan0 connect YOURSSID` 

Type `exit` to leave.

Update System clock with `timedatectl set-ntp true`

### Format Disk

* My Disk is `nvme0n1`, check with `lsblk`
* Format Disk using `gdisk /dev/nvme0n1` with this simple layout:

	* `o` for new partition table
	* `n,1,<ENTER>,+1024M,ef00` for EFI Boot
	* `n,2,<ENTER>,<ENTER>,8300` for the linux partition
	* `w` to save layout


Format the EFI Partition

```
mkfs.vfat -F 32 -n EFI /dev/nvme0n1p1
```

### Create encrypted filesystem 

```
cryptsetup luksFormat /dev/nvme0n1p2  
cryptsetup open /dev/nvme0n1p2 luks
```

### Create and Mount btrfs Subvolumes

Create btrfs filesystem for root partition
```
mkfs.btrfs -f -L ROOTFS /dev/mapper/luks
``` 


Mount Partitions und create Subvol for btrfs. I dont want home, etc in my snapshots, so create subvol for them.
```
mount -t btrfs LABEL=ROOTFS /mnt
btrfs sub create /mnt/@
btrfs sub create /mnt/@home
btrfs sub create /mnt/@snapshots
btrfs sub create /mnt/@swap
```


### Create a btrfs swapfile and remount subvols

```
btrfs filesystem mkswapfile --size ${SWAP_SIZE} /mnt/@swap/swapfile
```


Replace `${SWAP_SIZE}` with the amount of swap space you want. Typically you should have the same amount of swap as RAM. So if you have 16GB of ram, you should have 16GB of swap space. Note that the size in GB is denoted with a G as a suffix and **NOT** GB.

([Source](https://btrfs.readthedocs.io/en/latest/Swapfile.html))

After that, just unmount with `umount /mnt/` and remount with subvolumes

```
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@ /dev/mapper/luks /mnt
mkdir -p /mnt/boot
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/btrfs

mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@home /dev/mapper/luks /mnt/home/
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots/
mount -o noatime,space_cache=v2,commit=120,subvol=@swap /dev/mapper/luks /mnt/swap/

mount /dev/nvme0n1p1 /mnt/boot/

mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvolid=5 /dev/mapper/luks /mnt/btrfs/
```


Check mountpoints with `df -Th` and enable swap file
```
swapon swapfile
```

### Install the system using pacstrap

```
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs nano networkmanager amd-ucode
```

After this, generate the filesystem table using 

```
genfstab -Lp /mnt >> /mnt/etc/fstab
``` 

Add swapfile 
```
echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab
```

### Chroot into the new system and change language settings
You can use a hostname of your choice, I have gone with zephyrus-g14.
```
arch-chroot /mnt
echo zephyrus-g14 > /etc/hostname
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo LANGUAGE=en_US >> /etc/locale.conf
ln -sf /usr/share/zoneinfo/Asia/Karachi /etc/localtime
hwclock --systohc
```

Modify `/etc/hosts` and add these entries. For static IPs, remove 127.0.1.1. Replace zephyrus-g14 with your hostname.

```
127.0.0.1		localhost
::1				localhost
127.0.1.1		zephyrus-g14.localdomain	zephyrus-g14
```

Edit `/etc/locale.gen` and uncomment the following line

```
en_US.UTF-8
```
Execute `locale-gen` to create the locales now

Add a password for root using `passwd root`


### Add btrfs and encrypt to Initramfs

Edit `/etc/mkinitcpio.conf` and add `encrypt btrfs` to hooks between block/filesystems.

```
HOOKS="base udev autodetect modconf block encrypt btrfs filesystems keyboard fsck
```

Also include `amdgpu` in the MODULES section and run `mkinitcpio -P`

### Install Systemd Bootloader

`bootctl --path=/boot install` installs bootloader

Edit the bootloader config using `nano`
```
nano /boot/loader/loader.conf
```
Replace the existing text with the following lines.

```
default	arch.conf
timeout	3
editor	0
```

Then, replace the contents of `/boot/loader/entries/arch.conf` with the following

```
title	Arch Linux
linux	/vmlinuz-linux
initrd	/amd-ucode.img
initrd	/initramfs-linux.img
```

Finally, copy boot-options with
```
echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):luks root=/dev/mapper/luks rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf
```


### Blacklist Nouveau

Edit `/etc/modprobe.d/blacklist-nvidia-nouveau.conf` using `nano` and add the following lines to it

```
	blacklist nouveau
	options nouveau modeset=0
```

### Leave Chroot and Reboot

Type `exit` to exit chroot and unmount all the volumes using
```
umount -R /mnt/
```
and reboot the system.

## Finetuning after first Reboot

After a successful reboot, we will perform some necessary initial setup.

### Enable Networkmanager
To connect to WiFi using `nmcli`, enable the network manager

```
systemctl enable NetworkManager
systemctl start NetworkManager
nmcli device wifi connect YOURSSID password SSIDPASSWORD
```

### Install Zsh
I prefer zsh as my default shell. Install it with pacman if not already installed
```
sudo pacman -S zsh zsh-completions
```

### Create a new user 

Create a new local user, add it to relevant groups and point it to zsh.

```
useradd -m -g users -G wheel,power,audio -s /usr/bin/zsh MYUSERNAME
passwd MYUSERNAME
```

Now enable root access for the user by adding it to `sudoers`. Edit `/etc/sudoers` and uncomment 

```
`%wheel ALL=(ALL) ALL`
```
Now `exit` and relogin as the newly created user.

### Update the system
Ensure that the system is up to date by running
```
sudo pacman -Syu
```

Install `acpid` and `dbus` and reboot the system.

```
sudo pacman -S acpid dbus 
sudo systemctl enable acpid
```


## Install a Desktop Environment

### Get X.Org and KDE Plasma

Install xorg and kde packages. I prefer `wayland` over X, however, it is a good idea to have X installed as well. 

```
sudo pacman -S xorg plasma plasma-workspace sddm 
```

Enable SDDM login manager

``` 
sudo systemctl enable sddm
```

Reboot and login to your new Desktop.


### Oh-My-ZSH

I like to use `zsh` with `oh-my-zsh` with `Powerlevel10K` theme. To install that, first install the required dependencies.

```
sudo pacman -S git curl wget
```

Get the necessary fonts
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
mkdir .local/share/fonts
cd .local/share/fonts
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
fc-cache -v
```

Clone `Powerlevel10K` repo
```
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`

Run zsh and it should prompt you to set up the `Powerlevel10k` theme. If your prompt configuration includes text on the right side of terminal, disable "Reflow Lines" option in `Konsole` settings -> edit current profile -> Scrolling to avoid jitter when resizing terminal.

### Setup Plymouth for nice Password Prompt during Boot

Install plymouth from official repos if not already installed.

Now modify the Hooks for the Initramfs, Plymouth must be right after "base udev".

```
HOOKS="base udev plymouth autodetect modconf block encrypt btrfs filesystems keyboard fsck
```

Run `mkinitcpio -P` 

Add some kernel-parameters to make boot smooth. Edit `/boot/loader/entries/arch.conf` and append to options

```
...rootflags=subvol=@ quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```


For Plymouth Theming and Options, check [Plymouth on Arch Wiki](https://wiki.archlinux.org/title/plymouth). Run the following command to set ROG logo as the plymouth theme.
```
sudo plymouth-set-default-theme -R bgrt
```

## Customizations
Use these customizations to get the most out of your system.
### Install asusctl tool

Add asus-linux repo to pacman as detailed [here](https://asus-linux.org/wiki/arch-guide/) and install the following tools

```
sudo pacman -Syu
sudo pacman -S asusctl
sudo pacman -S supergfxctl
sudo pacman -S rog-control-center
```
Enable these tools by running
```
sudo systemctl enable --now power-profiles-daemon.service
systemctl enable --now supergfxd
```

Run the following commands to set charge limit and enable Quiet, Performance and Balanced Profiles:
```
asusctl -c 85 		# Sets charge limit to 85% if you do not want this, do not execute this line
asusctl fan-curve -m Quiet -f cpu -e true
asusctl fan-curve -m Quiet -f gpu -e true 
asusctl fan-curve -m Performance -f cpu -e true
asusctl fan-curve -m Performance -f gpu -e true
asusctl fan-curve -m Balanced -f cpu -e true
asusctl fan-curve -m Balanced -f gpu -e true
```

For fine-tuning read the [Arch Linux Wiki](https://wiki.archlinux.org/title/ASUS_GA401I#ASUSCtl) or the [Repository from Luke](https://gitlab.com/asus-linux/asusctl). 


### Install ROG Kernel
After adding the above repo, install the ROG kernel by running
```
sudo pacman -S linux-g14 linux-g14-headers 
#kernel headers are very important otherwise nvidia module will not load, resulting in black screen.	
```
Then edit `/boot/loader/loader.conf` and add the following to it:
```
default arch-g14.conf
timeout 3
editor 0
```

Then run 
```
sudo cp /boot/loader/entries/arch.conf /boot/loader/entries/arch-g14.conf && sudo nano /boot/loader/entries/arch-g14.conf
```

Replace the lines that start with `title`, `linux`, and `initrd` with this:
```
title   Arch Linux (g14)
linux   /vmlinuz-linux-g14
initrd  /amd-ucode.img
initrd  /initramfs-linux-g14.img
```

and finally do 
```
sudo mkinitcpio -P
```

Reboot and select the `Arch Linux (g14)` entry from the boot menu.

### Nvidia
Install the following packages

```
sudo pacman -S nvidia-dkms acpi_call
```


### Switch Profile On Charger Connect
Plasma now supports various power profiles depending on battery status. Go to
```
KDE Settings -> Power Management -> Energy Saving
```
In `On AC Power` tab, set `Power Management Profile` to "Performance", in `Battery` tab, set it to "Balanced" and in `On Low Battery` set it to "Quiet"

**Optional:** Enable battery full charge notification. Go to KDE Settings -> Notifications -> Application Settings -> Configure Events. Select Charge Complete and Select Show a message in popup

### ROG Key Map
Go to KDE Settings->Shortcuts. Click `Add Application`, select `ROG Control Center` and add it. Select `ROG Control Center` from Applications list, add custom shortcut, press the ROG key and click Apply.

### Change Fan Profile
Go to KDE Settings->Shortcuts. Click Add Command, in the dialogue box, enter `asusctl profile -n`. Set trigger to `fn + f5` and click Apply.

## Fixing Audio on Linux


Audio was exceptionally low on linux. To fix, first remove everything pulseaudio related by running:
```
sudo pacman -Rdd pulseaudio pulseaudio-alsa pulseaudio-bluetooth pulseaudio-ctl pulseaudio-equalizer pulseaudio-jack pulseaudio-lirc pulseaudio-rtp pulseaudio-zeroconf pulseaudio-equalizer-ladspa
```
Most of this may not be installed already so remove it from the command.
Then, install pipewire and its related packages.
```
sudo pacman -S pipewire pipewire-pulse gst-plugin-pipewire pipewire-alsa pipewire-media-session plasma pa
```

Install bluetooth related packages
```
sudo pacman -S bluez bluez-utils
sudo systemctl enable bluetooth.service
sudo systemctl start bluetooth.service
```
Install easyeffects
```
sudo pacman -S easyeffects
```
Run easyeffects and close. This will create the necessary directories.
Install easyeffects-presets.
```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/JackHack96/PulseEffects-Presets/master/install.sh)"
```
Launch easyeffects again, click presets on top left and try the installed presets to find the one that best suits your needs. You can also replace the default laptop config with the one from my repository if you like. Copy `~/.config/easyeffects/output/Laptop.json` from the repository to the same path in your laptop. In the top panel in easy effects, click on Pipewire, select presets autoloading and then choose the preset that you want to load automatically on startup. Click the plus button on the right to add it to the list. In the hamburger menu, click preferences and enable `Launch Service at startup` and close easyeffects. If needed, Easy effects can be run as daemon using
```
easyeffects --gapplication-service & 
```
It automatically adds itself to autostart, and runs as a service on reboot. No other config needed. [Source](https://askubuntu.com/questions/984109/dolby-equivalent-for-ubuntu)

Note: Do not set the audio device in system settings to easyeffets source or sink as it will cause problems. Use the default hardware device.

### Mic Mute Key
Mic mute key should work out of the box in latest versions of plasma, provided `plasma-pa` package is installed. If it doesnt work, do the following steps.
Run `usb-devices` and look for the device that says `Product=N-KEY Device`. Note the vendor id. For my zephyrus it is `0b05`.  Run 
```
sudo find /sys -name modalias | xargs grep -i 0b05
```

Find the line that goes like:
```
.../input/input18/modalias:input:b0003v0B05p1866e0110-e0...
```
Copy the part after `input:`, before the first `-e`. In my case, it is `b0003v0B05p1866e0110`.  Create a file named `/etc/udev/hwdb.d/90-nkey.hwdb` and add
```
/etc/udev/hwdb.d/90-nkey.hwdb
evdev:input:b0003v0B05p1866*
 KEYBOARD_KEY_ff31007c=f20 # x11 mic-mute, space in start is important in this line
```
After that, update `hwdb`.
```
sudo systemd-hwdb update
sudo udevadm trigger
```



## Setup Automatic Snapshots for pacman

At this point, we have installed everything we need. Reboot the system once to make sure everything works fine and set up BTRFS snapshots to ensure we always have a restore point in case something breaks in the future. To do so, first create a snapshot manually as follows
```
sudo -i
btrfs sub snap / /.snapshots/STABLE
cp /boot/vmlinuz-linux-g14 /boot/vmlinuz-linux-g14-stable
cp /boot/amd-ucode.img /boot/amd-ucode-stable.img
cp /boot/initramfs-linux-g14.img /boot/initramfs-linux-g14-stable.img
cp /boot/loader/entries/arch-g14.conf /boot/loader/entries/arch-g14-stable.conf
```
Edit `/boot/loader/entries/arch-g14-stable.conf` to boot from `STABLE` snapshot
```
title Arch Linux (G14) Stable
linux /vmlinuz-linux-g14-stable
initrd /amd-ucode-stable.img
initrd /initramfs-linux-g14-stable.img
options cryptdevice=UUID=62b5e6e0-6376-46d8-9faf-fe391a58c6b1:luks root=/dev/mapper/luks rootflags=subvol=@snapshots/STABLE quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```
Now edit `/.snapshots/STABLE/etc/fstab` to change the root of the STABLE snapshot.
```
 ... LABEL=ROOTFS / btrfs rw,noatime,.....subvol=@snapshots/STABLE ...
```
Reboot the system, in the boot menu, select `Arch Linux (G14) Stable` to see if it boots correctly. If it does, boot back into `Arch Linux (G14)`. 

Copy the script from repo to `/usr/bin/autosnap` and make it executable with `chmod +x /usr/bin/autosnap`. Then copy the pacman hook script from the repo to `/etc/pacman.d/hooks/00-autosnap.hook`. 

Now every time pacman installs or upgrades something, the oldest snapshot would be removed a new one will be created. Let's test everything one more time to ensure nothing breaks. To do so, install any package from pacman, e.g.
```
sudo pacman -S android-tools
```
The output should look something like this
```
resolving dependencies...
looking for conflicting packages...

Packages (1) android-tools-34.0.1-1

Total Installed Size:  5.66 MiB

:: Proceed with installation? [Y/n] Y
(1/1) checking keys in keyring                                     [####################################] 100%
(1/1) checking package integrity                                   [####################################] 100%
(1/1) loading package files                                        [####################################] 100%
(1/1) checking for file conflicts                                  [####################################] 100%
(1/1) checking available disk space                                [####################################] 100%
:: Running pre-transaction hooks...
(1/1) Creating btrfs snapshot
Delete subvolume (no-commit): '/.snapshots/STABLE'
Create a snapshot of '/' in '/.snapshots/STABLE'
:: Processing package changes...
(1/1) installing android-tools                                     [####################################] 100%
:: Running post-transaction hooks...
(1/1) Arming ConditionNeedsUpdate...
```

Note that in pre-transaction hooks, it deletes the STABLE snapshot, takes the snapshot of the current system in `/.snapshots/STABLE` before proceeding to install the package. Boot back into the stable snapshot and run `adb` in the terminal. It should say `command not found`. Now boot back into the normal system and try running `adb` again, it would work without issues.

The script maintains five recent snapshots, allowing you to boot back into an older one if something breaks.

## Install an AUR Helper
Install [pamac](https://aur.archlinux.org/packages/pamac-aur/) to easily manage AUR packages with a GUI. Enable AUR in pamac settings.

## Installing Waydroid
Waydroid helps run android apps on Linux. With `linux-g14` kernel installed, install the `binder_linux-dkms` package, which is available through AUR. Run these three commands, if at least one of them executes without any output or error, it means that the module is installed correctly
```
sudo modprobe -a binder-linux
sudo modprobe -a binder_linux
sudo modprobe -a binder
```
Now initialize waydroid and reboot (this will probably take a while because downloads from sourceforge are extremely slow)
```
sudo waydroid init
# OR WITH GAPPS
sudo waydroid init -s GAPPS
```
Sourceforge tends to selected the slowest possible mirror (I was getting ~45kbps). So alternatively, go to
```
https://sourceforge.net/projects/waydroid/files/images/vendor/waydroid_x86_64/
and
https://sourceforge.net/projects/waydroid/files/images/system/lineage/waydroid_x86_64/
```
and download the latest ones. As soon as it starts download, before the redirect, click `Problems Downloading?` button and select a different mirror (I used one from US and it worked fine). Extract the downloaded files to get `system.img` and `vendor.img`. In official docs, these are supposed to be placed in `/usr/share/waydroid-extra/images/` but the automatically downloaded ones were located in `/var/lib/waydroid/images/`, so just copy both extracted files to both of these locations (just in case), run the following command and reboot. I used the GApps image, but the regular one can also be used.
```
sudo waydroid init -f
```

After rebooting, verify if binderfs is correctly loaded by running
```
sudo ls -1 /dev/binderfs
```
It should return the following output (or something similar)
```
anbox-binder
anbox-hwbinder
anbox-vndbinder
binder-control
features
```
Enable waydroid by running
```
sudo systemctl enable --now waydroid-container
```
Then launch waydroid from application menu. Networking should work out of the box. To install an application, run
```
waydroid app install /path/to/apk
```
[Credits](https://forum.garudalinux.org/t/ultimate-guide-to-install-waydroid-in-any-arch-based-distro-especially-garuda/15902)

To enable windowed mode, run
```
waydroid prop set persist.waydroid.multi_windows true
```
To disable on screen keyboard
```
$ waydroid show-full-ui
Settings > System > Languages & input > Physical keyboard > Use on-screen keyboard
```
To use GApps, start waydroid, then run
```
sudo waydroid shell
ANDROID_RUNTIME_ROOT=/apex/com.android.runtime ANDROID_DATA=/data ANDROID_TZDATA_ROOT=/apex/com.android.tzdata ANDROID_I18N_ROOT=/apex/com.android.i18n sqlite3 /data/data/com.google.android.gsf/databases/gservices.db "select * from main where name = \"android_id\";"
```
Go to [Google Device Registration](https://www.google.com/android/uncertified) and paste the numbers shown after "android_id|" to register. Wait for a few minutes for Google services to reflect the change and then restart waydroid.
[Source](https://docs.waydro.id/faq/google-play-certification)


## Gamma Correction and Color profile
On X11, in settings, under display and monitor -> gamma, change gamma to 0.9 for better colors.
Gamma correction is not available on KDE wayland yet. Install and run fastfetch to get the built-in display code. In my case it is `CMN14D5`. Google search for your code and append notebookcheck, click the first link. It would be for a different laptop that uses the same display. Press `Ctrl+F` and enter the code to ensure that the laptop uses this display. Download the ICC file and copy it to `/usr/share/color/icc/colord`. Then run 
```
colormgr get-profiles
```
Find the profile that contains the filename you just copied. Copy the Profile ID and run
```
sudo colormgr device-add-profile eDP-1 <Profile ID goes here>
```
After that, in KDE settings, under color management, select this profile. To make this profile the default, run
```
sudo colormgr device-make-profile-default eDP-1 <Profile ID goes here>
```

## Miscellaneous
	
### Fetch on Terminal Start
Install fastfetch 
```
sudo pacman -S fastfetch
```
Copy the fastfetch config from the repo tothe relevant directory and add `fastfetch` to the top of your `.zshrc` file.


### QuickShare
[This awesome project](https://github.com/Martichou/rquickshare) provides a rust implementation of quickshare for linux. Works great in my testing. Download the latest AppImage, make it executable and move it to `~/.local/bin`. On first start, it will add itself to autostart. Includes a tray icon as well.
