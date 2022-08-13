# Postinstall

> **Note**
>
> A number of these steps require root

## Network

### DNS

Enable the service:

```zsh
systemctl enable --now systemd-resolved.service
```

Configure:

```zsh
mkdir /etc/systemd/resolved.conf.d
echo "[Resolve]" > /etc/systemd.resolved.conf.d/cloudflare.conf
echo "DNS=1.1.1.1 1.0.0.1 2606:4700:4700::1111 2606:4700:4700::1001" >> /etc/systemd.resolved.conf.d/cloudflare.conf
echo "FallbackDNS=84.200.69.80 84.200.70.40 2001:1608:10:25::1c04:b12f 2001:1608:10:25::9249:d69b" >> /etc/systemd.resolved.conf.d/cloudflare.conf
echo "DNSSEC=true" >> /etc/systemd.resolved.conf.d/cloudflare.conf
echo "DNSOverTLS=yes" >> /etc/systemd.resolved.conf.d/cloudflare.conf
echo "Domains=~." >> /etc/systemd.resolved.conf.d/cloudflare.conf
```

### Systemd-networkd

Enable the service:

```zsh
systemctl enable --now systemd-networkd.service
```

Configure:

```zsh
echo "[Match]" > /etc/systemd/network/20-wired.network
echo "Name=enp8s0" >> /etc/systemd/network/20-wired.network
echo "" >> /etc/systemd/network/20-wired.network
echo "[Network]" >> /etc/systemd/network/20-wired.network
echo "DHCP=true" >> /etc/systemd/network/20-wired.network
```

### Firewall

```zsh
ufw enable
```

### ZRAM

Install the generator:

```zsh
pacman -S zram-generator
```

Config:

```zsh
echo "[zram0]" >> /etc/systemd/zram-generator.conf
echo "zram-size = min(ram / 2, 4096)" >> /etc/systemd/zram-generator.conf
echo "compression-algorithm = zstd" >> /etc/systemd/zram-generator.conf
systemctl daemon-reload
systemctl enable --now systemd-zram-setup@zram0.service
```

## Security

### AppArmor

```zsh
pacman -S apparmor
systemctl enable --now apparmor.service
sed -i 's/#write-cache/write-cache/' /etc/apparmor/parser.conf
sed -i '/^options.*/ s/$/ lsm=landlock,lockdown,yama,apparmor,bpf audit=1/' /boot/loader/entries/arch.conf
sed -i '/^options.*/ s/$/ lsm=landlock,lockdown,yama,apparmor,bpf audit=1/' /boot/loader/entries/arch-fallback.conf
systemctl enable --now auditd.service
```

### Configurations

## Backups (snapper)

#### **Setup**

```zsh
pacman -S snapper
umount /.snapshots
rm -r /.snapshots
snapper -c root create-config /
btrfs subvolume delete /.snapshots
mkdir /.snapshots
mount -a
chmod 750 /.snapshots
sed -i 's/TIMELINE_LIMIT_HOURLY="10"/TIMELINE_LIMIT_HOURLY="2"/' /etc/snapper/configs/root
sed -i 's/TIMELINE_LIMIT_DAILY="10"/TIMELINE_LIMIT_DAILY="7"/' /etc/snapper/configs/root
sed -i 's/TIMELINE_LIMIT_MONTHLY="10"/TIMELINE_LIMIT_MONTHLY="0"/' /etc/snapper/configs/root
sed -i 's/TIMELINE_LIMIT_MONTHLY="10"/TIMELINE_LIMIT_YEARLY="0"/' /etc/snapper/configs/root
systemctl enable --now snapper-timeline.timer
systemctl enable --now snapper-cleanup.timer
pacman -S snap-pac
```

## Nvidia

Install:

```zsh
pacman -S nvidia
```

Enable DRM:

```zsh
sed -i '/^options.*/ s/$/ nvidia-drm.modeset=1/' /boot/loader/entries/arch.conf
sed -i '/^options.*/ s/$/ nvidia-drm.modeset=1/' /boot/loader/entries/arch-fallback.conf
sed -i 's/MODULES=()/MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)/' /etc/mkinitcpio.conf
```

Add pacman hook to update initramfs on nvidia upgrade:

```zsh
echo "[Trigger]" >> /etc/pacman.d/hooks/nvidia.hook
echo "Operation=Install" >> /etc/pacman.d/hooks/nvidia.hook
echo "Operation=Upgrade" >> /etc/pacman.d/hooks/nvidia.hook
echo "Operation=Remove" >> /etc/pacman.d/hooks/nvidia.hook
echo "Type=Package" >> /etc/pacman.d/hooks/nvidia.hook
echo "Target=nvidia" >> /etc/pacman.d/hooks/nvidia.hook
echo "Target=linux" >> /etc/pacman.d/hooks/nvidia.hook
echo "" >> /etc/pacman.d/hooks/nvidia.hook
echo "[Action]" >> /etc/pacman.d/hooks/nvidia.hook
echo "Description=Update Nvidia module in initcpio" >> /etc/pacman.d/hooks/nvidia.hook
echo "Depends=mkinitcpio" >> /etc/pacman.d/hooks/nvidia.hook
echo "When=PostTransaction" >> /etc/pacman.d/hooks/nvidia.hook
echo "NeedsTargets" >> /etc/pacman.d/hooks/nvidia.hook
echo "Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'" >> /etc/pacman.d/hooks/nvidia.hook
```

Set max console resolution:

```zsh
echo "console-mode max" >> /boot/loader/loader.conf
```

## Desktop

### Qtile window manager (Xorg)

> Hopefully on wayland in a year or two (fingers-cross)...fucking nvidia

Install qtile, xinit for starting, some python libs for widgets, some fonts and kitty for terminal emulation

```zsh
pacman -S xorg-xinit qtile python-dbus-next python-psutil python-xdg python-setproctitle lm_sensors kitty noto-fonts noto-fonts-emoji ttf-dejavu ttf-droid ttf-roboto terminus-font adobe-source-code-pro
```

### Pipeware (audio)

Install:

```zsh
pacman -S pipewire wireplumber pipewire-jack pipewire-alsa pipewire-pulse
```

### Firefox (web browser)

```zsh
pacman -S firefox-developer-edition
```
