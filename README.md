# Bli-PiKVM
Install Pi-KVM on BliKVM v4 Allwinner.

## Disable BliKVM services
```bash
ssh blikvm@BLIKVM_IP

sudo apt update
sudo apt upgrade
sudo systemctl disable --now kvmd-hid kvmd-janus.service kvmd-video.service kvmd-web.service
sudo reboot
```

**NOTE:** kvmd-main.service is for LCD and hardware monitoring. Just leave kvmd-main.service enabled.

## Install kvmd-armbian

```bash
ssh blikvm@BLIKVM_IP

sudo apt install -y git vim make python3-dev gcc
git clone https://github.com/srepac/kvmd-armbian.git
cd kvmd-armbian
sudo ./install.sh
sudo reboot

ssh blikvm@BLIKVM_IP

cd kvmd-armbian
sudo ./install.sh
```

Thanks to [kvmd-armbian](https://github.com/srepac/kvmd-armbian) by [@srepac](https://github.com/srepac).


## Prepare MSD Partition

### Create MSD Partition
Before applying the patch, you would need to resize your installation partition on the SD card using GParted and create an additional partition for the msd. Finally, you need to add a mount entry for the new partition to **/etc/fstab** where /dev/mmcblk0p3 matches the name of the new partition you created.
```
/dev/mmcblk0p3 /var/lib/kvmd/msd  ext4  nodev,nosuid,noexec,ro,errors=remount-ro,data=journal,X-kvmd.otgmsd-root=/var/lib/kvmd/msd,X-kvmd.otgmsd-user=kvmd  0 0
```

## Apply kvmd MSD Patch

1. Download the msd patch and apply it.

```bash
cd /usr/lib/python3/dist-packages/kvmd
sudo wget -q https://github.com/RainCat1998/Bli-PiKVM/raw/main/3.291msd.patch -O 3.291msd.patch
sudo patch -p1 < 3.291msd.patch
```

2. Remove the following entries from /etc/kvmd/override.yaml. 

```
    msd:
        type: disabled
```
3. Reboot & Enjoy

**NOTE:** This patch is only for kvmd 3.291.

The patch code is ported from [fruity-pikvm](https://github.com/jacobbar/fruity-pikvm) by [@jacobbar](https://github.com/jacobbar).

## Configure ATX Controller

1. Download the latest BliKVM Source and compile kvmd-atx
```bash
git clone https://github.com/ThomasVon2021/blikvm
cd blikvm/package/kvmd-atx
make
```
2. Copy the compiled atx binary into /usr/bin/ then add sudo premission 
```bash
sudo cp atx /usr/bin/
sudo nano /etc/sudoers.d/kvmd

kvmd ALL=(ALL) NOPASSWD: ALL
```

3. Create the following /etc/kvmd/override.d/atx.yaml file

```yaml
kvmd:
    gpio:
        drivers:
            power_short:
                type: cmd
                cmd: [/usr/bin/sudo, /usr/bin/atx, --v, v4, --c, power_on]
            power_long:
                type: cmd
                cmd: [/usr/bin/sudo, /usr/bin/atx, --v, v4, --c, power_off]
            reset_sw:
                type: cmd
                cmd: [/usr/bin/sudo, /usr/bin/atx, --v, v4, --c, power_reset]

        scheme:
            on-off-button:
                driver: power_short
                pin: 0
                mode: output
                switch: false
            force-off-button:
                driver: power_long
                pin: 0
                mode: output
                switch: false
            reset-button:
                driver: reset_sw
                pin: 0
                mode: output
                switch: false

        view:
            table:
                - []
                - ["#ATX on BliKVM hardware"]
                - []
                - ["on-off-button|On/Off", "force-off-button|Force Off", "reset-button|Reset"]
```

4. Reboot & Enjoy

Thanks to [@srepac](https://github.com/srepac).

## Configure ATX Controller (ALTERNATIVE)

## Summary

On **BliKVM v4 (H616 / Mango Pi MCore)**, GPIO lines used for ATX control/status are exported and held by `kvmd-main` controller/OLED service. When KVMD is configured to use the native **ATX GPIO plugin** (`kvmd.atx.type: gpio`), KVMD requests these lines via **libgpiod** and fails if sysfs already owns them, typically with:

* `OSError: [Errno 16] Device or resource busy`

To solve this, we add a **systemd pre-start hook** for `kvmd.service` that **unexports the ATX GPIO lines from sysfs** before KVMD starts. Optionally, the hook can stop `kvmd-main.service` if it keeps re-exporting the lines.

---

## Files / Configuration

### 1) Pre-start script: `/usr/local/sbin/kvmd-prestart-gpio.sh`

```bash
#!/bin/bash
set -euo pipefail

CHIP="/dev/gpiochip0"

PWR_SW=228
RST_SW=272
LED_PWR=234
LED_HDD=233

PINS=("$PWR_SW" "$RST_SW" "$LED_PWR" "$LED_HDD")

log() { logger -t kvmd-prestart "$*"; }

# If kvmd-main (BliKVM controller/OLED daemon) keeps exporting these GPIOs via sysfs,
# KVMD ATX (libgpiod) will fail with "Device or resource busy".
# Set STOP_KVMD_MAIN=1 in the systemd override if you want to stop it here.
if [[ "${STOP_KVMD_MAIN:-0}" == "1" ]]; then
  if systemctl is-active --quiet kvmd-main.service; then
    log "stopping kvmd-main.service to release sysfs GPIO owners"
    systemctl stop kvmd-main.service || true
    sleep 0.2
  fi
fi

# Unexport pins from sysfs if present
for n in "${PINS[@]}"; do
  if [[ -d "/sys/class/gpio/gpio${n}" ]]; then
    log "unexport gpio${n} from sysfs"
    echo "$n" > /sys/class/gpio/unexport || true
  fi
done

sleep 0.1

# Debug: show current consumers for these lines
if command -v gpioinfo >/dev/null 2>&1; then
  gpioinfo "$CHIP" | egrep -n "line[[:space:]]+(${PWR_SW}|${RST_SW}|${LED_HDD}|${LED_PWR}):" || true
fi

exit 0
```

Make it executable:

```bash
sudo chmod +x /usr/local/sbin/kvmd-prestart-gpio.sh
```

---

### 2) systemd drop-in override for `kvmd.service`

Create:
`/etc/systemd/system/kvmd.service.d/override.conf`

```ini
[Unit]
After=kvmd-main.service
Wants=kvmd-main.service

[Service]
# Optional: stop kvmd-main (vendor controller) before KVMD starts if it keeps re-exporting sysfs GPIOs.
# Remove this line or set to 0 if you need kvmd-main (OLED) to remain running.
# Environment=STOP_KVMD_MAIN=1

ExecStartPre=+/usr/local/sbin/kvmd-prestart-gpio.sh
Restart=always
RestartSec=2
```

Reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kvmd
```

---

### 3) KVMD configuration snippet (YAML)

```yaml
kvmd:
  hid:
    mouse_alt:
      device: /dev/kvmd-hid-mouse-alt  # allow relative mouse mode

  atx:
    type: gpio
    device: /dev/gpiochip0
    power_switch_pin: 228
    reset_switch_pin: 272
    power_led_pin: 234
    hdd_led_pin: 233

  streamer:
    forever: true
    cmd_append:
      - "--slowdown"  # so target doesn't have to reboot
    resolution:
      default: 1280x720
```
Remove /etc/kvmd/override.d/atx.yaml (if present)
---

## ALL CHANGES in one copy/paste


```bash
# Apply all changes in one shot:
# - Install kvmd pre-start GPIO cleanup script
# - Install systemd drop-in override for kvmd.service (ExecStartPre runs as root)
# - Backup and write /etc/kvmd/override.yaml (UNCHANGED)
# - Remove /etc/kvmd/override.d/atx.yaml (if present)
# - Reload systemd + restart kvmd

set -euo pipefail

echo "[1/6] Install pre-start script..."
sudo tee /usr/local/sbin/kvmd-prestart-gpio.sh >/dev/null <<'SH'
#!/bin/bash
set -euo pipefail

CHIP="/dev/gpiochip0"

PWR_SW=228
RST_SW=272
LED_PWR=234
LED_HDD=233

PINS=("$PWR_SW" "$RST_SW" "$LED_PWR" "$LED_HDD")

log() { logger -t kvmd-prestart "$*"; }

# If kvmd-main (vendor controller/OLED daemon) keeps exporting these GPIOs via sysfs,
# KVMD ATX (libgpiod) will fail with "Device or resource busy".
# Set STOP_KVMD_MAIN=1 in the systemd override if you want to stop it here.
if [[ "${STOP_KVMD_MAIN:-0}" == "1" ]]; then
  if systemctl is-active --quiet kvmd-main.service; then
    log "stopping kvmd-main.service to release sysfs GPIO owners"
    systemctl stop kvmd-main.service || true
    sleep 0.2
  fi
fi

# Unexport pins from sysfs if present
for n in "${PINS[@]}"; do
  if [[ -d "/sys/class/gpio/gpio${n}" ]]; then
    log "unexport gpio${n} from sysfs"
    echo "$n" > /sys/class/gpio/unexport || true
  fi
done

sleep 0.1

# Debug: show current consumers for these lines
if command -v gpioinfo >/dev/null 2>&1; then
  gpioinfo "$CHIP" | egrep -n "line[[:space:]]+(${PWR_SW}|${RST_SW}|${LED_HDD}|${LED_PWR}):" || true
fi

exit 0
SH

sudo chmod +x /usr/local/sbin/kvmd-prestart-gpio.sh

echo "[2/6] Install systemd drop-in override for kvmd.service..."
sudo mkdir -p /etc/systemd/system/kvmd.service.d
sudo tee /etc/systemd/system/kvmd.service.d/override.conf >/dev/null <<'EOF'
[Unit]
After=kvmd-main.service
Wants=kvmd-main.service

[Service]
# Optional: stop kvmd-main (vendor controller) before KVMD starts if it keeps re-exporting sysfs GPIOs.
# Uncomment the next line to enable:
# Environment=STOP_KVMD_MAIN=1

# Run ExecStartPre as root even if kvmd.service uses User=kvmd
ExecStartPre=+/usr/local/sbin/kvmd-prestart-gpio.sh
Restart=always
RestartSec=2
EOF

echo "[3/6] Remove /etc/kvmd/override.d/atx.yaml (if present)..."
if [[ -f /etc/kvmd/override.d/atx.yaml ]]; then
  sudo rm -f /etc/kvmd/override.d/atx.yaml
  echo "Removed /etc/kvmd/override.d/atx.yaml"
else
  echo "No /etc/kvmd/override.d/atx.yaml found"
fi

echo "[4/6] Backup and write /etc/kvmd/override.yaml..."
if [[ -f /etc/kvmd/override.yaml ]]; then
  ts="$(date +%Y%m%d-%H%M%S)"
  sudo cp -a /etc/kvmd/override.yaml "/etc/kvmd/override.yaml.bak.${ts}"
  echo "Backed up to /etc/kvmd/override.yaml.bak.${ts}"
fi

sudo tee /etc/kvmd/override.yaml >/dev/null <<'YAML'
kvmd:
  hid:
    mouse_alt:
      device: /dev/kvmd-hid-mouse-alt  # allow relative mouse mode

  atx:
    type: gpio
    device: /dev/gpiochip0
    power_switch_pin: 228
    reset_switch_pin: 272
    power_led_pin: 234
    hdd_led_pin: 233

  streamer:
    forever: true
    cmd_append:
      - "--slowdown"  # so target doesn't have to reboot
    resolution:
      default: 1280x720
YAML

echo "[5/6] Reload systemd and restart kvmd..."
sudo systemctl daemon-reload
sudo systemctl restart kvmd

echo "[6/6] Show status, drop-ins, and GPIO ownership..."
sudo systemctl show kvmd -p DropInPaths --no-pager || true
sudo systemctl status kvmd --no-pager -l || true
gpioinfo /dev/gpiochip0 | egrep -n 'line[[:space:]]+(228|272|233|234):' || true

echo "Done."
```

