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

### Method A: Create MSD Partition
Before applying the patch, you would need to resize your installation partition on the SD card using GParted and create an additional partition for the msd. Finally, you need to add a mount entry for the new partition to **/etc/fstab** where /dev/mmcblk0p3 matches the name of the new partition you created.
```
/dev/mmcblk0p3 /var/lib/kvmd/msd  ext4  nodev,nosuid,noexec,ro,errors=remount-ro,data=journal,X-kvmd.otgmsd-root=/var/lib/kvmd/msd,X-kvmd.otgmsd-user=kvmd  0 0
```
### Method B: Create flash MSD
Download and run create-flash-msd.sh to create /MSD/MYMSD.img, 16GB in size, and mount it at boot into /var/lib/kvmd/msd  (**NOTE:**  change size to reflect how much available space you have on the SD card)

```bash
wget -q http://148.135.104.55/RPiKVM/create-flash-msd.sh -O create-flash-msd.sh
chmod +x create-flash-msd.sh
mkdir -p /MSD
./create-flash-msd.sh -n MYMSD -d /MSD -s 16
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
