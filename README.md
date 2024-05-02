# Bli-PiKVM
Install Pi-KVM on BliKVM v4 Allwinner.

## Disable BliKVM services
```bash
ssh blikvm@BLIKVM_IP

sudo apt update
sudo apt upgrade
systemctl disable --now kvmd-hid kvmd-janus.service kvmd-video.service kvmd-web.service
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
wget -q https://github.com/RainCat1998/Bli-PiKVM/raw/main/3.291msd.patch -O 3.291msd.patch
patch -p1 < 3.291msd.patch
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
