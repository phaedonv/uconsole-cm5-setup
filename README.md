# uConsole CM5 Setup Guide

A practical, battle-tested setup guide for the **ClockworkPi uConsole with CM5 Lite**
and the **HackerGadgets AIO V2 upgrade kit**.

Everything here was discovered through real-world use, not theory. The goal is to save
you the hours of forum-diving and trial-and-error that went into building this.

---

## Hardware covered

- ClockworkPi uConsole CM5 Lite
- HackerGadgets AIO V2 (NVMe + RJ45 ethernet + USB3)
- NVMe SSD (any size — examples use 1TB)
- QMK keyboard firmware (j1n6 build)

---

## Contents

| Guide | What it covers |
|---|---|
| [trixie-setup.md](trixie-setup.md) | Flash Debian 13 Trixie to NVMe as the main OS, partition layout, work tools, theming |
| [retropie-setup.md](retropie-setup.md) | Flash RetroPie to an SD card, shared ROM/BIOS setup |
| [retropie-input-fix.md](retropie-input-fix.md) | **The definitive input fix** — correct GUID, button IDs, immutable config |
| [notes/lessons-learned.md](notes/lessons-learned.md) | CM5 gotchas, power management, PCIe, boot order |

### Ready-to-use config files

| File | Purpose |
|---|---|
| [configs/es_input.cfg](configs/es_input.cfg) | EmulationStation input config (copy to `~/.emulationstation/`) |
| [configs/Clockwork uConsole Keyboard.cfg](configs/Clockwork%20uConsole%20Keyboard.cfg) | RetroArch autoconfig (copy to `/opt/retropie/configs/all/retroarch/autoconfig/`) |

---

## Quick start

### Main OS (Trixie on NVMe)

Flash Trixie, fix the two boot files, partition the NVMe:

```bash
# After flashing with dd — fix root= (points at SD slot by default)
sudo sed -i 's|root=/dev/mmcblk0p2|root=PARTUUID=YOUR_P2_PARTUUID|' /mnt/nvme_boot/cmdline.txt

# Fix PCIe (disabled by default — kills NVMe + ethernet)
sudo sed -i 's|dtparam=pciex1=off|dtparam=pciex1|' /mnt/nvme_boot/config.txt
sudo sed -i '/^dtparam=pciex1$/a dtparam=pciex1_gen=3' /mnt/nvme_boot/config.txt
```

→ Full guide: [trixie-setup.md](trixie-setup.md)

### RetroPie input (the quick fix)

```bash
# Clone this repo on the uConsole
git clone https://github.com/phaedonv/uconsole-cm5-setup
cd uconsole-cm5-setup

# Copy the working configs
cp configs/es_input.cfg ~/.emulationstation/es_input.cfg
sudo cp configs/es_input.cfg /opt/retropie/configs/all/emulationstation/es_input.cfg
sudo cp "configs/Clockwork uConsole Keyboard.cfg" \
  "/opt/retropie/configs/all/retroarch/autoconfig/Clockwork uConsole Keyboard.cfg"

# Protect from being overwritten by ES on startup
sudo chattr +i ~/.emulationstation/es_input.cfg
sudo chattr +i /opt/retropie/configs/all/emulationstation/es_input.cfg
```

Press **Fn+G** after boot to activate gamepad mode. → Full guide: [retropie-input-fix.md](retropie-input-fix.md)

---

## The storage architecture

```
NVMe (1TB example)
├── p1   ~512MB   FAT32   boot
├── p2   ~60GB    ext4    Trixie root   ← boots with no SD card
└── p3   ~900GB   ext4    Shared data   ← all OSes mount this

SD cards (swap to switch OS)
├── RetroPie SD   → retro gaming
├── Kali SD       → pentesting
└── DragonOS SD   → SDR / radio
```

All OSes mount the shared partition at `/mnt/data` via one fstab line.
ROMs, wordlists, captures, and projects are shared with zero duplication.

---

## CM5 power rule (read this first)

**Never hold the power button to force off.** It triggers the "Dim Green Light of Death"
where the device won't power on until you physically remove the batteries.

Always use `sudo poweroff` and wait for the green LED to go **fully off**.

---

## Contributing

PRs welcome — especially for:
- Kali SD card input config (same GUID issue likely applies)
- DragonOS-specific setup notes
- Trackball sensitivity tuning
- Additional emulator-specific configs

---

## Credits

- **Rex** (ClockworkPi forums) — RetroPie image for uConsole
- **j1n6** — QMK firmware port with gamepad mode and trackball improvements
- ClockworkPi community for documenting the hardware quirks
