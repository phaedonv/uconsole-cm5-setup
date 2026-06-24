# Lessons Learned
**CM5 uConsole gotchas, quirks, and hard-won knowledge**

---

## Power management

### The Dim Green Light of Death
**Never hold the power button to force off.** On the CM5, holding the button
puts the PMIC into an undefined state — the green LED goes dim (not off) and
the device won't power on again until you physically remove the batteries.

- **Correct:** `sudo poweroff` → wait for green LED to go **fully off**
- **Recovery:** open the back, remove a battery for ~30 seconds, reinsert
- **Root cause:** `POWER_OFF_ON_HALT=1` is set in EEPROM but doesn't fully work
  with the CM5's power rail arrangement

### `shutdown -h now` vs `poweroff`
Both should work with `POWER_OFF_ON_HALT=1` set. If the device halts but
doesn't power off (staying at dim green), use `sudo poweroff` instead.

---

## NVMe boot — the two mandatory fixes

Any ClockworkPi image flashed to NVMe will **silently fail to boot** (bright
green LED, no backlight, no ethernet) until you fix two files:

### Fix 1 — cmdline.txt: root= points at the SD slot
```
root=/dev/mmcblk0p2    ← wrong, this is the SD card
root=PARTUUID=xxxxxxxx-02    ← correct, this is the NVMe root partition
```

The kernel loads from the NVMe but then looks for its root filesystem on the
absent SD card → black screen.

### Fix 2 — config.txt: PCIe is disabled
```
dtparam=pciex1=off    ← wrong, kills everything on the PCIe bus
dtparam=pciex1        ← correct
dtparam=pciex1_gen=3  ← also needed for full speed
```

The HackerGadgets AIO board connects both the NVMe and the RJ45 ethernet
through the CM5's PCIe interface. With PCIe off, neither comes up — the device
appears to boot (green LED) but with nothing working.

### Trixie vs Ubuntu
The Trixie image ships with correct settings (PCIe on, root= already set to
PARTUUID). The Ubuntu 24.04 image ships with PCIe explicitly disabled.
Always check before applying fixes — don't assume.

---

## Ubuntu 24.04 — why we don't use it

Tested and rejected for the following confirmed reasons:

| Problem | Root cause | Fixable? |
|---|---|---|
| Lag + heat | GNOME too heavy for CM5, even with V3D GPU accel | Only by removing GNOME |
| No RJ45 ethernet | Missing RP1/RJ45 driver in the image | No easy fix |
| Firefox crashes | Snap runtime removed but snap Firefox still present | Fixed (install .deb Firefox) |
| PCIe disabled | `dtparam=pciex1=off` in config.txt | Fixed, but see above |

**GPU acceleration was confirmed working** (`V3D 7.1.10.2` via `glxinfo -B`) —
the lag is GNOME's weight, not a missing driver. Trixie with a lightweight
desktop (labwc, XFCE) runs noticeably cooler and snappier.

---

## Desktop environment on Trixie

Trixie ships with **labwc** (Wayland) as the default compositor, running the
RPi desktop. This is the recommended daily driver because:
- Battery widget works correctly
- Brightness/volume keys are pre-configured
- All ClockworkPi hardware is recognized

**XFCE** is available as an alternative:
```bash
sudo apt install -y xfce4 xfce4-goodies xfce4-whiskermenu-plugin
```
Select it at the lightdm login screen. Useful for a more traditional desktop
look, but the battery widget requires additional setup.

**The session picker** (lightdm) shows the ClockworkPi modal by default with
no session selection. To enable proper session switching:

```bash
sudo apt install -y lightdm-gtk-greeter
# In /etc/lightdm/lightdm.conf, under [Seat:*]:
# greeter-session=lightdm-gtk-greeter
# comment out: autologin-user=
# comment out: autologin-session=
```

---

## Screen rotation

The uConsole screen is physically rotated. Different layers handle it differently:

| Context | Setting | Location |
|---|---|---|
| Console/boot | `fbcon=rotate:1` | `/boot/firmware/cmdline.txt` |
| X11 / XFCE | xorg.conf rotate | `/etc/X11/xorg.conf.d/90-uconsole-rotate.conf` |
| RetroPie/ES | `display_rotate=1` | `/boot/firmware/config.txt` under `[pi5]` |
| lightdm greeter | same xorg.conf | (applies automatically) |

The confirmed output name for X11 rotation on CM5 is **`DSI-2`** (not DSI-1).

---

## QMK keyboard firmware

**j1n6 QMK firmware** (`clockworkpi_uconsole_default.bin`) is a significant
improvement over stock for desktop use:
- Smoother trackball (optimized CPI for 5" viewport)
- `Select + trackball` = scroll wheel
- `Fn+G` = toggle gamepad mode (all inputs on same joystick interface)
- `Fn+Space` = keyboard backlight toggle

### Flashing
```bash
cd ~/uconsole_keyboard_flash
sed -i 's/750/1500/g' maple_upload    # increase timeout
sudo ./maple_upload ttyACM0 2 1EAF:0003 clockworkpi_uconsole_default.bin
```

### Stock vs QMK on different OSes
| OS | Recommended firmware | Reason |
|---|---|---|
| Trixie | QMK | Better trackball, scroll, gamepad mode |
| RetroPie | QMK (with Fn+G) or stock | Stock simpler but D-pad + buttons can't work simultaneously |

The keyboard firmware is stored on the keyboard MCU, not the SD card or NVMe.
Swapping cards does NOT change the firmware.

### Backlight
The uConsole keyboard backlight is physically uneven — some keys have LEDs
directly underneath and others rely on diffusion. This is a hardware design
characteristic, not a firmware issue. The community-developed 3D-printable
diffuser (by j1n6) significantly improves light distribution.

---

## RetroPie input — root cause analysis

The single most common issue: **wrong device name and GUID in all community configs**.

- Community configs use: `ClockworkPI uConsole` / `030000fdaf1e...`
- Actual kernel device: `Clockwork uConsole Keyboard` / `030000fdfe0000000000000011010000`

EmulationStation uses SDL2 which matches devices by GUID. With the wrong GUID,
ES never identifies the joystick and all button mappings are ignored. The D-pad
appearing to work is actually the keyboard config (arrow keys), not the joystick.

See [retropie-input-fix.md](../retropie-input-fix.md) for the complete fix.

---

## HackerGadgets AIO V2 — PCIe peripheral notes

The AIO board exposes several peripherals through GPIO and PCIe:

| Peripheral | Interface | Notes |
|---|---|---|
| NVMe | PCIe (via `dtparam=pciex1`) | Must be enabled in config.txt |
| RJ45 ethernet | PCIe (same as NVMe) | Shares the same PCIe enable |
| USB3 ports | PCIe | Same |
| SDR (RTL-SDR) | GPIO power rail | GPIO 7 high by default on CM5 |
| GPS | GPIO power rail | Separate rail, needs explicit enable |
| LoRa | GPIO power rail | Separate rail, needs explicit enable |

The RTL-SDR powers on automatically on CM5 (GPIO 7 is high by default).
Other peripherals may need `aiov2_ctl --boot-rail <name> on`.

---

## WiFi antenna

The CM5 has its own U.FL connector for WiFi (separate from the AIO board's
U.FL). The correct config to use an external antenna:

```
dtparam=ant2
```

This is already set in ClockworkPi images. If WiFi signal is weak despite
having an external antenna, check:
1. The pigtail is fully clicked into the CM5's U.FL (tiny snap connector)
2. The antenna is not touching the metal chassis (causes significant attenuation)
3. A larger antenna improves signal substantially over the included stub

The CM5 WiFi and Bluetooth share the same antenna (`ant2` covers both).

---

## Battery life (CM5)

Real-world measurements with Samsung 35E 3500mAh cells:
- Idle at half brightness: ~5 hours
- Light work (SSH, terminal, web): ~3-4 hours
- SDR / intensive use: ~2 hours or less

The CM5 draws more power than the CM4. Optimization tips:
- Keep brightness at half or lower (biggest single factor)
- `auto-cpufreq` for automatic CPU scaling on battery
- `arm_freq_min=600` in config.txt
- WiFi power management enabled by default — no action needed

---

## Shared data partition across all OSes

All OSes mount the NVMe's p3 partition at `/mnt/data` via a single fstab line:

```
UUID=<your-p3-uuid>  /mnt/data  ext4  defaults,nofail  0  2
```

The `nofail` flag is important — it prevents a boot halt if the NVMe isn't
present for some reason. Only one OS runs at a time (SD card or NVMe, not both),
so there are no write conflicts.

UID consistency: creating the same username across all OSes results in UID 1000
everywhere (standard Linux behavior for the first user), so file ownership is
consistent without extra configuration.
