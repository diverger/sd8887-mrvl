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

sudo insmod sd8xxx.ko cal_data_cfg=none drv_mode=1 sta_name=wlan
```

### 4. Verify

1. `dmesg`

2. `sudo nmcli d wifi rescan ifname wlan0`

3. `nmcli d wifi`

4. `iw phy`

5. `iw dev`

6. `ip a`

7. `sudo wifi-connect -i wlan0`

### 5. Driver unload

```bash
sudo ip link set wlan0 down

sudo rmmod sd8xxx mlan
```

### 6. Bluetooth

```bash
cd ~/sd8887-mrvl/mbt_src

make -j 4 KERNELDIR=/usr/src/linux-headers-$(uname -r) build

cd ../bin_sd8xxx/

sudo modprobe bluetooth

sudo insmod bt8xxx.ko
```

### 7. Verify

1. `sudo bluetoothctl`

### 8. Driver unload

```bash
sudo hciconfig hciX down

sudo rmmod bt8xxx
```

### 8. Notes

#### Building

I modified the Makefiles for the two drivers with minimal changes to make it run.
Only one source code change was needed, but this is natural for this driver (see
Unresolved symbols note below).

#### Firmware files

There are three firmware files included: sd8887_uapsta_a2.bin, sd8887_wlan_a2.bin
and sd8887_bt_a2.bin. The contents of the first equals the contents of the other
two, which are the WLAN and the Bluetooth parts of it. Those are used when parallel
firmware download is enabled or firmware reload is triggered. The driver operates
correctly with only sd8887_uapsta_a2.bin included when I tried it.

#### Calibration data (from Taiyo manual)

The calibration data needs to be stored in a configuration file or OTP. Although
originally driver requires the configuration file, in the case of this product, it
is not necessary to set the configuration file, as the calibration data is stored
in OTP.

When driver is installed to this product, please execute the following command.
Please set "none" for cal_data_cfg: `insmod sd8887.ko cal_data_cfg=none`.

#### Unresolved symbols

There are two unresolved symbols referenced in the WLAN sources:
`pxa3xx_enable_wifi_host_sleep_pins` and `pxa3xx_wifi_wakeup`. They are used by
the `GPIO_PORT_INIT`, `GPIO_PORT_TO_HIGH` and `GPIO_PORT_TO_LOW` macros. Those
macros are hardware platform dependent and their definition should match the
actual platform. For now I made their definitions empty and this functionality
is disabled. When the host CPU is in sleep mode, the module may wake it up upon
receipt of WLAN data. The module notifies the host with WL_HOST_WKUP. I am not
aware how this could be implemented with our board and how essential that is
for its operation.

#### STA interface name - wlan0

The sta_name parameter defaults to `mlan`. We are redefining it as `wlan`.

#### uAP interface

By default the driver will create a second interface `uap0`. This interface runs
in access point mode and can be useful for control type of applications. In the
future we may make WiFi Connect be able to run on two interfaces as well - one
for the captive portal and one for establishing Internet connectivity. I imagine
we can set defaults for Fin like this for easier setup.

For now I have disabled the AP interface by specifying `drv_mode=1`, but we may
consider leaving it running. We have users that use additional dongle or module
for similar purposes and by leaving the AP interface exposed, they can take
advantage of it. It could be an additional selling point for our board.

It is also possible to turn on the uAP interface during runtime using `echo
drv_mode=3 > /proc/mwlan/config`, but it errored out with a firmware init error
when I tried it.
