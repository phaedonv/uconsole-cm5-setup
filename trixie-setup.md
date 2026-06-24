# Trixie Setup Guide
**Debian 13 Trixie on NVMe — ClockworkPi uConsole CM5 + HackerGadgets AIO V2**

> Trixie is the recommended main OS for the uConsole CM5. Ubuntu 24.04 was tested
> and rejected: GNOME is too heavy (laggy and hot even with V3D GPU accel confirmed),
> the RJ45 ethernet driver is missing, and snap apps crash. Trixie is the same Debian
> family as the Bookworm SD that ships with the device — ethernet, GPU, and no snaps
> all work out of the box.

---

## Target layout

```
NVMe
├── p1   ~512MB   FAT32   boot
├── p2   ~60GB    ext4    Trixie root   ← boots when no SD card inserted
└── p3   ~900GB   ext4    Shared data   ← all OSes mount this

SD cards (insert to switch OS, remove to return to Trixie)
├── RetroPie   retro gaming
├── Kali       pentesting
└── DragonOS   SDR / radio
```

**Boot logic:** CM5 EEPROM default (`BOOT_ORDER=0xf461`) = SD first, then NVMe.
No EEPROM changes needed — insert an SD to boot it, remove it to boot Trixie.

---

## ⚠️ Read these first

### The Dim Green Light of Death
Holding the power button to force-off puts the CM5 power management into an undefined
state. The green LED goes dim (not off) and the device won't boot until you pull the
batteries. **Never hold the power button. Always `sudo poweroff`** and wait for the
green LED to go **fully off**. If you're stuck at a black screen with dim green, open
the back and remove a battery for ~30 seconds.

### A freshly-flashed image will NOT boot from NVMe
ClockworkPi images are built for SD cards. On NVMe they fail silently — bright green
LED, no backlight, no ethernet lights. Two files need fixing before the first boot:

1. `cmdline.txt` has `root=/dev/mmcblk0p2` (the SD slot) — repoint to NVMe PARTUUID
2. `config.txt` has `dtparam=pciex1=off` — the NVMe and RJ45 ethernet both run on PCIe,
   so with it off you get a dead boot

Both fixes are in Step 4 below.

---

## Requirements

- A working SD card running Bookworm (the ClockworkPi default image)
- Trixie image: `ClockworkPi-Trixie-6.12.87.img.xz`
- SSH access or USB keyboard for running commands
- The NVMe already installed in the AIO board

---

## Step 1 — Flash Trixie to the NVMe

Boot from the Bookworm SD card. Transfer the Trixie image to the uConsole
(via SCP, USB stick, or whatever method works for you), then flash:

```bash
# Confirm NVMe is visible
lsblk    # look for nvme0n1 (~953.9G or similar)

# Optional: wipe any existing partition table first
sudo wipefs -a /dev/nvme0n1
sudo dd if=/dev/zero of=/dev/nvme0n1 bs=4M count=100 status=progress
sync

# Flash Trixie
xz -dc /path/to/ClockworkPi-Trixie-6.12.87.img.xz \
  | sudo dd of=/dev/nvme0n1 bs=4M status=progress conv=fsync
sync
```

---

## Step 2 — Disable auto-expansion

The image flashes a small (~7.6GB) root that auto-expands to fill the disk on first
boot. Disable that so you can control the partition layout:

```bash
sudo mkdir -p /mnt/nvme_root
sudo mount /dev/nvme0n1p2 /mnt/nvme_root
grep -i resize /mnt/nvme_root/etc/rc.local 2>/dev/null
```

If grep prints a line containing `resize2fs`, comment it out:

```bash
sudo nano /mnt/nvme_root/etc/rc.local
# Put # in front of the resize line
# Ctrl+O, Enter, Ctrl+X to save
```

```bash
sudo umount /mnt/nvme_root
```

---

## Step 3 — Partition: expand p2 to ~60GB, create shared p3

```bash
# Filesystem check (required before resize)
sudo e2fsck -fy /dev/nvme0n1p2

# Open parted
sudo parted /dev/nvme0n1
```

Inside parted:
```
print
# note p2 currently ends around 8GB

resizepart 2 63GB
# if "in use" warning appears, answer: Ignore

mkpart primary ext4 63GB 100%

print
# verify: p1 ~512MB | p2 ends 63GB | p3 fills the rest

quit
```

```bash
# Grow the filesystem to fill p2 (NO size argument — fills automatically)
sudo resize2fs /dev/nvme0n1p2

# Format the shared data partition
sudo mkfs.ext4 -L data /dev/nvme0n1p3

# Note these values — you'll need them in fstab
sudo blkid /dev/nvme0n1p2    # note PARTUUID (e.g. xxxxxxxx-02)
sudo blkid /dev/nvme0n1p3    # note UUID (for fstab on every OS)
```

---

## Step 4 — Fix the two boot files (critical)

```bash
sudo mkdir -p /mnt/nvme_boot
sudo mount /dev/nvme0n1p1 /mnt/nvme_boot

# Check current state
cat /mnt/nvme_boot/cmdline.txt
grep -i pciex1 /mnt/nvme_boot/config.txt
```

**Fix 1 — repoint root to NVMe** (use YOUR p2 PARTUUID from Step 3):
```bash
sudo sed -i 's|root=/dev/mmcblk0p2|root=PARTUUID=YOUR_P2_PARTUUID|' \
  /mnt/nvme_boot/cmdline.txt
```

**Fix 2 — enable PCIe** (only needed if config shows `pciex1=off`):
```bash
sudo sed -i 's|dtparam=pciex1=off|dtparam=pciex1|' /mnt/nvme_boot/config.txt
sudo sed -i '/^dtparam=pciex1$/a dtparam=pciex1_gen=3' /mnt/nvme_boot/config.txt
```

**Fix 3 — remove the resize flag from cmdline** (prevents auto-expand on boot):
```bash
sudo sed -i 's/ resize / /' /mnt/nvme_boot/cmdline.txt
```

Verify:
```bash
cat /mnt/nvme_boot/cmdline.txt    # should show root=PARTUUID=... no "resize"
grep pciex1 /mnt/nvme_boot/config.txt    # should show pciex1 and pciex1_gen=3, no =off
sudo umount /mnt/nvme_boot
```

> **Note:** The Trixie image (unlike Ubuntu) shipped with the correct root= and PCIe
> settings already. The `resize` flag removal is the only edit that was strictly needed.
> Always check before editing — don't apply fixes blindly.

---

## Step 5 — First NVMe boot

```bash
sudo poweroff    # watch green LED go FULLY off
```

Remove the SD card → single short press of power (never hold) → wait ~2 min.
Trixie boots to the LCD. Ethernet comes up automatically (PCIe is now enabled).

---

## Step 6 — Mount shared data partition

```bash
sudo mkdir -p /mnt/data

# Use YOUR p3 UUID from Step 3
echo 'UUID=YOUR_P3_UUID  /mnt/data  ext4  defaults,nofail  0  2' \
  | sudo tee -a /etc/fstab

sudo mount -a
df -h /mnt/data    # should show ~900GB available

# Create shared folder structure
mkdir -p /mnt/data/{Movies,Projects,Scripts,Wordlists,Logs,Captures}
mkdir -p /mnt/data/ROMs/{nes,snes,gba,gbc,gb,n64,psx,psp,mame,megadrive,nds,dreamcast,neogeo}

# Write UUID to a note on the partition itself
echo "p3 UUID: $(sudo blkid -s UUID -o value /dev/nvme0n1p3)" \
  > /mnt/data/README.txt

sudo chown -R $USER:$USER /mnt/data
```

---

## Step 7 — Install work tools

No snaps on Trixie — everything is a native .deb.

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  curl wget git vim tmux htop net-tools nmap dnsutils \
  wireguard-tools \
  remmina remmina-plugin-rdp freerdp3-x11 \
  retroarch python3-pip python3-venv build-essential \
  chromium firefox-esr

# Docker CE (official)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

# VS Code arm64
curl -Lo /tmp/code.deb \
  "https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-arm64"
sudo dpkg -i /tmp/code.deb && sudo apt install -f -y

# Mattermost — download arm64 .deb from https://mattermost.com/download/#mattermostApps
# sudo dpkg -i mattermost-desktop-*-linux-arm64.deb
```

### WireGuard (client only — split tunnel)
```bash
sudo cp your-client.conf /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf

# Manual control — do NOT enable on boot for a client
sudo wg-quick up wg0      # connect
sudo wg-quick down wg0    # disconnect
sudo wg show              # check status
```

### Optional — move Docker storage to the shared partition
Keeps the 60GB root from filling up with images:
```bash
sudo systemctl stop docker
sudo mkdir -p /mnt/data/docker
echo '{ "data-root": "/mnt/data/docker" }' | sudo tee /etc/docker/daemon.json
sudo rsync -a /var/lib/docker/ /mnt/data/docker/ 2>/dev/null || true
sudo systemctl start docker
docker info | grep "Docker Root Dir"
```

### RetroArch → shared ROMs
```bash
mkdir -p ~/.config/retroarch
echo 'rgui_browser_directory = "/mnt/data/ROMs"' >> ~/.config/retroarch/retroarch.cfg
```

### RDP to Windows servers
```bash
remmina &
# New connection → Protocol: RDP → server IP, username, domain
```

### Power optimisation
```bash
curl https://raw.githubusercontent.com/AdnanHodzic/auto-cpufreq/master/auto-cpufreq-installer \
  | sudo bash
sudo auto-cpufreq --install

# Lower minimum CPU frequency in boot config
sudo tee -a /boot/firmware/config.txt <<'EOF'
arm_freq_min=600
EOF
```

---

## Step 8 — Desktop theming (labwc / Wayland)

Trixie ships with **labwc** — a lightweight Wayland compositor. It's fast and
cool-running on the CM5. The desktop environment is the RPi desktop, which has
all the uConsole hardware support built in (battery widget, brightness keys, etc.).

### Theme packages
```bash
sudo apt install -y papirus-icon-theme materia-gtk-theme swaybg \
  fonts-firacode nwg-look fuzzel

# From source (recommended)
mkdir -p ~/Themes
cd ~/Themes
git clone https://github.com/vinceliuice/Graphite-gtk-theme
cd Graphite-gtk-theme && ./install.sh -d ~/.themes && cd ~/Themes

git clone https://github.com/Fausto-Korpsvart/Gruvbox-GTK-Theme
cd Gruvbox-GTK-Theme/themes && ./install.sh -d ~/.themes && cd ~/Themes

git clone https://github.com/vinceliuice/Tela-icon-theme
cd Tela-icon-theme && ./install.sh -d ~/.local/share/icons
```

### Apply a dark theme
```bash
# Use nwg-look (GUI) or gsettings directly
gsettings set org.gnome.desktop.interface gtk-theme 'Gruvbox-Dark'
gsettings set org.gnome.desktop.interface icon-theme 'Papirus-Dark'
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'

mkdir -p ~/.config/gtk-3.0
cat > ~/.config/gtk-3.0/settings.ini <<'EOF'
[Settings]
gtk-theme-name=Gruvbox-Dark
gtk-icon-theme-name=Papirus-Dark
gtk-application-prefer-dark-theme=1
EOF
```

### App launcher — fuzzel (Alt+Space)
```bash
mkdir -p ~/.config/fuzzel
cat > ~/.config/fuzzel/fuzzel.ini <<'EOF'
[main]
font=FiraCode:size=11

[colors]
background=282828ff
text=ebdbb2ff
match=d79921ff
selection=504945ff
selection-text=ebdbb2ff
border=928374ff
EOF

# Add to labwc keybinds — edit ~/.config/labwc/rc.xml
# Add inside <keyboard>:
# <keybind key="A-space">
#   <action name="Execute" command="fuzzel" />
# </keybind>
```

### Screen rotation (X11 sessions and greeter)
```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo tee /etc/X11/xorg.conf.d/90-uconsole-rotate.conf <<'EOF'
Section "Monitor"
    Identifier "DSI-2"
    Option "Rotate" "right"
EndSection
EOF
```

---

## SD card OSes

Each SD card is flashed with Raspberry Pi Imager (Device: Pi 5, OS: Use custom).
After flashing, boot from the card and add one fstab line to mount the shared data:

```bash
sudo mkdir -p /mnt/data
echo 'UUID=YOUR_P3_UUID  /mnt/data  ext4  defaults,nofail  0  2' \
  | sudo tee -a /etc/fstab
sudo mount -a
```

See [retropie-setup.md](retropie-setup.md) for the full RetroPie setup.

---

## Boot scenarios

| Want | Action |
|---|---|
| Trixie (main OS) | No SD card → power on |
| RetroPie | Insert RetroPie SD → power on |
| Kali | Insert Kali SD → power on |
| DragonOS | Insert DragonOS SD → power on |
| Return to Trixie | `sudo poweroff` → remove SD → power on |

---

## Troubleshooting

### NVMe won't boot — bright green LED, no backlight, no ethernet
PCIe is disabled or root= points at SD slot. Boot from Bookworm SD:
```bash
sudo mount /dev/nvme0n1p1 /mnt/nvme_boot
cat /mnt/nvme_boot/cmdline.txt        # check root=
grep pciex1 /mnt/nvme_boot/config.txt # check for =off
# Apply fixes from Step 4
sudo umount /mnt/nvme_boot && sudo poweroff
```

### Dim green LED, won't power on
Pull a battery ~30s, reinsert. Never hold the power button.

### Check GPU acceleration (run on device, not over SSH)
```bash
glxinfo -B | grep -i renderer    # want "V3D ...", not "llvmpipe"
```

### p3 not mounting
```bash
sudo blkid /dev/nvme0n1p3    # confirm UUID matches fstab entry
sudo mount /mnt/data          # run manually to see the error
```

### Ethernet works on Trixie/Bookworm but not Ubuntu
Expected — the Ubuntu image lacks the RJ45 driver. Use Trixie.
