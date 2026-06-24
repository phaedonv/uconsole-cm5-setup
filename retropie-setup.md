# RetroPie Setup Guide
**ClockworkPi uConsole CM5 + HackerGadgets AIO V2**

> RetroPie lives on an SD card and boots when you insert it. The NVMe stays in
> place — all OSes share the same data partition (`/mnt/data`) for ROMs, BIOS,
> and captures. Nothing is duplicated.

---

## Target layout

```
NVMe p3 (shared data, mounted at /mnt/data on all OSes)
├── ROMs/          (nes snes gba gbc gb n64 psx psp megadrive dreamcast neogeo ...)
├── retroarch-system/   (BIOS files — SCPH1001.bin etc.)
└── Captures/

SD card (128GB recommended)
└── RetroPie image
```

---

## Step 1 — Flash the SD card

Use **Raspberry Pi Imager** on Windows or Linux:
1. Device → Raspberry Pi 5
2. OS → Use custom → `ClockworkPi-RetroPie-6.12.67.img.xz`
3. Storage → your SD card
4. Edit Settings:
   - Hostname: `uconsole-retro` (or your preference)
   - SSH: enabled, password auth
   - Username + password
   - WiFi credentials
5. Flash

---

## Step 2 — First boot

Insert SD → power on. RetroPie boots into EmulationStation.

On first launch it shows **"Hold any button on your gamepad"**.

> **If using QMK firmware:** press **Fn+G first** to enter gamepad mode, THEN
> hold a button. Without Fn+G, the gamepad buttons aren't recognized.
>
> **If using stock firmware:** just hold any button directly.

→ For the complete input fix (QMK + RetroPie), see [retropie-input-fix.md](retropie-input-fix.md).

To get a terminal: press **F4** on the keyboard, or SSH in:
```bash
ssh user@uconsole-retro.local
```

---

## Step 3 — Mount the shared data partition

SSH in or use F4 terminal:

```bash
sudo mkdir -p /mnt/data
echo 'UUID=YOUR_P3_UUID  /mnt/data  ext4  defaults,nofail  0  2' \
  | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/data    # should show your NVMe shared partition
```

> Get your p3 UUID from: `cat /mnt/data/README.txt` (if you created it during
> Trixie setup) or `sudo blkid /dev/nvme0n1p3` on any booted OS.

---

## Step 4 — Point RetroPie at shared ROMs and BIOS

```bash
# ROMs: replace RetroPie's default roms dir with a link to shared partition
mv ~/RetroPie/roms ~/RetroPie/roms.bak
ln -s /mnt/data/ROMs ~/RetroPie/roms

# BIOS: link to shared system folder
mv ~/RetroPie/BIOS ~/RetroPie/BIOS.bak 2>/dev/null || true
ln -s /mnt/data/retroarch-system ~/RetroPie/BIOS

# Restart EmulationStation to rescan
sudo systemctl restart emulationstation 2>/dev/null \
  || pkill emulationstation && emulationstation &
```

After restart, ES shows only systems that have ROMs in `/mnt/data/ROMs/`.

---

## Step 5 — ROM folder structure

RetroPie expects ROMs directly in each system subfolder (not nested). If your
ROMs arrived nested (e.g. `psx/PSX - PlayStation/game.cue`), flatten them:

```bash
cd /mnt/data/ROMs
for sys in */; do
  find "$sys" -mindepth 2 -type f -exec mv -t "$sys" {} + 2>/dev/null
  find "$sys" -mindepth 1 -type d -empty -delete 2>/dev/null
done

# Remove Windows scripts that may have come along
find /mnt/data/ROMs -name "*.ps1" -delete 2>/dev/null
find /mnt/data/ROMs -type d -name "images" -exec rm -rf {} + 2>/dev/null
```

> PSX games must be `.cue`+`.bin` or `.pbp` — NOT `.7z`. Extract compressed files:
> ```bash
> sudo apt install -y p7zip-full
> cd /mnt/data/ROMs/psx
> for f in *.7z; do 7z x "$f" && rm "$f"; done
> ```

---

## Step 6 — BIOS files

Put BIOS files in `/mnt/data/retroarch-system/` (linked as `~/RetroPie/BIOS/`).

| System | File(s) needed | Notes |
|---|---|---|
| PSX | `SCPH1001.bin` (US) / `SCPH7502.bin` (EU) | required |
| NeoGeo | `neogeo.zip` | put in ROMs/neogeo/ too |
| Dreamcast | `dc_boot.bin`, `dc_flash.bin` | required for Flycast |
| GBA | `gba_bios.bin` | optional, improves accuracy |
| PSP | none | PPSSPP is self-contained |
| N64 | none | core has it built in |

> Filenames are case-sensitive. Some cores expect lowercase:
> ```bash
> cd /mnt/data/retroarch-system
> cp SCPH1001.BIN scph1001.bin 2>/dev/null || true
> ```

---

## Step 7 — Screen rotation fix

Add to `/boot/firmware/config.txt` under the `[pi5]` section:

```bash
sudo nano /boot/firmware/config.txt
# Add under [pi5]:
# display_rotate=1
```

Also check `cmdline.txt` has `fbcon=rotate:1` for the console:
```bash
grep rotate /boot/firmware/cmdline.txt
```

---

## Emulator recommendations for CM5

| System | Best emulator | Notes |
|---|---|---|
| NES/SNES | lr-snes9x / lr-fceumm | flawless |
| GB/GBC/GBA | lr-mgba / lr-gambatte | flawless |
| Megadrive | lr-genesis-plus-gx | flawless |
| N64 | lr-mupen64plus-next | good; heavy games may drop frames |
| PSX | lr-pcsx-rearmed | excellent — ARM-optimized |
| PSP | standalone PPSSPP | runs better than lr-ppsspp |
| Dreamcast | lr-flycast | demanding; mixed results |
| NeoGeo | lr-fbneo | needs neogeo.zip |

Switch emulator per game via the **runcommand menu** (press any button in the
first 2 seconds when a game launches).

---

## Input reference

See [retropie-input-fix.md](retropie-input-fix.md) for the complete input fix.

Quick reference:

| Action | Key |
|---|---|
| Navigate ES menu | D-pad (keyboard arrows without Fn+G) |
| Select / confirm | A button (button 0 in gamepad mode) |
| Back | B button (button 1) |
| Open game menu | Start (button 5) |
| Exit game | Select + Start |
| Enter gamepad mode | Fn+G |

---

## Boot scenarios

| Want | Action |
|---|---|
| RetroPie | Insert this SD → power on |
| Trixie (main OS) | Remove SD → power on |
| Kali / DragonOS | Insert that SD → power on |

Always shut down cleanly: ES → Start → Quit → Shutdown, or `sudo poweroff`.
