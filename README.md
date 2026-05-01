# Arch Linux — konfiguracja po instalacji

Ten plik jest przeznaczony jako osobne README dla repo z konfiguracją **po instalacji Archa**.

Założenie: bazowy system Arch już działa, masz Btrfs + Snapper + grub-btrfs, więc przed większymi zmianami robisz snapshot.

## Spis treści

- [1. Instalacja Firefox](#toc-01)
- [2. Katalog na klony / AUR / własne rzeczy](#toc-02)
- [3. Układ klawiatury GUI (Plasma)](#toc-03)
- [4. Fstrim](#toc-04)
- [5. Pakiety bazowe po instalacji](#toc-05)
- [6. Kodeki multimedialne](#toc-06)
- [7. `yay`](#toc-07)
- [8. Pakiety AUR](#toc-08)
- [9. Plymouth i splash screen](#toc-09)
- [10. Motyw GRUB Breeze](#toc-10)
- [11. Drukarka i skaner Brother DCP-B7520DW](#toc-11)
- [12. Brave — wolny start na KDE](#toc-12)
- [13. iPhone — parowanie i pliki](#toc-13)
- [14. Bluetooth — AirPods / sterowanie mediami](#toc-14)
- [15. Bluetooth — Xbox pad](#toc-15)
- [16. Aktualizacje firmware Dell / LVFS](#toc-16)
- [17. Snapshot po post-install](#toc-17)
- [18. Power Profiles Daemon](#toc-18)
- [19. Geolokalizacja — Night Light, strefa](#toc-19)
- [20. Geolokalizacja — widget pogody, strefa czasowa](#toc-20)
- [21. Zsh + Oh My Zsh + Powerlevel10k](#toc-21)
- [22. LazyVim](#toc-22)
  - [22.1. Motyw OneDark](#toc-23)
  - [22.2. Motyw OneHalfDark dla `bat`](#toc-24)
- [23. KDE Power Management — poziomy baterii](#toc-25)
- [24. Google Cloud CLI, Terraform, Go](#toc-26)

---

<a name="toc-01"></a>

## 1. Instalacja Firefox

```bash
sudo pacman -S --needed firefox 
```

<a name="toc-02"></a>

## 2. Katalog na klony/AUR/własne rzeczy

```bash
mkdir -p ~/.gc
```

<a name="toc-03"></a>

## 3. Wymuszenie polskiego układu klawiatury dla GUI

Systemowo masz `KEYMAP=pl`, ale dla GUI/Plasma możesz dodatkowo ustawić:

```bash
sudo localectl --no-convert set-x11-keymap pl
```

Sprawdzenie:

```bash
localectl status
```
Dodatkowo NumLock Settins > Keyboard > Numlock on startup > Turn on

<a name="toc-04"></a>

## 4. Fstrim

`fstrim.timer` warto mieć na SSD/NVMe:

```bash
sudo systemctl enable --now fstrim.timer
```

<a name="toc-05"></a>

## 5. Pakiety bazowe po instalacji

Przed większym blokiem zmian:

```bash
sudo snapper -c root create --description "before post-install tweaks"
sudo snapper -c home create --description "home before post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

```bash
sudo pacman -S --needed \
  bash-completion btop fastfetch openssh playerctl \
  zip unzip p7zip ark unrar \
  exfatprogs dosfstools \
  usbutils lsof smartmontools traceroute \
  wireguard-tools firefox thunderbird base-devel dkms btop fastfetch gcc spectacle \
  gwenview kcalc kdeplasma-addons eza ttf-jetbrains-mono-nerd ttf-nerd-fonts-symbols \
  ripgrep bat fd fzf micro most zoxide wl-clipboard python3
```

```bash
fc-cache -fv
```

---

<a name="toc-06"></a>

## 6. Kodeki multimedialne

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

<a name="toc-07"></a>

## 7. `yay`

Instalacja `yay`:

```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

<a name="toc-08"></a>

## 8. Pakiety AUR, jeśli ich chcesz:

```bash
yay -S brave-origin-nightly
```

---

<a name="toc-09"></a>

## 9. Plymouth i splash screen

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

<a name="toc-10"></a>

## 10. Motyw GRUB Breeze

Instalacja:

```
sudo pacman -S --needed breeze-grub
```

Kopia motywu na ESP:

```
sudo mkdir -p /boot/grub/themes
sudo cp -r /usr/share/grub/themes/breeze /boot/grub/themes/
```

Ustawienie w `/etc/default/grub`:

```
grep -q '^GRUB_THEME=' /etc/default/grub \
  && sudo sed -i 's|^GRUB_THEME=.*|GRUB_THEME="/boot/grub/themes/breeze/theme.txt"|' /etc/default/grub \
  || echo 'GRUB_THEME="/boot/grub/themes/breeze/theme.txt"' | sudo tee -a /etc/default/grub
```

Upewnij się, że tryb graficzny jest włączony (bez tego motyw się nie wyświetli):

```
grep -q '^GRUB_GFXMODE=' /etc/default/grub \
  || echo 'GRUB_GFXMODE=auto' | sudo tee -a /etc/default/grub

grep -q '^GRUB_GFXPAYLOAD_LINUX=' /etc/default/grub \
  || echo 'GRUB_GFXPAYLOAD_LINUX=keep' | sudo tee -a /etc/default/grub
```

Regeneracja konfiguracji:

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Weryfikacja — w wygenerowanym `grub.cfg` powinna pojawić się linia z `set theme=...`:

```
sudo grep -E 'set theme|set gfxmode|terminal_output' /boot/grub/grub.cfg
```

Oczekiwany efekt:

```
set gfxmode=auto
terminal_output gfxterm
set theme=($root)/grub/themes/breeze/theme.txt
```

Jeśli linia `set theme` się pojawia — restart i GRUB pokaże motyw Breeze.

<a name="toc-11"></a>

## 11. Drukarka i skaner Brother DCP-B7520DW

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

<a name="toc-12"></a>

## 12. Brave — obejście wolnego startu na KDE

Brave potrafi długo startować przez integrację z secret service/KWallet. Flaga `--password-store=basic` to obchodzi. Brave (zarówno nightly, jak i snapshot) instaluje **dwa** pliki `.desktop` — stary i nowy reverse-DNS — KDE używa nowego, więc trzeba zmodyfikować oba.

Sprzątanie poprzednich (potencjalnie zepsutych) kopii:

```
rm -f ~/.local/share/applications/brave*.desktop \
      ~/.local/share/applications/com.brave.*.desktop
```

Kopia świeżych oryginałów i wstrzyknięcie flagi (zachowuje pełną ścieżkę binarki niezależnie od kanału):

```
mkdir -p ~/.local/share/applications

for f in /usr/share/applications/brave-origin-nightly.desktop \
         /usr/share/applications/com.brave.Origin.nightly.desktop; do
  [ -f "$f" ] && cp "$f" ~/.local/share/applications/
done

for f in ~/.local/share/applications/brave-origin-nightly.desktop \
         ~/.local/share/applications/com.brave.Origin.nightly.desktop; do
  [ -f "$f" ] && sed -i 's|^Exec=\([^ ]*\)\(.*\)$|Exec=\1 --password-store=basic\2|' "$f"
done
```

Odświeżenie cache menu:

```
update-desktop-database ~/.local/share/applications 2>/dev/null || true
XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental
```

Weryfikacja — wszystkie linie `Exec=` powinny mieć `--password-store=basic` zaraz po nazwie binarki:

```
grep ^Exec= ~/.local/share/applications/brave-origin-nightly.desktop \
            ~/.local/share/applications/com.brave.Origin.nightly.desktop
```

Oczekiwany efekt (6 linii — 3 w każdym pliku):

```
Exec=/usr/bin/brave-origin-nightly --password-store=basic %U
Exec=/usr/bin/brave-origin-nightly --password-store=basic
Exec=/usr/bin/brave-origin-nightly --password-store=basic --incognito
```

Jeśli Brave dalej nie startuje, uruchom go z terminala żeby zobaczyć błąd:

```
/usr/bin/brave-origin-nightly --password-store=basic
```

<a name="toc-13"></a>

## 13. iPhone — parowanie i dostęp do plików

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

<a name="toc-14"></a>

## 14. Bluetooth — AirPods Pro i sterowanie mediami

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

<a name="toc-15"></a>

## 15. Bluetooth — Xbox pad

Pakiet z AUR:

```bash
yay -S xpadneo-dkms
```

Po instalacji:

```bash
sudo reboot
```

---

<a name="toc-16"></a>

## 16. Aktualizacje firmware Dell / LVFS

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

<a name="toc-17"></a>

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

<a name="toc-18"></a>

## 18. Power Profiles Daemon — przełącznik trybów zasilania w KDE

W Plasma w *Settings → Power and Battery* widać komunikat „Power profiles may be supported on your device” — w Arch pakiet nie jest instalowany domyślnie (w odróżnieniu od Fedory/Ubuntu).

```
sudo pacman -S --needed power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon.service
```

Po włączeniu w KDE pojawią się trzy profile (Performance / Balanced / Power Saver) oraz przełącznik na panelu systemowym.

Weryfikacja:

```
powerprofilesctl
```

**Uwaga:** `power-profiles-daemon` jest niekompatybilny z `tlp`. Nie instaluj obu naraz — wybierz jedno.

<a name="toc-19"></a>

## 19. Geolokalizacja — Night Light, automatyczna strefa

W Arch domyślnie nie ma demona lokalizacji (w Ubuntu/Fedorze jest preinstalowany). Bez niego **Night Light → Automatically detect location** nie działa — pokazuje pustą mapę.

Instalacja:

```
sudo pacman -S --needed geoclue
```

`geoclue` to usługa aktywowana przez D-Bus — uruchamia się sama gdy aplikacja jej potrzebuje. **Nie trzeba `systemctl enable`.**

Test:

```
/usr/lib/geoclue-2.0/demos/where-am-i
```

Po kilku sekundach powinno pokazać współrzędne i miasto. Następnie w *System Settings → Display & Monitor → Night Light → Day-Night Cycle* wykryje lokalizację automatycznie.

Jeśli aplikacje KDE dalej nie dostają lokalizacji, dodaj uprawnienia w `/etc/geoclue/geoclue.conf`:

```
[plasmashell]
allowed=true
system=true
users=

[kded6]
allowed=true
system=true
users=
```

<a name="toc-20"></a>

## 20. Geolokalizacja — Night Light, widget pogody, strefa czasowa

```
sudo pacman -S --needed geoclue
```

Backend `reallyfreegeoip` (MLS nie żyje od 2024):

```
sudo tee /etc/geoclue/conf.d/90-ip-method.conf > /dev/null << 'EOF'
[ip]
enable=true
method=reallyfreegeoip
EOF
```

Autostart agenta w KDE:

```
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/geoclue-agent.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=GeoClue Agent
Exec=/usr/lib/geoclue-2.0/demos/agent
NoDisplay=true
X-KDE-autostart-after=panel
EOF
```

Test:

```
/usr/lib/geoclue-2.0/demos/agent &
/usr/lib/geoclue-2.0/demos/where-am-i
```

Oczekiwane: `Description: GeoIP (reallyfreegeoip)`.

Opcjonalnie statyczna lokalizacja (wygrywa z IP gdy ma lepszą dokładność):

```
sudo tee /etc/geolocation > /dev/null << 'EOF'
50.0413
21.9990
100
200
EOF
```

Format: szerokość, długość, dokładność [m], wysokość [m] — każda wartość w osobnej linii.

<a name="toc-21"></a>

## 21. Zsh + Oh My Zsh + Powerlevel10k

Instalacja pakietów:

```bash
sudo pacman -S --needed \
  zsh git curl \
  zsh-autosuggestions zsh-syntax-highlighting
```

Instalacja Oh My Zsh:

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Instalacja motywu Powerlevel10k:

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"
```

W `~/.zshrc` ustaw motyw:

```zsh
ZSH_THEME="powerlevel10k/powerlevel10k"
```

Przykładowa lista pluginów w `~/.zshrc`:

```zsh
plugins=(
  git
  sudo
  fzf
  zoxide
)
```

Na końcu `~/.zshrc` dodaj pluginy z pacmana:

```zsh
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

Zmiana domyślnego shella użytkownika na Zsh:

```bash
sudo usermod -s /usr/bin/zsh "$USER"
```

Sprawdzenie:

```bash
getent passwd "$USER"
```

Linia użytkownika powinna kończyć się na:

```text
:/usr/bin/zsh
```

Jeśli Konsole nadal uruchamia Basha, zmień profil Konsole:

```text
Konsole → Settings → Manage Profiles → Edit Profile → General → Command
```

Ustaw:

```text
/usr/bin/zsh
```

albo zostaw pole `Command` puste, żeby Konsole używało domyślnego shella użytkownika.

Zamknij wszystkie okna Konsole i otwórz nowe.

Sprawdzenie aktywnego shella:

```bash
echo "$SHELL"
ps -p $$ -o comm=
```

Oczekiwany wynik:

```text
/usr/bin/zsh
zsh
```

Konfiguracja Powerlevel10k:

```bash
p10k configure
```

<a name="toc-22"></a>

## 22. LazyVim

Backup obecnej konfiguracji Neovima:

```bash
mv ~/.config/nvim ~/.config/nvim.bak
```

Opcjonalnie, ale zalecane:

```bash
mv ~/.local/share/nvim ~/.local/share/nvim.bak
mv ~/.local/state/nvim ~/.local/state/nvim.bak
mv ~/.cache/nvim ~/.cache/nvim.bak
```

Klonowanie LazyVim Starter:

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
```

Usunięcie katalogu `.git`, żeby później można było dodać konfigurację do własnego repo:

```bash
rm -rf ~/.config/nvim/.git
```

Pierwsze uruchomienie:

```bash
nvim
```

Po pierwszym uruchomieniu LazyVim pobierze pluginy.

<a name="toc-23"></a>

### 22.1. Motyw OneDark

Utwórz plik:

```bash
nvim ~/.config/nvim/lua/plugins/colorscheme.lua
```

Wklej:

```lua
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
```

Zsynchronizuj pluginy w Neovimie:

```vim
:Lazy sync
```

Zamknij i uruchom ponownie:

```bash
nvim
```

<a name="toc-24"></a>

### 22.2. Motyw OneHalfDark dla `bat`

Utwórz katalog konfiguracji:

```bash
mkdir -p ~/.config/bat
```

Utwórz plik konfiguracyjny:

```bash
nvim ~/.config/bat/config
```

Wklej:

```text
--theme="OneHalfDark"
--style="plain"
--paging=never
```

Test:

```bash
bat ~/.zshrc
```

<a name="toc-25"></a>

## 23. KDE Power Management — poziomy baterii

Ustaw zalecane poziomy baterii:

```text
Low level: 15%
Critical level: 7%
At critical level: Hibernate
```

Ścieżka w KDE Plasma:

```text
System Settings → Power Management → Advanced Power Settings
```

Ustaw:

```text
Battery Levels:
  Low level:       15%
  Critical level:  7%
  At critical:     Hibernate
```

Przy zużytej baterii niskie procenty oznaczają bardzo mało realnej energii. `Low level` na `15%` daje wcześniejsze ostrzeżenie, a `Critical level` na `7%` zostawia mały bufor przed hibernacją.

<a name="toc-26"></a>

## 24. Google Cloud CLI, Terraform, Go

Narzędzia GCP (CLI), Terraform i kompilator Go — typowo po zainstalowanym `yay` ([§ 7](#toc-07)):

```bash
yay -S google-cloud-cli gsutil
sudo pacman -S --needed terraform go python3 bind
```

`python3` jest też na liście w [§ 5](#toc-05); `--needed` nie nadpisuje nic bez potrzeby.

Po instalacji SDK (konto, projekt, domyślny region):

```bash
gcloud init
```
