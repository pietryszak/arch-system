Arch Linux — konfiguracja po instalacji

Ten plik jest przeznaczony jako osobne README dla repo z konfiguracją po instalacji Archa.

Założenie: bazowy system Arch już działa, masz Btrfs + Snapper + grub-btrfs, więc przed większymi zmianami robisz snapshot.

Spis treści

- 1. Subvolumy profili — fstab i mount -a (#toc-01)
- 2. Fstrim (#toc-02)
- 3. Instalacja Firefox (#toc-03)
- 4. Katalog na klony/AUR/własne rzeczy (#toc-04)
- 5. Wymuszenie polskiego układu klawiatury dla GUI (#toc-05)
- 6. Pakiety bazowe po instalacji (#toc-06)
- 7. Kodeki multimedialne (#toc-07)
- 8. yay (#toc-08)
- 9. Pakiety AUR, jeśli ich chcesz: (#toc-09)
- 10. Plymouth i splash screen (#toc-10)
- 11. Motyw GRUB Breeze (#toc-11)
- 12. Drukarka i skaner Brother DCP-B7520DW (#toc-12)
- 13. Vivaldi Snapshot — obejście wolnego startu na KDE (#toc-13)
- 14. iPhone — parowanie i dostęp do plików (#toc-14)
- 15. Bluetooth — AirPods Pro i sterowanie mediami (#toc-15)
- 16. Bluetooth — Xbox pad (#toc-16)
- 17. Aktualizacje firmware Dell / LVFS (#toc-17)
- 18. Snapshot po post-install (#toc-18)
- 19. Power Profiles Daemon — przełącznik trybów zasilania w KDE (#toc-19)
- 20. Geolokalizacja — Night Light, automatyczna strefa (#toc-20)
- 21. Geolokalizacja — Night Light, widget pogody, strefa czasowa (#toc-21)
- 22. Zsh + Oh My Zsh + Powerlevel10k (#toc-22)
- 23. LazyVim (#toc-23)
  - 23.1. Motyw OneDark (#toc-23-1)
  - 23.2. Motyw OneHalfDark dla bat (#toc-23-2)
- 24. KDE Power Management — poziomy baterii (#toc-24)
- 25. Google Cloud CLI, Terraform, Go (#toc-25)
- 26. KVM, libvirt i virt-manager (#toc-26)
  - 26.1. Weryfikacja sprzętu i kernela (#toc-26-1)
  - 26.2. Instalacja pakietów (#toc-26-2)
  - 26.3. Usługi i grupy (#toc-26-3)
  - 26.4. Domyślna sieć NAT (#toc-26-4)
  - 26.5. Domyślny URI dla virsh (#toc-26-5)
  - 26.6. Nested virtualization (#toc-26-6)
  - 26.7. Sterowniki virtio-win (Windows) (#toc-26-7)
  - 26.8. Pierwsza maszyna wirtualna (#toc-26-8)
  - 26.9. Bridged networking (#toc-26-9)
  - 26.10. Shared folders (virtiofs) (#toc-26-10)
  - 26.11. Przydatne komendy (#toc-26-11)

1. Subvolumy profili — fstab i mount -a

W arch-install (§4) tworzysz subvolumy @mozilla, @vivaldi, @vivaldi-snapshot, @thunderbird, ale nie montujesz ich w instalatorze (żeby useradd w §8 czysto skopiował /etc/skel). Tutaj — już na żywym systemie, zalogowany jako pietryszak — dopisujesz wpisy do /etc/fstab i montujesz profile.

UUID Btrfsa wyciągamy z findmnt, użytkownik z ${USER}:

FS_UUID=$(findmnt -no UUID /)
echo "$FS_UUID"

OPTS="noatime,compress=zstd:3,ssd,space_cache=v2,discard=async"

sudo tee -a /etc/fstab >/dev/null <<EOF
UUID=${FS_UUID} /home/${USER}/.mozilla                 btrfs ${OPTS},subvol=/@mozilla          0 0
UUID=${FS_UUID} /home/${USER}/.config/vivaldi          btrfs ${OPTS},subvol=/@vivaldi          0 0
UUID=${FS_UUID} /home/${USER}/.config/vivaldi-snapshot btrfs ${OPTS},subvol=/@vivaldi-snapshot 0 0
UUID=${FS_UUID} /home/${USER}/.thunderbird             btrfs ${OPTS},subvol=/@thunderbird      0 0
EOF

Punkty montowania, montaż i właściciel: katalogi utworzone jako root podczas instalacji albo puste subvolumy po pierwszym mount często mają root:root na granicy mountpointu — bez chown Firefox / Thunderbird nie zapiszą profilu. Ustaw siebie na całym drzewie pod tymi ścieżkami:

mkdir -p ~/.mozilla ~/.config/vivaldi ~/.config/vivaldi-snapshot ~/.thunderbird
sudo systemctl daemon-reload
sudo mount -a

sudo chown -R "${USER}:${USER}" \
  ~/.mozilla ~/.config/vivaldi ~/.config/vivaldi-snapshot ~/.thunderbird

findmnt -R /home

Uwaga — Snapper: te subvolumy są zagnieżdżone wewnątrz /home, więc snapper -c home nie obejmuje ich plików. Jeśli chcesz mieć snapshoty profili, dodaj im własne konfiguracje (snapper -c vivaldi-snapshot create-config /home/${USER}/.config/vivaldi-snapshot itd.) albo backup poza Snapperem.

2. Fstrim

fstrim.timer warto mieć na SSD/NVMe:

sudo systemctl enable --now fstrim.timer

3. Instalacja Firefox

sudo pacman -S --needed firefox 

4. Katalog na klony/AUR/własne rzeczy

mkdir -p ~/.gc

5. Wymuszenie polskiego układu klawiatury dla GUI

Systemowo masz KEYMAP=pl, ale dla GUI/Plasma możesz dodatkowo ustawić:

sudo localectl --no-convert set-x11-keymap pl

Sprawdzenie:

localectl status
Dodatkowo NumLock Settins > Keyboard > Numlock on startup > Turn on

6. Pakiety bazowe po instalacji

Przed większym blokiem zmian:

sudo snapper -c root create --description "before post-install tweaks"
sudo snapper -c home create --description "home before post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg

sudo pacman -S --needed \
  bash-completion btop fastfetch openssh playerctl \
  zip unzip p7zip ark unrar \
  exfatprogs dosfstools \
  usbutils lsof smartmontools traceroute \
  wireguard-tools firefox thunderbird base-devel dkms gcc spectacle \
  gwenview kcalc kdeplasma-addons eza ttf-jetbrains-mono-nerd ttf-nerd-fonts-symbols \
  ripgrep bat fd fzf micro most zoxide wl-clipboard python3 \
  plasma-browser-integration partitionmanager

fc-cache -fv

7. Kodeki multimedialne

Minimalny system ma część bibliotek multimedialnych jako zależności KDE, ale do pełniejszej obsługi audio/wideo warto doinstalować:

sudo pacman -S --needed \
  ffmpeg \
  gst-plugins-base gst-plugins-good gst-plugins-bad gst-plugins-ugly gst-libav \
  gst-plugin-pipewire

Opcjonalnie miniatury filmów i dodatkowe formaty obrazów w KDE/Dolphin:

sudo pacman -S --needed \
  ffmpegthumbs \
  kimageformats \
  qt6-imageformats

libdvdcss instaluj tylko, jeśli faktycznie potrzebujesz odtwarzania szyfrowanych DVD:

sudo pacman -S --needed libdvdcss

8. yay

Instalacja yay:

cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si

9. Pakiety AUR, jeśli ich chcesz:

yay -S vivaldi-snapshot mediawriter

10. Plymouth i splash screen

Instalacja Plymouth:

sudo pacman -S --needed plymouth plymouth-kcm

Motyw z AUR:

yay -S plymouth-theme-arch-breeze-git

Ustawienie motywu:

sudo plymouth-set-default-theme -R arch-breeze

Dodanie splash do GRUB:

sudo grep -q 'splash' /etc/default/grub || \
  sudo sed -i 's/\(GRUB_CMDLINE_LINUX_DEFAULT="[^"]*\)"/\1 splash"/' /etc/default/grub

Dodanie hooka plymouth do mkinitcpio przy układzie systemd + sd-encrypt:

sudo grep -q ' plymouth ' /etc/mkinitcpio.conf || \
  sudo sed -i 's/\<sd-vconsole\>/sd-vconsole plymouth/' /etc/mkinitcpio.conf

grep '^HOOKS=' /etc/mkinitcpio.conf

Oczekiwany układ z Plymouth:

HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole plymouth block sd-encrypt filesystems fsck)

Przebudowa:

sudo mkinitcpio -P
sudo grub-mkconfig -o /boot/grub/grub.cfg

11. Motyw GRUB Breeze

Instalacja:

sudo pacman -S --needed breeze-grub

Kopia motywu na ESP:

sudo mkdir -p /boot/grub/themes
sudo cp -r /usr/share/grub/themes/breeze /boot/grub/themes/

Ustawienie w /etc/default/grub:

grep -q '^GRUB_THEME=' /etc/default/grub \
  && sudo sed -i 's|^GRUB_THEME=.*|GRUB_THEME="/boot/grub/themes/breeze/theme.txt"|' /etc/default/grub \
  || echo 'GRUB_THEME="/boot/grub/themes/breeze/theme.txt"' | sudo tee -a /etc/default/grub

Upewnij się, że tryb graficzny jest włączony (bez tego motyw się nie wyświetli):

grep -q '^GRUB_GFXMODE=' /etc/default/grub \
  || echo 'GRUB_GFXMODE=auto' | sudo tee -a /etc/default/grub

grep -q '^GRUB_GFXPAYLOAD_LINUX=' /etc/default/grub \
  || echo 'GRUB_GFXPAYLOAD_LINUX=keep' | sudo tee -a /etc/default/grub

Regeneracja konfiguracji:

sudo grub-mkconfig -o /boot/grub/grub.cfg

Weryfikacja — w wygenerowanym grub.cfg powinna pojawić się linia z set theme=...:

sudo grep -E 'set theme|set gfxmode|terminal_output' /boot/grub/grub.cfg

Oczekiwany efekt:

set gfxmode=auto
terminal_output gfxterm
set theme=($root)/grub/themes/breeze/theme.txt

Jeśli linia set theme się pojawia — restart i GRUB pokaże motyw Breeze.

12. Drukarka i skaner Brother DCP-B7520DW

Jeżeli chcesz obsługę drukarki/skanera Brother, doinstaluj:

sudo pacman -S --needed cups system-config-printer sane simple-scan avahi nss-mdns
sudo systemctl enable --now cups.service
sudo systemctl enable --now avahi-daemon.service

Pakiety Brother z AUR:

yay -S brother-dcp-b7520dw brscan4 brscan-skey

Konfiguracja skanera po IP:

sudo brsaneconfig4 -a name=Brother model=DCP-B7520DW ip=192.168.1.100
scanimage -L

Jeśli scanimage -L pokazuje urządzenie, skanowanie jest gotowe.

13. Vivaldi Snapshot — obejście wolnego startu na KDE

Ten rozdział dotyczy wyłącznie kanału snapshot (vivaldi-snapshot z AUR). Stabilny pakiet vivaldi z repozytoriów ma osobny profil w ~/.config/vivaldi, snapshot — w ~/.config/vivaldi-snapshot; przy obu zainstalowanych naraz w ~/.config będą dwa katalogi — to normalne i nie oznacza, że „snapshot nadpisuje” stabilny.

Vivaldi (Chromium) potrafi długo startować przez integrację z secret service/KWallet. Flaga --password-store=basic to obchodzi. Pakiet vivaldi-snapshot instaluje vivaldi-snapshot.desktop w /usr/share/applications/ — skopiuj go do ~/.local/share/applications/ i dopisz flagę do każdej linii Exec=. Jeśli zobaczysz drugi plik w stylu reverse-DNS (com.vivaldi.*), dodaj go do pętli tak samo.

Sprzątanie poprzednich (potencjalnie zepsutych) kopii:

rm -f ~/.local/share/applications/vivaldi-snapshot.desktop \
      ~/.local/share/applications/com.vivaldi.*.desktop

Kopia świeżego oryginału i wstrzyknięcie flagi:

mkdir -p ~/.local/share/applications

for f in /usr/share/applications/vivaldi-snapshot.desktop; do
  [ -f "$f" ] && cp "$f" ~/.local/share/applications/
done

for f in ~/.local/share/applications/vivaldi-snapshot.desktop; do
  [ -f "$f" ] && sed -i 's|^Exec=\([^ ]*\)\(.*\)$|Exec=\1 --password-store=basic\2|' "$f"
done

Odświeżenie cache menu:

update-desktop-database ~/.local/share/applications 2>/dev/null || true
XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental

Weryfikacja — każda linia Exec= powinna mieć --password-store=basic zaraz po ścieżce binarki:

grep ^Exec= ~/.local/share/applications/vivaldi-snapshot.desktop

Jeśli Vivaldi dalej nie startuje, uruchom go z terminala, żeby zobaczyć błąd:

/usr/bin/vivaldi-snapshot --password-store=basic

14. iPhone — parowanie i dostęp do plików

Pakiety:

sudo pacman -S --needed gvfs gvfs-afc gvfs-gphoto2 libimobiledevice usbmuxd ifuse

Podłącz iPhone'a kablem, odblokuj ekran i wykonaj:

idevicepair pair
idevicepair validate

Na iPhonie potwierdź „Ufaj temu komputerowi”.

Jeśli Dolphin pokazuje błąd Unhandled lockdownd code '-5':

sudo rm -rf /var/lib/lockdown/*
sudo systemctl restart usbmuxd

Odłącz/podłącz iPhone'a, odblokuj ekran i ponów:

idevicepair pair
idevicepair validate

Montaż ręczny:

mkdir -p ~/iPhone
ifuse ~/iPhone

Zdjęcia:

~/iPhone/DCIM

Odmontowanie:

fusermount -u ~/iPhone

15. Bluetooth — AirPods Pro i sterowanie mediami

Do sterowania mediami z przycisków słuchawek Bluetooth:

systemctl --user enable --now mpris-proxy.service

Test MPRIS:

playerctl -l
playerctl play-pause

Jeśli nie masz playerctl:

sudo pacman -S --needed playerctl

16. Bluetooth — Xbox pad

Pakiet z AUR:

yay -S xpadneo-dkms

Po instalacji:

sudo reboot

17. Aktualizacje firmware Dell / LVFS

W bazowym minimalnym systemie fwupd nie był instalowany. Jeśli chcesz aktualizacje BIOS/UEFI/Thunderbolt/NVMe przez LVFS:

sudo pacman -S --needed fwupd
sudo systemctl enable --now fwupd-refresh.timer

Sprawdzenie:

fwupdmgr refresh
fwupdmgr get-updates

Aktualizacja:

fwupdmgr update

18. Snapshot po post-install

Po większych zmianach po instalacji:

sudo snapper -c root create --description "after post-install tweaks"
sudo snapper -c home create --description "home after post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg

Jeśli snapshot ma zostać na dłużej, oznacz go jako important:

sudo snapper -c root list
sudo snapper -c root modify --cleanup-algorithm important NUMER

sudo snapper -c home list
sudo snapper -c home modify --cleanup-algorithm important NUMER

19. Power Profiles Daemon — przełącznik trybów zasilania w KDE

W Plasma w Settings → Power and Battery widać komunikat „Power profiles may be supported on your device” — w Arch pakiet nie jest instalowany domyślnie (w odróżnieniu od Fedory/Ubuntu).

sudo pacman -S --needed power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon.service

Po włączeniu w KDE pojawią się trzy profile (Performance / Balanced / Power Saver) oraz przełącznik na panelu systemowym.

Weryfikacja:

powerprofilesctl

Uwaga: power-profiles-daemon jest niekompatybilny z tlp. Nie instaluj obu naraz — wybierz jedno.

20. Geolokalizacja — Night Light, automatyczna strefa

W Arch domyślnie nie ma demona lokalizacji (w Ubuntu/Fedorze jest preinstalowany). Bez niego Night Light → Automatically detect location nie działa — pokazuje pustą mapę.

Instalacja:

sudo pacman -S --needed geoclue

geoclue to usługa aktywowana przez D-Bus — uruchamia się sama gdy aplikacja jej potrzebuje. Nie trzeba systemctl enable.

Test:

/usr/lib/geoclue-2.0/demos/where-am-i

Po kilku sekundach powinno pokazać współrzędne i miasto. Następnie w System Settings → Display & Monitor → Night Light → Day-Night Cycle wykryje lokalizację automatycznie.

Jeśli aplikacje KDE dalej nie dostają lokalizacji, dodaj uprawnienia w /etc/geoclue/geoclue.conf:

[plasmashell]
allowed=true
system=true
users=

[kded6]
allowed=true
system=true
users=

21. Geolokalizacja — Night Light, widget pogody, strefa czasowa

sudo pacman -S --needed geoclue

Backend reallyfreegeoip (MLS nie żyje od 2024):

sudo tee /etc/geoclue/conf.d/90-ip-method.conf > /dev/null << 'EOF'
[ip]
enable=true
method=reallyfreegeoip
EOF

Autostart agenta w KDE:

mkdir -p ~/.config/autostart

cat > ~/.config/autostart/geoclue-agent.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=GeoClue Agent
Exec=/usr/lib/geoclue-2.0/demos/agent
NoDisplay=true
X-KDE-autostart-after=panel
EOF

Test:

/usr/lib/geoclue-2.0/demos/agent &
/usr/lib/geoclue-2.0/demos/where-am-i

Oczekiwane: Description: GeoIP (reallyfreegeoip).

Opcjonalnie statyczna lokalizacja (wygrywa z IP gdy ma lepszą dokładność):

sudo tee /etc/geolocation > /dev/null << 'EOF'
50.0413
21.9990
100
200
EOF

Format: szerokość, długość, dokładność [m], wysokość [m] — każda wartość w osobnej linii.

22. Zsh + Oh My Zsh + Powerlevel10k

Instalacja pakietów:

sudo pacman -S --needed \
  zsh git curl \
  zsh-autosuggestions zsh-syntax-highlighting

Instalacja Oh My Zsh:

sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

Instalacja motywu Powerlevel10k:

git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"

W ~/.zshrc ustaw motyw:

ZSH_THEME="powerlevel10k/powerlevel10k"

Przykładowa lista pluginów w ~/.zshrc:

plugins=(
  git
  sudo
  fzf
  zoxide
)

Na końcu ~/.zshrc dodaj pluginy z pacmana:

source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

Zmiana domyślnego shella użytkownika na Zsh:

sudo usermod -s /usr/bin/zsh "$USER"

Sprawdzenie:

getent passwd "$USER"

Linia użytkownika powinna kończyć się na:

:/usr/bin/zsh

Jeśli Konsole nadal uruchamia Basha, zmień profil Konsole:

Konsole → Settings → Manage Profiles → Edit Profile → General → Command

Ustaw:

/usr/bin/zsh

albo zostaw pole Command puste, żeby Konsole używało domyślnego shella użytkownika.

Zamknij wszystkie okna Konsole i otwórz nowe.

Sprawdzenie aktywnego shella:

echo "$SHELL"
ps -p $$ -o comm=

Oczekiwany wynik:

/usr/bin/zsh
zsh

Konfiguracja Powerlevel10k:

p10k configure

23. LazyVim

Backup obecnej konfiguracji Neovima:

mv ~/.config/nvim ~/.config/nvim.bak

Opcjonalnie, ale zalecane:

mv ~/.local/share/nvim ~/.local/share/nvim.bak
mv ~/.local/state/nvim ~/.local/state/nvim.bak
mv ~/.cache/nvim ~/.cache/nvim.bak

Klonowanie LazyVim Starter:

git clone https://github.com/LazyVim/starter ~/.config/nvim

Usunięcie katalogu .git, żeby później można było dodać konfigurację do własnego repo:

rm -rf ~/.config/nvim/.git

Pierwsze uruchomienie:

nvim

Po pierwszym uruchomieniu LazyVim pobierze pluginy.

23.1. Motyw OneDark

Utwórz plik:

nvim ~/.config/nvim/lua/plugins/colorscheme.lua

Wklej:

return {
  {
    "navarasu/onedark.nvim",
    priority = 1000,
    lazy = false,
    opts = {
      style = "darker",
    },
  },

  {
    "LazyVim/LazyVim",
    opts = {
      colorscheme = "onedark",
    },
  },
}

Zsynchronizuj pluginy w Neovimie:

:Lazy sync

Zamknij i uruchom ponownie:

nvim

23.2. Motyw OneHalfDark dla bat

Utwórz katalog konfiguracji:

mkdir -p ~/.config/bat

Utwórz plik konfiguracyjny:

nvim ~/.config/bat/config

Wklej:

--theme="OneHalfDark"
--style="plain"
--paging=never

Test:

bat ~/.zshrc

24. KDE Power Management — poziomy baterii

Ustaw zalecane poziomy baterii:

Low level: 15%
Critical level: 7%
At critical level: Hibernate

Ścieżka w KDE Plasma:

System Settings → Power Management → Advanced Power Settings

Ustaw:

Battery Levels:
  Low level:       15%
  Critical level:  7%
  At critical:     Hibernate

Przy zużytej baterii niskie procenty oznaczają bardzo mało realnej energii. Low level na 15% daje wcześniejsze ostrzeżenie, a Critical level na 7% zostawia mały bufor przed hibernacją.

25. Google Cloud CLI, Terraform, Go

Narzędzia GCP (CLI), Terraform i kompilator Go — typowo po zainstalowanym yay (§ 8 (#toc-08)):

yay -S google-cloud-cli gsutil
sudo pacman -S --needed terraform go python3 bind

python3 jest też na liście w § 6 (#toc-06); --needed nie nadpisuje nic bez potrzeby.

Po instalacji SDK (konto, projekt, domyślny region):

gcloud init

26. KVM, libvirt i virt-manager

Kompletny setup wirtualizacji na Arch Linux: KVM/QEMU, libvirt, virt-manager.

Sprzęt referencyjny: Dell Latitude 5421, Intel i7-11850H (Tiger Lake H, 16 wątków), 32 GB RAM.

26.1. Weryfikacja sprzętu i kernela

Sprawdź, czy CPU wspiera wirtualizację i czy moduły KVM są załadowane:

LC_ALL=C lscpu | grep -E "Virtualization|vmx"
lsmod | grep kvm

Oczekiwane:

- w lscpu: Virtualization: VT-x (Intel) lub AMD-V; w flagach CPU obecność vmx (Intel) lub svm (AMD);
- w lsmod: moduły kvm oraz kvm_intel (Intel) lub kvm_amd (AMD).

Jeśli brak kvm_intel / kvm_amd — włącz Virtualization Support (VT-x / SVM) w firmware. Na Dellu: F2 przy starcie → Virtualization Support.

26.2. Instalacja pakietów

sudo pacman -S --needed qemu-full libvirt virt-manager virt-viewer \
  dnsmasq openbsd-netcat \
  swtpm edk2-ovmf dmidecode libguestfs

| Pakiet | Rola |
|--------|------|
| qemu-full | Emulator QEMU (lżej: qemu-desktop — tylko x86_64) |
| libvirt | Daemon zarządzający VM |
| virt-manager | Graficzny menedżer maszyn wirtualnych |
| virt-viewer | Klient SPICE/VNC do okien gościa |
| dnsmasq | DHCP/DNS dla domyślnej sieci NAT |
| openbsd-netcat | Pomocniczo przy łączeniu z innymi hostami libvirt |
| swtpm | Emulacja TPM 2.0 (np. Windows 11) |
| edk2-ovmf | Firmware UEFI dla VM (Secure Boot, nowoczesne OS) |
| dmidecode | Informacje o sprzęcie hosta |
| libguestfs | Narzędzia do pracy na obrazach dysków offline |

Uwaga: pakiet bridge-utils został wycofany z repozytoriów Arch — mostki konfiguruje się przez iproute2 (ip link), które jest w systemie domyślnie. Nie instaluj bridge-utils.

26.3. Usługi i grupy

sudo systemctl enable --now libvirtd.service
sudo usermod -aG libvirt,kvm "$USER"

virtlogd i virtnetworkd używają aktywacji przez socket — uruchomią się w razie potrzeby.

Po dodaniu do grup wyloguj się i zaloguj ponownie (albo tymczasowo: newgrp libvirt). Weryfikacja:

groups | grep -E "libvirt|kvm"

26.4. Domyślna sieć NAT

sudo virsh net-autostart default
sudo virsh net-start default

Weryfikacja:

virsh net-list --all
ip a show virbr0

Interfejs virbr0 powinien mieć adres w stylu 192.168.122.1/24. Goście dostają adresy z tej podsieci i wychodzą na świat przez NAT hosta.

26.5. Domyślny URI dla virsh

Bez jawnej konfiguracji virsh bez sudo może łączyć się z sesją użytkownika (qemu:///session), gdzie inaczej wygląda stan sieci i listy domen. Dla spójności z libvirtd systemowym ustaw:

# ~/.bashrc lub ~/.zshrc
export LIBVIRT_DEFAULT_URI="qemu:///system"

Przeładuj powłokę, potem:

virsh net-list --all

Powinna być widoczna sieć default w stanie active z Autostart yes.

virt-manager przy członkostwie w grupie libvirt zwykle i tak wskazuje qemu:///system — powyższe jest przede wszystkim dla CLI.

26.6. Nested virtualization (opcjonalnie)

Umożliwia np. Dockera lub kolejne VM wewnątrz gościa.

Intel:

echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf

AMD:

echo "options kvm_amd nested=1" | sudo tee /etc/modprobe.d/kvm-amd.conf

Po rebootcie (lub przeładowaniu modułów) sprawdź np.:

cat /sys/module/kvm_intel/parameters/nested   # Intel: oczekiwane Y lub 1
# AMD: /sys/module/kvm_amd/parameters/nested

26.7. Sterowniki virtio-win (tylko dla gościa Windows)

Pomiń, jeśli planujesz wyłącznie Linuxa w VM.

Z AUR (po zainstalowanym yay — sekcja 8 (#toc-08)):

yay -S virtio-win

Jeśli budowa z AUR się wywali (np. problem z fedorapeople.org), możesz pobrać ISO stabilnej linii:

mkdir -p ~/Downloads
curl -L -o ~/Downloads/virtio-win.iso \
  https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

Alternatywnie czasem działa bezpośredni link z Koji (wersja w ścieżce zmienia się wraz z wydaniami — sprawdź aktualny katalog na kojipkgs.fedoraproject.org (https://kojipkgs.fedoraproject.org/packages/virtio-win/)):

curl -L -o ~/Downloads/virtio-win.iso \
  'https://kojipkgs.fedoraproject.org/packages/virtio-win/0.1.285/1.fc41/data/virtio-win-0.1.285.iso'

ISO podpinasz jako drugi napęd optyczny podczas instalacji Windows — ułatwia sterowniki virtio-net i virtio-scsi / dyski VirtIO.

26.8. Pierwsza maszyna wirtualna

virt-manager

Skrót workflow:

1. File → New Virtual Machine → Local install media (ISO) — wskaż ISO, typ OS (często wykryje się sam).
2. RAM / vCPU — przy ~32 GB RAM na hoście typowo 8 GB RAM i 4 vCPU to bezpieczny punkt wyjścia dla jednej gościnnej stacji roboczej.
3. Dysk — często 40–80 GB, format qcow2 (alokacja dynamiczna).
4. Zaznacz Customize configuration before install — wygodnie ustawić firmware, dysk i sieć przed pierwszym startem.

W oknie konfiguracji m.in.:

- Overview → Firmware: UEFI (OVMF_CODE.fd) dla nowoczesnych systemów.
- CPUs → Configuration: Copy host CPU configuration (host passthrough — pełniejsza zgodność/wydajność, świadomie używaj na laptopach).
- CPUs → Topology: sensownie 1 socket × N rdzeni × 1 wątek (dopasuj do obciążenia hosta).
- Disk → Bus: VirtIO (szybsze niż emulowany SATA).
- NIC → Device model: virtio (szybsze niż domyślne e1000).
- Display: Spice; Video: QXL lub Virtio (dla Linuxa często lepszy Virtio).
- Windows 11: Add Hardware → TPM → Emulated, version 2.0 (wraz z UEFI/OVMF).

26.9. Bridged networking (opcjonalnie)

Domyślny NAT (virbr0) wystarcza w większości przypadków. Bridge ma sens, gdy VM ma być pełnoprawnym hostem w LAN (osobny MAC/IP jak fizyczna maszyna).

Uwaga: mostkowanie Wi-Fi w praktyce bywa problematyczne; stabilniej Ethernet (przykładowa nazwa interfejsu enp0s31f6 — podmień na swoją z ip link).

Przykład z NetworkManager:

nmcli con add type bridge ifname br0 con-name br0
nmcli con add type bridge-slave ifname enp0s31f6 master br0
nmcli con up br0

W virt-manager: NIC → Network source: Bridge device → br0.

26.10. Shared folders (virtiofs)

Udostępnianie katalogu z hosta do gościa przez virtiofs (szybka ścieżka I/O).

W virt-manager: Add Hardware → Filesystem → Driver: virtiofs (ustaw tag i katalog hosta).

Gość Linux:

sudo mount -t virtiofs <tag> /mnt/shared

Gość Windows: zainstaluj WinFsp oraz usługę VirtIO-FS z ISO virtio-win.

26.11. Przydatne komendy

# Lista domen (VM)
virsh list --all

# Start / graceful shutdown / force off
virsh start <nazwa>
virsh shutdown <nazwa>
virsh destroy <nazwa>

# Snapshoty
virsh snapshot-create-as <nazwa> snap1 "opis"
virsh snapshot-list <nazwa>
virsh snapshot-revert <nazwa> snap1

# Edycja XML
virsh edit <nazwa>

# Konsola szeregowa (jeśli skonfigurowana w XML)
virsh console <nazwa>

# Sieci i pule storage
virsh net-list --all
virsh pool-list --all

Domyślna lokalizacja obrazów dysków (pula default): /var/lib/libvirt/images/.
