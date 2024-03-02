# Arch Linux Setup
+ luks veschlüsselt
+ btrfs Dateisystem
+ swap Partition
+ Hibernation
+ zsh

## 1. Installationsmedium
Installationsmedium erstellen (dd, ventoy, balenaEtcher, ...)

## 2. Partitionierung der Festplatte

EFI-Systempartition: 512 MiB
Swap-Partition: >= RAM für Hibernation
Root: Rest der Festplatte
```
cfdisk # [Type] Flags EFI, Linux swap, Linux Filesystem
```

## 3. Dateisysteme anlegen, Swap erstellen, Verschlüsselung einrichten
```
mkfs.vfat /dev/sdaE # boot/efi
mkswap /dev/sdaY # Swap-Partition
swapon /dev/sdaY
cryptsetup luksFormat /dev/sdaX # Root-Partition, 'YES' in Großbuchstaben
cryptsetup open /dev/sdaX cryptroot
```

## 4. BTRFS einrichten und Subvolumes erstellen
```
mkfs.btrfs /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
umount /mnt
```

## 5. Mounten der Subvolumes und Installation
```
mount -o subvol=@,compress=zstd,noatime /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{boot/efi,home,.snapshots}
mount /dev/sdaE /mnt/boot/efi
mount -o subvol=@home,compress=zstd,noatime /dev/mapper/cryptroot /mnt/home
mount -o subvol=@snapshots,compress=zstd,noatime /dev/mapper/cryptroot /mnt/.snapshots
```

## 6. Installation des Grundsystems
```
pacstrap /mnt base linux linux-firmware btrfs-progs
```

## 7. Konfiguration des Systems
Generiere fstab, chroot, setze Zeitzone, Lokalisierung, Netzwerk und Passwort.
```
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

## 8. Bootloader konfigurieren (GRUB)
```
pacman -S grub efibootmgr vi
vi /etc/default/grub # uncomment GRUB_ENABLE_CRYPTODISK=y
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
# Füge GRUB_CMDLINE_LINUX="cryptdevice=UUID=deine-root-partition-uuid:cryptroot root=/dev/mapper/cryptroot resume=/dev/sdaY" zu /etc/default/grub hinzu
grub-mkconfig -o /boot/grub/grub.cfg
```

## 9. Basisprogramme installieren und Standardshell ändern
```
pacman -S zsh sway swaylock swayidle waybar wofi alacritty greetd
chsh -s /usr/bin/zsh
```

## 10. Hibernation einrichten
Swap-Partition in der GRUB_CMDLINE_LINUX erwähnen und mkinitcpio.conf anpassen, um resume zu HOOKS hinzuzufügen, dann generiere mkinitcpio neu.
```
mkinitcpio -p linux
```
