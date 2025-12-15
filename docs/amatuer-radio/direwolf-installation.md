# Direwolf Installation on Ubuntu

The steps below take you from a clean Ubuntu/Lubuntu workstation to a working Direwolf software TNC that can talk to your radio interface, expose AGW/KISS ports, and integrate with the AX.25 user-space tools.

## 1. Install Required Packages
=== "Terminal"
```bash
sudo apt update
sudo apt install direwolf ax25-apps ax25-tools alsa-utils
```

## 2. Grant Device Permissions
Direwolf needs access to serial ports, audio devices, and (on some laptops) PulseAudio. Add your user to the appropriate groups, then **log out/in or reboot** so the new membership takes effect. **Ensure that `$USER` is replaced by the current Linux user.**

=== "Bash"
```bash
sudo usermod -aG dialout "$USER"
sudo usermod -aG audio "$USER"
sudo usermod -aG plugdev "$USER"
sudo usermod -aG pulse-access "$USER"
```

!!! warning
    If you skip the logout/login step you will receive "Permission denied" errors when Direwolf tries to open `/dev/ttyUSB*` or the sound card.

## 3. Identify the Correct Sound Card
Run the following commands to list playback/record devices. Record the **card** and **device** numbers for the interface connected to your radio (USB dongle, Digirig, SignaLink, etc.).

=== "Bash"
```bash
aplay -l
arecord -l
```

Example (USB interface on card 0, device 0): `card 0: Audio [USB Audio], device 0: USB Audio`.

## 4. Set Volume Levels with `alsamixer`
1. Launch the mixer:
   ```bash
   alsamixer
   ```
2. Press **F6** and choose the sound card you will use for packet.
3. Make sure the playback and capture sliders are unmuted (`OO` instead of `MM`).
4. Start with speaker/output around 75%, microphone/input around 10%, disable AGC if present.
5. Save the levels:
   ```bash
   sudo alsactl store
   ```

## 5. Locate the Serial/PTT Interface
Determine which device file your PTT interface uses. For USB adapters you will typically see `/dev/ttyUSB0` or `/dev/ttyACM0`.

=== "Bash"
```bash
ls -l /dev/ttyUSB* /dev/ttyACM*
```

If you rely on Raspberry Pi GPIO or CAT control, note the pin numbers instead. We will reference the path inside `direwolf.conf`.

## 6. Create `direwolf.conf`
Use the information gathered above to create a basic configuration. The snippet below assumes:
- Sound card: card 0, device 0
- Callsign: `VK4XSS-1`
- PTT interface: `/dev/ttyUSB0`

=== "direwolf.conf"
```ini
# Force Direwolf to use the USB audio interface (card 0, device 0)
ADEVICE plughw:0,0 plughw:0,0
ACHANNELS 1
CHANNEL 0

# Station identity
MYCALL VK4XSS-1

# Modem + network ports
MODEM 1200
AGWPORT 8000
KISSPORT 8001

# Push-to-talk control (RTS pin on USB serial adapter)
PTT /dev/ttyUSB0 RTS
```

Adjust `ADEVICE`, `MYCALL`, and `PTT` to match your setup. If you need dual-channel operation, add another `CHANNEL` block and set `ACHANNELS 2`.

## 7. Start Direwolf
From the directory containing `direwolf.conf` run:

=== "Bash"
```bash
direwolf -c direwolf.conf -p
```

The `-p` flag enables a waterfall display so you can verify audio levels.

## 8. Configure AX.25 Integration
If you want traditional AX.25 tools to talk to Direwolf’s virtual TNC:

1. Edit `/etc/ax25/axports` and add a port definition:
   ```
   1    VK4XSS-1    1200    512    1    Direwolf port
   ```
2. Attach the kernel AX.25 stack to Direwolf’s KISS socket:
   ```bash
   sudo kissattach -l "$(readlink /tmp/kisstnc)" 1
   sudo kissparms -c 1 -p 1
   ```
3. (Optional) Configure `ax25d`/`axspawn` if you plan to accept incoming connects.
4. Test outbound calls:
   ```bash
   axcall 1 OTHERCALL-1
   ```
