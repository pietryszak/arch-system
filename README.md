# Arch Linux — konfiguracja po instalacji

Ten plik jest przeznaczony jako osobne README dla repo z konfiguracją **po instalacji Archa**.

Założenie: bazowy system Arch już działa, masz Btrfs + Snapper + grub-btrfs, więc przed większymi zmianami robisz snapshot.

---
## 1.Katalog na klony/AUR/własne rzeczy:

```bash
mkdir -p ~/.gc
```

## 2. Wymuszenie polskiego układu klawiatury dla GUI

Systemowo masz `KEYMAP=pl`, ale dla GUI/Plasma możesz dodatkowo ustawić:

```bash
sudo localectl --no-convert set-x11-keymap pl
```

Sprawdzenie:

```bash
localectl status
```
Dodatkowo NumLock Settins > Keyboard > Numlock on startup > Turn on

## 3. Fstrim

`fstrim.timer` warto mieć na SSD/NVMe:

```bash
sudo systemctl enable --now fstrim.timer
```

## 4. Pakiety bazowe po instalacji

Przed większym blokiem zmian:

```bash
sudo snapper -c root create --description "before post-install tweaks"
sudo snapper -c home create --description "home before post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

```bash
sudo pacman -S --needed \
  bash-completion btop fastfetch openssh playerctl \
  zip unzip p7zip \
  exfatprogs dosfstools \
  usbutils lsof smartmontools traceroute \
  wireguard-tools firefox thunderbird base-devel linux-headers dkms
```

---

## 5. Kodeki multimedialne

Minimalny system ma część bibliotek multimedialnych jako zależności KDE, ale do pełniejszej obsługi audio/wideo warto doinstalować:

```bash
sudo pacman -S --needed \
  ffmpeg \
  gst-plugins-base gst-plugins-good gst-plugins-bad gst-plugins-ugly gst-libav \
  gst-plugin-pipewire
```

Opcjonalnie miniatury filmów i dodatkowe formaty obrazów w KDE/Dolphin:

```bash
sudo pacman -S --needed \
  ffmpegthumbs \
  kimageformats \
  qt6-imageformats
```

`libdvdcss` instaluj tylko, jeśli faktycznie potrzebujesz odtwarzania szyfrowanych DVD:

```bash
sudo pacman -S --needed libdvdcss
```
---

## 6. `yay`

Instalacja `yay`:

```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

## 7. Pakiety AUR, jeśli ich chcesz:

```bash
yay -S brave-origin-nightly
```


---

## 5. Plymouth i splash screen

Instalacja Plymouth:

```bash
sudo pacman -S --needed plymouth plymouth-kcm
```

Motyw z AUR:

```bash
yay -S plymouth-theme-arch-breeze-git
```

Ustawienie motywu:

```bash
sudo plymouth-set-default-theme -R arch-breeze
```

Dodanie `splash` do GRUB:

```bash
sudo grep -q 'splash' /etc/default/grub || \
  sudo sed -i 's/\(GRUB_CMDLINE_LINUX_DEFAULT="[^"]*\)"/\1 splash"/' /etc/default/grub
```

Dodanie hooka `plymouth` do `mkinitcpio` przy układzie `systemd + sd-encrypt`:

```bash
sudo grep -q ' plymouth ' /etc/mkinitcpio.conf || \
  sudo sed -i 's/\<sd-vconsole\>/sd-vconsole plymouth/' /etc/mkinitcpio.conf

grep '^HOOKS=' /etc/mkinitcpio.conf
```

Oczekiwany układ z Plymouth:

```text
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole plymouth block sd-encrypt filesystems fsck)
```

Przebudowa:

```bash
sudo mkinitcpio -P
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 6. Motyw GRUB Breeze

Motyw GRUB Breeze:

```bash
sudo pacman -S --needed breeze-grub
```

Ustawienie:

```bash
grep -q '^GRUB_THEME=' /etc/default/grub \
  && sudo sed -i 's|^GRUB_THEME=.*|GRUB_THEME="/usr/share/grub/themes/breeze/theme.txt"|' /etc/default/grub \
  || echo 'GRUB_THEME="/usr/share/grub/themes/breeze/theme.txt"' | sudo tee -a /etc/default/grub

sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 7. Drukarka i skaner Brother DCP-B7520DW

Jeżeli chcesz obsługę drukarki/skanera Brother, doinstaluj:

```bash
sudo pacman -S --needed cups system-config-printer sane simple-scan avahi nss-mdns
sudo systemctl enable --now cups.service
sudo systemctl enable --now avahi-daemon.service
```

Pakiety Brother z AUR:

```bash
yay -S brother-dcp-b7520dw brscan4 brscan-skey
```

Konfiguracja skanera po IP:

```bash
sudo brsaneconfig4 -a name=Brother model=DCP-B7520DW ip=192.168.1.100
scanimage -L
```

Jeśli `scanimage -L` pokazuje urządzenie, skanowanie jest gotowe.

---

## 8. Brave — obejście wolnego startu na KDE

Jeśli Brave uruchamia się długo przez integrację z portfelem/secret service, możesz ustawić `--password-store=basic`.

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/brave-origin-nightly.desktop ~/.local/share/applications/

sed -i 's|^Exec=.*|Exec=brave --password-store=basic %U|' ~/.local/share/applications/brave-origin-nightly.desktop
update-desktop-database ~/.local/share/applications 2>/dev/null || true
XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental
```

---

## 9. iPhone — parowanie i dostęp do plików

Pakiety:

```bash
sudo pacman -S --needed gvfs gvfs-afc gvfs-gphoto2 libimobiledevice usbmuxd ifuse
```

Podłącz iPhone'a kablem, odblokuj ekran i wykonaj:

```bash
idevicepair pair
idevicepair validate
```

Na iPhonie potwierdź „Ufaj temu komputerowi”.

Jeśli Dolphin pokazuje błąd `Unhandled lockdownd code '-5'`:

```bash
sudo rm -rf /var/lib/lockdown/*
sudo systemctl restart usbmuxd
```

Odłącz/podłącz iPhone'a, odblokuj ekran i ponów:

```bash
idevicepair pair
idevicepair validate
```

Montaż ręczny:

```bash
mkdir -p ~/iPhone
ifuse ~/iPhone
```

Zdjęcia:

```text
~/iPhone/DCIM
```

Odmontowanie:

```bash
fusermount -u ~/iPhone
```

---

## 10. Bluetooth — AirPods Pro i sterowanie mediami

Do sterowania mediami z przycisków słuchawek Bluetooth:

```bash
systemctl --user enable --now mpris-proxy.service
```

Test MPRIS:

```bash
playerctl -l
playerctl play-pause
```

Jeśli nie masz `playerctl`:

```bash
sudo pacman -S --needed playerctl
```

---

## 11. Bluetooth — Xbox pad

Pakiet z AUR:

```bash
yay -S xpadneo-dkms
```

Po instalacji:

```bash
sudo reboot
```

---

## 12. Aktualizacje firmware Dell / LVFS

W bazowym minimalnym systemie `fwupd` nie był instalowany. Jeśli chcesz aktualizacje BIOS/UEFI/Thunderbolt/NVMe przez LVFS:

```bash
sudo pacman -S --needed fwupd
sudo systemctl enable --now fwupd-refresh.timer
```

Sprawdzenie:

```bash
fwupdmgr refresh
fwupdmgr get-updates
```

Aktualizacja:

```bash
fwupdmgr update
```

---

## 13. Snapshot po post-install

Po większych zmianach po instalacji:

```bash
sudo snapper -c root create --description "after post-install tweaks"
sudo snapper -c home create --description "home after post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Jeśli snapshot ma zostać na dłużej, oznacz go jako important:

```bash
sudo snapper -c root list
sudo snapper -c root modify --cleanup-algorithm important NUMER

sudo snapper -c home list
sudo snapper -c home modify --cleanup-algorithm important NUMER
```

---

## 18. Stan końcowy systemu po konfiguracji opcjonalnej

Po wykonaniu opcjonalnych kroków post-install możesz dodatkowo mieć:

- Plymouth splash
- motyw GRUB Breeze
- `yay`
- Brave
- Firefox
- Thunderbird
- Brother DCP-B7520DW
- obsługę iPhone'a
- AirPods Pro MPRIS
- Xbox pad przez `xpadneo`
- aktualizacje firmware przez `fwupd`
- firewall
- kodeki
- narzędzia diagnostyczne

