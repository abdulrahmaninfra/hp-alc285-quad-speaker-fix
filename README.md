# HP ALC285 Quad-Speaker Fix (Linux)

Fix for muted/weak quad-speaker audio on HP laptops using the Realtek ALC285 codec on Linux. By default, only the tweeters are driven — the amplifiers and DSP aren't enabled — so audio comes out thin and quiet. This repo enables the amp GPIO at boot and ships an EasyEffects preset to restore full, balanced sound.

## The Problem

On Linux, the ALC285 codec on many HP laptops only drives the tweeters out of the box. The woofer amplifiers are never switched on because the required GPIO pin isn't set, and there's no equivalent of Windows' Realtek/Bang & Olufsen DSP chain to compensate. The result: quiet, tinny audio with no low end, even though the hardware supports full quad-speaker output.

## Tested / Supported Devices

Tested on:
- **HP EliteBook 845 G8**

Should also work on any HP laptop with the same ALC285 codec and speaker topology, including:

**HP EliteBook series**
- EliteBook 840 G7 / G8 / G9
- EliteBook 845 G7 / G8 / G9
- EliteBook 830 / 835 / 850 / 855 (G7–G9)

**HP ZBook series**
- ZBook Firefly 14 / 15 (G7 / G8 / G9)
- ZBook Power G8 / G9

**HP Envy & Spectre series**
- HP Envy x360 (models with ALC285 + quad speakers)
- HP Spectre x360 13/14/15 (models with Realtek ALC285)

If your device has ALC285 and quad speakers and isn't listed here, it's very likely compatible — please open an issue with your results so it can be added to this list.

## How It Works

1. A small script uses `hda-verb` to set GPIO pin `0x08` on the codec, which switches on the speaker amplifiers.
2. A `systemd` service runs that script once at every boot (as root), so the fix is automatic — no need to run anything manually after each restart.
3. An included EasyEffects preset shapes the now-audible full-range output into something that actually sounds good, using this pipeline:

   ```
   Autogain -> Convolver -> EQ -> Compressor -> Limiter
   ```

   - **Autogain** — normalizes input level before further processing.
   - **Convolver** — applies a correction impulse response to flatten the speakers' frequency response.
   - **EQ** — fine-tunes tonal balance (bass/mid/treble).
   - **Compressor** — evens out volume swings between quiet and loud passages.
   - **Limiter** — caps peak output to prevent clipping/distortion at high volume.

## Installation

### 1. Create the fix script

```bash
sudo nano /usr/local/bin/hp-sound-fix.sh
```

Paste in:

```bash
#!/bin/bash
/usr/bin/hda-verb /dev/snd/hwC1D0 0x01 SET_GPIO_DIR 0x08
/usr/bin/hda-verb /dev/snd/hwC1D0 0x01 SET_GPIO_DATA 0x08
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/hp-sound-fix.sh
```

> **Note:** `hwC1D0` may differ on your system. Run `aplay -l` to find the correct card/device index for your codec if the fix doesn't apply.

### 2. Create the systemd service

```bash
sudo nano /etc/systemd/system/hp-sound-fix.service
```

Paste in:

```ini
[Unit]
Description=HP EliteBook ALC285 Quad-Speaker Amps Fix
After=sound.target multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/hp-sound-fix.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

### 3. Enable and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hp-sound-fix.service
```

Check that it's running correctly:

```bash
sudo systemctl status hp-sound-fix.service
```

You should see `active (exited)` with no errors.

## Importing the EasyEffects Preset

1. Install EasyEffects if you don't already have it (available in most distro repos, e.g. `sudo apt install easyeffects` / `sudo dnf install easyeffects` / `sudo pacman -S easyeffects`).
2. Download [`alc285-configration.json`](./alc285-configration.json) from the root of this repo.
3. Open EasyEffects, and on the **Output** tab, open the menu (☰) in the top-right corner.
4. Go to **Presets** → **Import Preset**, and select the downloaded JSON file.
5. Select the imported preset from the list to load it, and make sure the **Output** effects toggle is switched on.
6. (Optional) To have it apply automatically at login, enable "Autostart on login" in EasyEffects' general settings, and make sure the preset stays loaded — EasyEffects remembers the last active preset by default.

Once the preset is loaded and the systemd service is active, the amps will be enabled at boot and audio will be routed through the corrective EQ/compression chain automatically — no manual steps needed after the first setup.

## Uninstalling

```bash
sudo systemctl disable --now hp-sound-fix.service
sudo rm /etc/systemd/system/hp-sound-fix.service
sudo rm /usr/local/bin/hp-sound-fix.sh
sudo systemctl daemon-reload
```

Then remove the EasyEffects preset from the Presets menu if you no longer want it.

## Contributing

If this fix works (or doesn't work) on a device not listed above, please open an issue with your model name and the output of `aplay -l`. Pull requests adding tested devices or improved presets are welcome.
