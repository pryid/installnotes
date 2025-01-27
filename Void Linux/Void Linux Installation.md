# Void Linux installation (btrfs, LVM, full disk encryption using LUKS, SSD TRIM)

## Basics
- Laptop: Lenovo ThinkPad x240

## Features
This guide explains how to set up Void Linux:
- Using full disk encryption - **including** /boot, with LUKS + LVM
- Uses btrfs as filesystem
- Uses niri, a scrollable-tiling Wayland compositor.
- Uses my predefined settings, such as polkit rules and dotfiles
## The process

### Pre-chroot
```bash
# boot Void live system and log in using root:voidlinux
# if running XFCE Live, then execute sudo -i bash

fdisk /dev/sda

# g to create a new GPT partition table
# n new partition with +300M
# t 1 to set partition type to EFI System Partition
# n new partition with remaining space
# w to write partition map and exit

cryptsetup luksFormat --type=luks1 /dev/sda2
cryptsetup open /dev/sda2 crypt

# prepare LVM

vgcreate vg0 /dev/mapper/crypt
lvcreate --name void -l +100%FREE vg0

# filesystems

mkfs.vfat -n BOOT -F 32 /dev/sda1
mkfs.btrfs -L void /dev/mapper/vg0-void

mount -o rw,noatime,ssd,compress=zstd,space_cache=v2 /dev/mapper/vg0-void /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
umount /mnt
mount -o rw,noatime,ssd,compress=zstd,space_cache=v2,subvol=@ /dev/mapper/vg0-void /mnt
mkdir /mnt/home
mkdir /mnt/.snapshots
mount -o rw,noatime,ssd,compress=zstd,space_cache=v2,subvol=@home /dev/mapper/vg0-void /mnt/home/
mount -o rw,noatime,ssd,compress=zstd,space_cache=v2,subvol=@snapshots /dev/mapper/vg0-void /mnt/.snapshots/
mkdir -p /mnt/boot/efi
mount -o rw,noatime /dev/sda1 /mnt/boot/efi/
mkdir -p /mnt/var/cache
export XBPS_ARCH=x86_64
xbps-install -Sy -R https://repo-default.voidlinux.org/current/ -r /mnt base-system btrfs-progs acpi alsa-pipewire alsa-utils brightnessctl btop cryptsetup curl dbus elinks fastfetch fish-shell flatpak fuzzel gnome-keyring grub-x86_64-efi gvfs htop intel-video-accel kitty lvm2 mako mesa-dri nano nautilus niri openbsd-netcat opendoas pavucontrol pipewire podman polkit-gnome rsync screen seatd swaylock tlp tree tuigreet turnstile udisks2 vulkan-loader wget xdg-desktop-portal-gnome xdg-user-dirs xdg-utils terminus-font git NetworkManager unzip chrony micro
mount -t proc proc /mnt/proc/
mount -t sysfs sys /mnt/sys/
mount -o bind /dev /mnt/dev
mount -t devpts pts /mnt/dev/pts
cp -L /etc/resolv.conf /mnt/etc/
xchroot /mnt /bin/bash
```

### Post-chroot
```bash
passwd root -l #Disables the root account. If you want to keep it, simply execute passwd root instead.
chown root:root /
chmod 755 /

echo voidpc > /etc/hostname

cat <<EOF > /etc/rc.conf
# /etc/rc.conf - system configuration for void
HARDWARECLOCK="UTC"
TIMEZONE="Asia/Yekaterinburg"
CGROUP_MODE=unified
EOF

echo 'en_US.UTF-8 UTF-8' > /etc/default/libc-locales
echo LANG=en_US.UTF-8 > /etc/locale.conf
export UEFI_UUID=$(blkid -s UUID -o value /dev/sda1)
export LUKS_UUID=$(blkid -s UUID -o value /dev/sda2)
export ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/vg0-void)

cat <<EOF > /etc/fstab
UUID=$ROOT_UUID / btrfs rw,noatime,ssd,compress=zstd,space_cache=v2,subvol=@ 0 1
UUID=$ROOT_UUID /home btrfs rw,noatime,ssd,compress=zstd,space_cache=v2,subvol=@home 0 2
UUID=$ROOT_UUID /.snapshots btrfs rw,noatime,ssd,compress=zstd,space_cache=v2,subvol=@snapshots 0 2
UUID=$UEFI_UUID /boot/efi vfat defaults,noatime 0 2
tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
EOF

cat <<EOF >> /etc/default/grub
GRUB_ENABLE_CRYPTODISK=y
EOF

sed -i "/GRUB_CMDLINE_LINUX_DEFAULT=/s/\"$/ rd.auto=1 cryptdevice=UUID=$LUKS_UUID:lvm:allow-discards&/" /etc/default/grub

dd bs=512 count=4 if=/dev/urandom of=/boot/volume.key
cryptsetup luksAddKey /dev/sda2 /boot/volume.key
chmod 000 /boot/volume.key
chmod -R g-rwx,o-rwx /boot

cat <<EOF >> /etc/crypttab
crypt /dev/sda2 /boot/volume.key luks
EOF
cat <<EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

echo 'add_dracutmodules+=" crypt btrfs lvm resume "' >> /etc/dracut.conf
echo 'tmpdir=/tmp' >> /etc/dracut.conf
dracut --force --hostonly --kver "$(ls /usr/lib/modules | sort -V | tail -n 1)"
mount -t efivarfs none /sys/firmware/efi/efivars

# Configure grub

mkdir /boot/grub
grub-mkconfig -o /boot/grub/grub.cfg
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void --boot-directory=/boot  --recheck
sed -i 's/issue_discards = 0/issue_discards = 1/' /etc/lvm/lvm.conf


# Activate services

ln -s /etc/sv/chronyd /etc/runit/runsvdir/default
ln -s /etc/sv/NetworkManager /etc/runit/runsvdir/default
ln -s /etc/sv/dbus /etc/runit/runsvdir/default
ln -s /etc/sv/tlp /etc/runit/runsvdir/default
ln -s /etc/sv/turnstiled /etc/runit/runsvdir/default
ln -s /etc/sv/polkitd /etc/runit/runsvdir/default
ln -s /etc/sv/seatd /etc/runit/runsvdir/default
ln -s /etc/sv/acpid /etc/runit/runsvdir/default
ln -s /etc/sv/greetd /etc/runit/runsvdir/default

# Configure PipeWire

mkdir -p /etc/pipewire/pipewire.conf.d
ln -s /usr/share/examples/wireplumber/10-wireplumber.conf /etc/pipewire/pipewire.conf.d/
ln -s /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/

mkdir -p /etc/alsa/conf.d
ln -s /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
ln -s /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d

# Adjust tuigreet configuration

sed -i "s/^command.*/command = \"tuigreet -r --power-shutdown 'doas shutdown -h now' --power-reboot 'doas shutdown -r now' --cmd 'dbus-run-session niri --session'\"/" /etc/greetd/config.toml

# Set up polkit rules for udisks

cat <<EOF > /etc/polkit-1/rules.d/50-udisks.rules
polkit.addRule(function(action, subject) {
   if (subject.isInGroup("wheel")) {
        if (action.id.startsWith("org.freedesktop.udisks2.")) {
            return polkit.Result.YES;
        }
    }
});
EOF

# Create a new user and set password

useradd -m -G wheel,audio,video,storage,plugdev,network,_seatd voiduser
passwd voiduser

# Configure doas

cat <<EOF > /etc/doas.conf
permit persist :wheel
permit nopass :video as root cmd shutdown
EOF

# Install fonts for the user, Inter, JetBrains Mono, Apple Color Emoji

su - voiduser -c "
mkdir -p ~/.fonts/inter ~/.fonts/jbmono ~/.fonts/emoji && \

wget -O inter.zip https://github.com/rsms/inter/releases/download/v4.1/Inter-4.1.zip && \
unzip -q inter.zip -d ~/.fonts/inter && \
mv ~/.fonts/inter/extras/ttf/* ~/.fonts/inter/ && \
rm -rf ~/.fonts/inter/extras ~/.fonts/inter/web ~/.fonts/inter/*.txt ~/.fonts/inter/*.css && \
rm ~/.fonts/inter/InterVariable-Italic.ttf ~/.fonts/inter/InterVariable.ttf && \
rm inter.zip && \

wget -O jetbrains.zip https://github.com/ryanoasis/nerd-fonts/releases/download/v3.3.0/JetBrainsMono.zip && \
unzip -q jetbrains.zip -d ~/.fonts/jbmono && \
find ~/.fonts/jbmono -type f ! -name '*.ttf' -delete && \
rm jetbrains.zip && \

wget -O ~/.fonts/emoji/AppleColorEmoji.ttf https://github.com/samuelngs/apple-emoji-linux/releases/download/v17.4/AppleColorEmoji.ttf
"

# Download and apply dotfiles

su - voiduser -c "git clone https://github.com/pryid/dotfiles ~/dotfiles && cp -r ~/dotfiles/.config ~/.config && rm -rf ~/dotfiles && mkdir ~/.ssh"

# Update XDG dirs

su - voiduser -c "xdg-user-dirs-update"

# Reconfigure everything
xbps-reconfigure -fa

# Unmount all and reboot

exit
umount -R /mnt && reboot

# Optional: LibreWolf browser. It is not available in Void Linux repository, but can be installed from third-party repository

echo 'repository=https://github.com/index-0/librewolf-void/releases/latest/download/' > /etc/xbps.d/20-librewolf.conf
xbps-install -Su librewolf

# Optional: enable a dark theme across all GTK applications and disable the "close" button

gsettings set org.gnome.desktop.interface color-scheme prefer-dark
gsettings set org.gnome.desktop.wm.preferences button-layout ':'

# Optional: enable zram

xbps-install -S zramen

cat <<EOF >> /etc/sv/zramen/conf
export ZRAM_COMP_ALGORITHM=lz4
export ZRAM_PRIORITY=32767
export ZRAM_SIZE=1024
export ZRAM_MAX_SIZE=2048
export ZRAM_STREAMS=1
EOF

ln -s /etc/sv/zramen /var/service
```
