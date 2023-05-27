# Arch install guide

## Keys and clock

Set keyboard layout:

```zsh
loadkeys uk
```

Update system clock:

```zsh
timedatectl set-ntp true
```

## Partition

We're using btrfs + zram so the only partitions we need are `root` and `boot`. Since we're most likely dual booting and using UEFI `boot` should already be there so we'll likely just need one to create one partition for `root`.

Our partition scheme should look something like:

Open gdisk on the device:

```zsh
gdisk /dev/nvme0n1
```

create the `root` partition and `boot` if needed.

### Crypt

We're just encrypting root. Boots stays unencrypted due to dual boot.

Encrypt:

```zsh
cryptsetup luksFormat --perf-no_read_workqueue --perf-no_write_workqueue --type luks2 --cipher aes-xts-plain64 --key-size 512 --iter-time 2000 --pbkdf argon2id --hash sha3-512 /dev/nvme0n1p4
```

Open crypt:

```zsh
cryptsetup --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent open /dev/nvme0n1p4 root
```

Format the mapped partition:

```zsh
mkfs.btrfs /dev/mapper/root
```

### Btrfs subvolumes

Create the subvolumes:

```zsh
mount /dev/mapper/root /mnt
btrfs sub create /mnt/@
btrfs sub create /mnt/@home
btrfs sub create /mnt/@abs
btrfs sub create /mnt/@tmp
btrfs sub create /mnt/@srv
btrfs sub create /mnt/@snapshots
btrfs sub create /mnt/@log
btrfs sub create /mnt/@cache
umount /mnt
```

Mount the shizz:

```zsh
mount -o noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@ /dev/mapper/root /mnt
mkdir -p /mnt/{boot,home,var/cache,var/log,.snapshots,var/tmp,var/abs,srv}
mount -o noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@home /dev/mapper/root /mnt/home
mount -o noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@srv /dev/mapper/root /mnt/srv &&
mount -o noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@snapshots /dev/mapper/root /mnt/.snapshots
mount -o nodev,nosuid,noexec,noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@abs /dev/mapper/root /mnt/var/abs
mount -o nodev,nosuid,noexec,noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@tmp /dev/mapper/root /mnt/var/tmp
mount -o nodev,nosuid,noexec,noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@log /dev/mapper/root /mnt/var/log
mount -o nodev,nosuid,noexec,noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag,subvol=@cache /dev/mapper/root /mnt/var/cache
```

Disable COW for vms/dbs:

```zsh
mkdir -p /mnt/var/lib/{docker,machines,mysql,postgres}
chattr +C /mnt/var/lib/{docker,machines,mysql,postgres}
```

Mount boot:

```zsh
mount -o nodev,nosuid,noexec /dev/nvme0n1p1 /mnt/boot
```

## Install the system

> **Note**
>
> Before installing ensure that `/mnt/boot` is cleared of any old system images, if you are replacing an old system that is.
> You will need to remove `grub/` `vmlinuz*` `initramfs*` and `amd-ucode.img` for instance

Minimal pacstrap:

```zsh
pacstrap /mnt base linux linux-firmware amd-ucode git vim btrfs-progs zsh base-devel
```

## Confirgure the system

### fstab

```zsh
genfstab -U /mnt > /mnt/etc/fstab
```

### chroot

```zsh
arch-chroot /mnt
```

### Timezone

Set the time zone:

```zsh
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```

Run `hwclock` to generate /etc/adjtime:

```zsh
hwclock --systohc
```

### Localization

Gen:

```zsh
echo "en_GB.UTF-8 UTF-8" >> /etc/locale.gen
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
```

Lang & keymap:

```zsh
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
echo "KEYMAP=uk" > /etc/vconsole.conf
export LANG="en_GB.UTF-8"
export LC_COLLATE="C"
```

hostname

```zsh
echo "desktop" > /etc/hostname
```

hosts

```zsh
echo -e "127.0.0.1\tlocalhost" >> /etc/hosts
echo -e "::1\t\tlocalhost" >> /etc/hosts
echo -e "127.0.1.1\tdesktop.localdomain\tdesktop" >> /etc/hosts
```

mkinitcpio.conf

```zsh
sed -i 's/BINARIES=()/BINARIES=("\/usr\/bin\/btrfs")/' /etc/mkinitcpio.conf
sed -i 's/^HOOKS.*/HOOKS=(base systemd autodetect modconf block sd-encrypt filesystems keyboard fsck)/' /etc/mkinitcpio.conf
mkinitcpio -p linux
```

### Users

Root:

```zsh
passwd
chsh -s /bin/zsh
```

Admin:

```zsh
useradd -m -s /bin/zsh brendan
passwd brendan
echo "brendan ALL=(ALL) ALL" >> /etc/sudoers
echo "Defaults timestamp_timeout=0" >> /etc/sudoers
```

### Firewall

```zsh
pacman -S ufw
systemctl enable ufw.service
ufw default deny
```

### Boot manager (systemd-boot)

Install some packages:

```zsh
pacman -S efibootmgr dosfstools mtools
```

Install systemd-boot:

```zsh
bootctl --path=/boot install
echo "timeout 10" >> /boot/loader/loader.conf
echo "default arch" >> /boot/loader/loader.conf
```

Add arch entry:

```zsh
echo "title Arch Linux" > /boot/loader/entries/arch.conf
echo "linux /vmlinuz-linux" >> /boot/loader/entries/arch.conf
echo "initrd /amd-ucode.img" >> /boot/loader/entries/arch.conf
echo "initrd /initramfs-linux.img" >> /boot/loader/entries/arch.conf
echo "options rd.luks.name=partuuid=root root=/dev/mapper/root rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf
sed -i "s/partuuid/$(blkid /dev/nvme0n1p4 -s UUID -o value)/" /boot/loader/entries/arch.conf
```

Add fallback entry

```zsh
cp /boot/loader/entries/arch.conf /boot/loader/entries/arch-fallback.conf
sed -i "s/initramfs-linux/initramfs-linux-fallback/" /boot/loader/entries/arch-fallback.conf
```

### DOTDIRS (Keep home clean!!)

```zsh
echo -e "EDITOR\tDEFAULT=vim" >> /etc/security/pam_env.conf
echo -e "VISUAL\tDEFAULT=vim" >> /etc/security/pam_env.conf
echo -e "XDG_CONFIG_HOME\tDEFAULT=@{HOME}/.config" >> /etc/security/pam_env.conf
echo -e "XDG_CACHE_HOME\tDEFAULT=@{HOME}/.cache" >> /etc/security/pam_env.conf
echo -e "XDG_DATA_HOME\tDEFAULT=@{HOME}/.local/share" >> /etc/security/pam_env.conf
echo -e "XDG_STATE_HOME\tDEFAULT=@{HOME}/.local/state" >> /etc/security/pam_env.conf
echo -e "XDG_DATA_DIRS\tDEFAULT=/usr/local/share:/usr/share" >> /etc/security/pam_env.conf
echo -e "XDG_CONFIG_DIRS\tDEFAULT=/etc/xdg" >> /etc/security/pam_env.conf
echo -e "ZDOTDIR=$XDG_CONFIG_HOME/zsh" >> /etc/zsh/zshenv
echo -e "HISTFILE=$XDG_STATE_HOME/zsh/history" >> /etc/zsh/zshenv
```

**DONE!**

> Next steps can be completed after booting in and can be found within postinstall.md
