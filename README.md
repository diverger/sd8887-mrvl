Raspbian build instructions 
---------------------------

### 1. Preparation

1. Install latest Raspbian Stretch Lite

2. `sudo apt-get update && sudo apt-get upgrade`

3. `sudo apt-get install raspberrypi-kernel-headers build-essential bc git`

4. Setup your ~/.ssh config

5. `git clone -b raspbian-build git@github.com:resin-io/sd8887-mrvl.git`

6. `sudo cp sd8887-mrvl/firmware/15.68.7.p154/* /lib/firmware/mrvl/`

7. Create `/etc/modprobe.d/blacklist-mrvl.conf`

```
blacklist mwifiex
blacklist mwifiex_sdio
blacklist btmrvl
blacklist btmrvl_sdio
```

8. `sudo reboot now`

### 2. NetworkManager and WiFi Connect

The WiFi Connect install script will enable NetworkManager and install wifi-connect:

`bash <(curl -L https://github.com/resin-io/resin-wifi-connect/raw/master/scripts/raspbian-install.sh)`

### 3. WLAN

```bash
cd ~/sd8887-mrvl/wlan_src

make -j 4 KERNELDIR=/usr/src/linux-headers-$(uname -r) build

cd ../bin_sd8xxx/

sudo modprobe cfg80211

sudo insmod mlan.ko

sudo insmod sd8xxx.ko cal_data_cfg=none drv_mode=1
```

### 4. Verify

1. `dmesg`

2. `sudo nmcli d wifi rescan ifname mlan0`

3. `nmcli d wifi`

4. `iw phy`

5. `iw dev`

6. `ip a`

7. `sudo wifi-connect -i mlan0`

### 5. Driver unload

```bash
sudo ip link set mlan0 down

sudo rmmod sd8xxx mlan
```

