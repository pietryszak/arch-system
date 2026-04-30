# Arch Linux — konfiguracja po instalacji

Ten plik jest przeznaczony jako osobne README dla repo z konfiguracją **po instalacji Archa**.

Założenie: bazowy system Arch już działa, masz Btrfs + Snapper + grub-btrfs, więc przed większymi zmianami robisz snapshot.

---

## 1. Pakiety bazowe po instalacji, opcjonalne

Przed większym blokiem zmian:

```bash
sudo snapper -c root create --description "before post-install tweaks"
sudo snapper -c home create --description "home before post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Sensowny zestaw bez nadmiarowego śmietnika:

```bash
sudo pacman -S --needed \
  bash-completion btop fastfetch openssh playerctl \
  zip unzip p7zip \
  exfatprogs dosfstools \
  usbutils lsof smartmontools traceroute \
  wireguard-tools
```

`firewalld` instaluj w sekcji **16. Bezpieczne SSH tylko w domu**, jeśli chcesz firewall i SSH ograniczone do domowej sieci LAN.

`unrar`, `mtools`, `net-tools`, `networkmanager-openvpn` instaluj tylko wtedy, gdy faktycznie ich potrzebujesz.

`fstrim.timer` warto mieć na SSD/NVMe:

```bash
sudo systemctl enable --now fstrim.timer
```

---

## 2. Kodeki multimedialne

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

## 3. Firefox, Thunderbird, Brave

Firefox i Thunderbird z repo:

```bash
sudo pacman -S --needed firefox thunderbird
```

Brave z AUR, więc najpierw `yay`.

---

## 4. `base-devel` i `yay`

Do AUR potrzebujesz `base-devel`.

```bash
sudo pacman -S --needed base-devel
```

Instalacja `yay`:

```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

Opcjonalny katalog na klony/AUR/własne rzeczy:

```bash
mkdir -p ~/.gc
```

Pakiety AUR, jeśli ich chcesz:

```bash
yay -S brave-bin brother-dcp-b7520dw brscan4 brscan-skey xpadneo-dkms plymouth-theme-arch-breeze-git
```

Uwaga: `xpadneo-dkms` wymaga `dkms` i nagłówków kernela. `linux-headers` powinien być już w bazowej instalacji, ale jeśli go nie masz:

```bash
sudo pacman -S --needed linux-headers dkms
```

Po instalacji `xpadneo-dkms` najlepiej zrobić restart.

---

## 5. Plymouth i splash screen

W bazowej instalacji celowo nie było Plymouth, bo minimalny system i hibernacja/LUKS/TPM powinny najpierw działać bez upiększeń.

Instalacja Plymouth:

```bash
sudo pacman -S --needed plymouth plymouth-kcm
```

Jeśli używasz motywu z AUR:

```bash
yay -S plymouth-theme-arch-breeze-git
```

Ustawienie motywu:

```bash
sudo plymouth-set-default-theme -R arch-breeze
```

Powrót do zwykłego Breeze:

```bash
sudo pacman -S --needed breeze-plymouth
sudo plymouth-set-default-theme -R breeze
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

Uwaga: przy LUKS + TPM2 PIN najpierw upewnij się, że zwykły boot i hibernacja działają. Plymouth dodawaj dopiero później.

---

## 6. Motyw GRUB Breeze

Jeśli chcesz motyw GRUB Breeze:

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

## 7. Wymuszenie polskiego układu klawiatury dla GUI

Systemowo masz `KEYMAP=pl`, ale dla GUI/Plasma możesz dodatkowo ustawić:

```bash
sudo localectl --no-convert set-x11-keymap pl
```

Sprawdzenie:

```bash
localectl status
```

---

## 8. Dell Latitude 5421 — fix DPTF throttling po hibernacji / monitorach USB-C

Fix na przypadek, gdy Dell Latitude 5421 po zewnętrznych monitorach USB-C/Thunderbolt lub po hibernacji dławi CPU do około 200 MHz.

Utwórz usługę:

```bash
sudo tee /etc/systemd/system/fix-dptf-throttle.service << 'EOF'
[Unit]
Description=Unbind DPTF proc_thermal after resume
After=hibernate.target suspend.target hybrid-sleep.target suspend-then-hibernate.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'sleep 2 && echo 0000:00:04.0 > /sys/bus/pci/drivers/proc_thermal/unbind 2>/dev/null || true'

[Install]
WantedBy=hibernate.target suspend.target hybrid-sleep.target suspend-then-hibernate.target
EOF

sudo systemctl enable fix-dptf-throttle.service
```

Diagnostyka, gdy CPU znowu spada do bardzo niskich taktowań:

```bash
cat /proc/cpuinfo | grep "MHz" | head -4
```

Ręczny fix:

```bash
sudo sh -c 'echo 0000:00:04.0 > /sys/bus/pci/drivers/proc_thermal/unbind'
```

Uwaga: po odpięciu DPTF kernel nadal ma własne zabezpieczenia termiczne, ale po tej zmianie warto obserwować temperatury przez kilka dni, np. w `btop`.

---

## 9. Drukarka i skaner Brother DCP-B7520DW

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

## 10. Brave — obejście wolnego startu na KDE

Jeśli Brave uruchamia się długo przez integrację z portfelem/secret service, możesz ustawić `--password-store=basic`.

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/brave-browser.desktop ~/.local/share/applications/

sed -i 's|^Exec=.*|Exec=brave --password-store=basic %U|' ~/.local/share/applications/brave-browser.desktop
update-desktop-database ~/.local/share/applications 2>/dev/null || true
XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental
```

Sprawdzenie:

```bash
grep '^Exec=' ~/.local/share/applications/brave-browser.desktop
```

Od tej chwili Brave z menu aplikacji użyje `--password-store=basic`.

---

## 11. iPhone — parowanie i dostęp do plików

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

## 12. Bluetooth — AirPods Pro i sterowanie mediami

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

## 13. Bluetooth — Xbox pad

Pakiet z AUR:

```bash
yay -S xpadneo-dkms
```

Po instalacji:

```bash
sudo reboot
```

---

## 14. Aktualizacje firmware Dell / LVFS

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

## 15. Ukrywanie niechcianych wpisów w menu KDE

Nie usuwać pakietów typu `avahi`, `qt6-tools`, `v4l-utils`, jeśli są zależnościami. Lepiej ukryć wpisy `.desktop`.

Sprawdzenie właścicieli:

```bash
pacman -Qo /usr/share/applications/*avahi* 2>/dev/null
pacman -Qo /usr/share/applications/*qt* 2>/dev/null
pacman -Qo /usr/share/applications/*qv4l* 2>/dev/null
pacman -Qo /usr/share/applications/*emoji* 2>/dev/null
```

Sprawdzenie zależności:

```bash
pacman -Qi avahi qt6-tools v4l-utils plasma-workspace | grep -E 'Name|Required By|Optional For'
```

Ukrycie Avahi, qv4l2 i Emoji Selector:

```bash
mkdir -p ~/.local/share/applications

for f in \
  /usr/share/applications/avahi-discover.desktop \
  /usr/share/applications/qv4l2.desktop \
  /usr/share/applications/org.kde.plasma.emojier.desktop
do
  cp "$f" ~/.local/share/applications/
  grep -q '^NoDisplay=true' ~/.local/share/applications/"$(basename "$f")" || \
    echo 'NoDisplay=true' >> ~/.local/share/applications/"$(basename "$f")"
done

XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental
systemctl --user restart plasma-plasmashell.service
```

Przywrócenie:

```bash
rm ~/.local/share/applications/avahi-discover.desktop
rm ~/.local/share/applications/qv4l2.desktop
rm ~/.local/share/applications/org.kde.plasma.emojier.desktop
XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental
```

---

## 16. Bezpieczne SSH tylko w domu

Cel:

```text
sshd działa
root przez SSH zablokowany
logowanie hasłem wyłączone
logowanie tylko kluczem SSH
firewall wpuszcza SSH tylko z domowej podsieci LAN
publiczna strefa firewalld nie ma otwartego SSH
```

### 16.1 Instalacja narzędzi i firewalla

Jeśli nie były jeszcze zainstalowane:

```bash
sudo pacman -S --needed \
  bash-completion btop fastfetch openssh playerctl \
  zip unzip p7zip \
  exfatprogs dosfstools \
  usbutils lsof smartmontools traceroute \
  wireguard-tools \
  firewalld
```

Włączenie `sshd` i `firewalld`:

```bash
sudo systemctl enable --now sshd
sudo systemctl enable --now firewalld
```

Sprawdzenie:

```bash
systemctl is-enabled sshd
systemctl is-active sshd

sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

### 16.2 Firewall: SSH tylko z domowej podsieci

Najpierw sprawdź adres IP laptopa i podsieć:

```bash
ip -4 addr show wlp0s20f3
ip route
```

Przykład:

```text
192.168.1.123/24
```

Wtedy domowa podsieć to:

```text
192.168.1.0/24
```

W poniższych komendach podmień `192.168.1.0/24`, jeśli Twoja sieć ma inny zakres.

Usuń SSH ze strefy `public`:

```bash
sudo firewall-cmd --permanent --zone=public --remove-service=ssh
```

Utwórz osobną strefę dla domowego SSH:

```bash
sudo firewall-cmd --permanent --new-zone=home-ssh
```

Dodaj domową podsieć:

```bash
sudo firewall-cmd --permanent --zone=home-ssh --add-source=192.168.1.0/24
```

Pozwól na SSH tylko w tej strefie:

```bash
sudo firewall-cmd --permanent --zone=home-ssh --add-service=ssh
```

Przeładuj firewalla:

```bash
sudo firewall-cmd --reload
```

Sprawdź:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --zone=home-ssh --list-all
```

Oczekiwany wynik logiczny:

```text
public:
  interfaces: wlp0s20f3
  services: dhcpv6-client
  brak ssh

home-ssh:
  sources: 192.168.1.0/24
  services: ssh
```

### 16.3 Hardening SSH — etap bezpieczny, jeszcze z hasłem

Najpierw ustaw konfigurację tak, żeby nie odciąć się przed dodaniem klucza:

```bash
sudo mkdir -p /etc/ssh/sshd_config.d

sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication yes
KbdInteractiveAuthentication no
PubkeyAuthentication yes

X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no

MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

AllowUsers pietryszak
EOF

sudo sshd -t
sudo systemctl restart sshd
```

Sprawdzenie realnej konfiguracji:

```bash
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|x11forwarding|allowtcpforwarding|allowagentforwarding|permittunnel|allowusers'
```

Na tym etapie oczekiwane:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication yes
kbdinteractiveauthentication no
x11forwarding no
allowtcpforwarding no
allowagentforwarding no
allowusers pietryszak
permittunnel no
```

### 16.4 Dodanie klucza SSH

Na komputerze, z którego będziesz się łączyć:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_arch
```

Skopiuj klucz na laptopa z Archem:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_arch.pub pietryszak@IP_ARCHA
```

Test:

```bash
ssh -i ~/.ssh/id_ed25519_arch pietryszak@IP_ARCHA
```

Jeżeli klucz działa, można wyłączyć logowanie hasłem.

Jeśli nie używasz `ssh-copy-id`, dodaj klucz lokalnie na Archu do:

```text
/home/pietryszak/.ssh/authorized_keys
```

i ustaw prawa:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 16.5 Finalnie: SSH tylko kluczem, bez hasła

Dopiero po potwierdzeniu, że logowanie kluczem działa:

```bash
sudo sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config.d/99-hardening.conf
sudo sshd -t
sudo systemctl restart sshd
```

Sprawdzenie:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin|allowusers'
```

Oczekiwane:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
allowusers pietryszak
```

Finalny stan bezpieczeństwa:

```text
SSH działa tylko dla użytkownika pietryszak
root login przez SSH jest zablokowany
hasła przez SSH są zablokowane
działają tylko klucze SSH
firewalld wpuszcza SSH tylko z 192.168.1.0/24
publiczna strefa nie ma otwartego SSH
```

### 16.6 Snapshot po zabezpieczeniu SSH

Po potwierdzeniu, że logowanie kluczem działa:

```bash
sudo snapper -c root create --description "secure ssh key-only home LAN"
sudo snapper -c home create --description "home after secure ssh key-only"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 17. Snapshot po post-install

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
- bezpieczne SSH tylko z domowej sieci LAN, tylko kluczem
- kodeki
- narzędzia diagnostyczne
- fix DPTF throttling na Dell Latitude 5421
