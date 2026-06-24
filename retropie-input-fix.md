# RetroPie Input Fix
**uConsole CM5 + QMK Firmware (j1n6) — Definitive Configuration**

> This guide documents the working input configuration after extensive testing.
> Every config file in this repo was verified via `evtest` on real hardware.
> If you're fighting "buttons not recognized" in RetroPie, this is the fix.

---

## Why nothing works out of the box

The uConsole keyboard exposes **two separate input devices**:

1. **Keyboard device** — sends standard keyboard keycodes for arrow keys, letters, etc.
2. **Joystick device** — sends axis events (D-pad) and button events (A/B/X/Y/Select/Start)

EmulationStation and RetroArch expect a single unified gamepad. The split causes
everything to fail or work partially.

**Making it worse:** every community config uses the wrong device name and GUID.
The actual kernel device name is `Clockwork uConsole Keyboard` — most configs use
`ClockworkPI uConsole`. ES cannot match the device, so buttons are dead.

**QMK firmware (j1n6) `Fn+G` gamepad mode** puts the D-pad on the joystick axes
alongside A/B/X/Y. Without Fn+G, the D-pad sends keyboard arrows and is on a
completely separate device from the buttons.

---

## Hardware facts (verified via evtest)

```
Joystick device: /dev/input/js0 → /dev/input/event2
Device name:     Clockwork uConsole Keyboard
Bus:     0x0003
Vendor:  0xfeed
Product: 0x0000
Version: 0x0111
SDL2 GUID: 030000fdfe0000000000000011010000
```

---

## Button mapping (confirmed via evtest in Fn+G gamepad mode)

| Physical button | evtest event | jstest # | ES/RetroArch id |
|---|---|---|---|
| **A** | BTN_TRIGGER (288) | 0 | `id="0"` |
| **B** | BTN_THUMB (289) | 1 | `id="1"` |
| **X** | BTN_THUMB2 (290) | 2 | `id="2"` |
| **Y** | BTN_TOP (291) | 3 | `id="3"` |
| **Select** | BTN_TOP2 (292) | 4 | `id="4"` |
| **Start** | BTN_PINKIE (293) | 5 | `id="5"` |
| **D-pad Up** | ABS_Y = -127 | axis 1 neg | `axis id="1" value="-1"` |
| **D-pad Down** | ABS_Y = 127 | axis 1 pos | `axis id="1" value="1"` |
| **D-pad Left** | ABS_X = -127 | axis 0 neg | `axis id="0" value="-1"` |
| **D-pad Right** | ABS_X = 127 | axis 0 pos | `axis id="0" value="1"` |
| **L** | mouse button | n/a | not on joystick |
| **R** | mouse button | n/a | not on joystick |

> **L and R are mouse buttons** on a separate device. They cannot be mapped as
> joystick L/R triggers. Set them to `nul` in RetroArch config.

---

## The fix — quick version

```bash
# 1. Remove all existing broken configs
sudo rm -f /opt/retropie/configs/all/emulationstation/es_input.cfg
sudo rm -rf /opt/retropie/configs/all/retroarch/autoconfig/*

# 2. Copy working configs from this repo
mkdir -p ~/.emulationstation
cp configs/es_input.cfg ~/.emulationstation/es_input.cfg
sudo cp configs/es_input.cfg \
  /opt/retropie/configs/all/emulationstation/es_input.cfg
sudo cp "configs/Clockwork uConsole Keyboard.cfg" \
  "/opt/retropie/configs/all/retroarch/autoconfig/Clockwork uConsole Keyboard.cfg"

# 3. Protect from being overwritten by ES on startup
sudo chattr +i ~/.emulationstation/es_input.cfg
sudo chattr +i /opt/retropie/configs/all/emulationstation/es_input.cfg

# 4. Reboot
sudo reboot
```

After reboot: **press Fn+G once** before doing anything. Then navigate and play.

---

## The fix — manual version

### EmulationStation config

Write to BOTH locations (they may be hardlinked — that's fine):

```bash
mkdir -p ~/.emulationstation

cat > ~/.emulationstation/es_input.cfg << 'EOF'
<?xml version="1.0"?>
<inputList>
  <inputAction type="onfinish">
    <command>/opt/retropie/supplementary/emulationstation/scripts/inputconfiguration.sh</command>
  </inputAction>
  <inputConfig type="joystick" deviceName="Clockwork uConsole Keyboard" deviceGUID="030000fdfe0000000000000011010000">
    <input name="a" type="button" id="0" value="1" />
    <input name="b" type="button" id="1" value="1" />
    <input name="x" type="button" id="2" value="1" />
    <input name="y" type="button" id="3" value="1" />
    <input name="select" type="button" id="4" value="1" />
    <input name="start" type="button" id="5" value="1" />
    <input name="hotkeyenable" type="button" id="4" value="1" />
    <input name="up" type="axis" id="1" value="-1" />
    <input name="down" type="axis" id="1" value="1" />
    <input name="left" type="axis" id="0" value="-1" />
    <input name="right" type="axis" id="0" value="1" />
  </inputConfig>
  <inputConfig type="keyboard" deviceName="Keyboard" deviceGUID="-1">
    <input name="up" type="key" id="1073741906" value="1" />
    <input name="down" type="key" id="1073741905" value="1" />
    <input name="left" type="key" id="1073741904" value="1" />
    <input name="right" type="key" id="1073741903" value="1" />
    <input name="a" type="key" id="13" value="1" />
    <input name="b" type="key" id="27" value="1" />
    <input name="start" type="key" id="13" value="1" />
    <input name="select" type="key" id="8" value="1" />
    <input name="hotkey" type="key" id="27" value="1" />
  </inputConfig>
</inputList>
EOF

sudo cp ~/.emulationstation/es_input.cfg \
  /opt/retropie/configs/all/emulationstation/es_input.cfg
```

### RetroArch autoconfig

```bash
sudo tee "/opt/retropie/configs/all/retroarch/autoconfig/Clockwork uConsole Keyboard.cfg" << 'EOF'
input_device = "Clockwork uConsole Keyboard"
input_driver = "udev"
input_a_btn = "0"
input_b_btn = "1"
input_x_btn = "2"
input_y_btn = "3"
input_select_btn = "4"
input_start_btn = "5"
input_l_btn = "nul"
input_r_btn = "nul"
input_up_axis = "-1"
input_down_axis = "+1"
input_left_axis = "-0"
input_right_axis = "+0"
input_enable_hotkey_btn = "4"
input_exit_emulator_btn = "5"
input_save_state_btn = "nul"
input_load_state_btn = "nul"
input_joypad_driver = "udev"
EOF
```

### Protect configs from being overwritten

ES runs `inputconfiguration.sh` on startup which overwrites your config. Lock it:

```bash
sudo chattr +i ~/.emulationstation/es_input.cfg
sudo chattr +i /opt/retropie/configs/all/emulationstation/es_input.cfg
```

To edit later: `sudo chattr -i ~/.emulationstation/es_input.cfg`

---

## Usage

**Every session:**
1. Boot → wait for EmulationStation to fully appear
2. Press **Fn+G** once → D-pad switches to gamepad mode
3. Navigate with D-pad, confirm with A, back with B
4. Launch a game → plays with full controls

**In-game hotkeys (Select = hotkey button):**
- **Select + Start** → exit game back to ES
- **Select + any button** → RetroArch hotkeys (depends on your mapping)
- **F1** → open RetroArch menu (keyboard)

---

## Known limitations

**L and R cannot be mapped** as gamepad triggers — they're mouse buttons on a
separate device. Games that require L/R (e.g. many PSX games, GBA games) need
a USB gamepad. The 8Bitdo range works perfectly with RetroPie plug-and-play.

**Fn+G must be pressed after every boot** and after RetroArch exits (the device
re-enumerates on USB). This is a hardware/firmware limitation, not a config issue.

**Keys may stop working when returning to ES from a game.** Press Fn+G again.
For extended gaming sessions, a USB gamepad is more reliable.

---

## Verifying your device name and GUID

If the fix doesn't work, confirm your exact device:

```bash
cat /proc/bus/input/devices | grep -B2 -A15 "js0"
```

The `Name=` line must match exactly what's in the config. If it differs, update
`deviceName` in `es_input.cfg` and `input_device` in the RetroArch cfg.

To recalculate the GUID from Bus/Vendor/Product/Version values (swap each pair
of bytes and concatenate):
- Bus `0003` → `03000000`
- Vendor `feed` → `fdfe0000`
- Product `0000` → `00000000`
- Version `0111` → `11010000`
- GUID: `030000fdfe0000000000000011010000`

---

## Debugging button IDs

To verify what each physical button sends:

```bash
# Press Fn+G first to enter gamepad mode, then:
sudo evtest /dev/input/event2 >> /tmp/buttons.txt
# Press each button, then Ctrl+C
cat /tmp/buttons.txt
```

The `code` field in the output maps to jstest button numbers:
- BTN_TRIGGER (288) = button 0
- BTN_THUMB (289) = button 1
- BTN_THUMB2 (290) = button 2
- BTN_TOP (291) = button 3
- BTN_TOP2 (292) = button 4
- BTN_PINKIE (293) = button 5
