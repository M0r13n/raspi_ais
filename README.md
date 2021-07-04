# Raspberry PI based AIS receiver

## Hardware

- Raspberry PI 4 2GB
- RTL Receiver 

## Setup a (headless) Raspi

- flash Raspian (with or without gui) to an SD card
- place file called `ssh` under `/boot/` to enable SSH access on first boot
- insert the SD card into the raspi
- connect raspi to your local network (LAN)
- power it on

## Prepare RTL receiver

- Connect to Pi: `ssh pi@192.168.0.42` (default password is `raspberry`)
- Install required Software: `sudo apt-get -fym install git cmake libusb-1.0-0-dev build-essential`
- Plugin the RTL receiver and make sure, that is detected:
```bash
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 0bda:2838 Realtek Semiconductor Corp. RTL2838 DVB-T  # <-- This is the receiver
Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
- prevent the Kernel from loading it as a DVB-T device (we don't want to watch TV):
    * `nano /etc/modprobe.d/rtlsdr.conf`
    ```bash
    # Do NOT load the DVB device driver -> RTL-SDR devices will show up as SDR receivers
    blacklist dvb_usb_rtl28xxu
    ```
- Reboot : `sudo reboot` and connect again
- If everything went well, the command `lsmod | egrep 'sdr|dvb'` should not return any output
- Test that the receiver works:

```bash
$ sudo apt install rtl_test -y
$ sudo rtl_test
Found 1 device(s):
  0:  Realtek, RTL2838UHIDIR, SN: 00000001

Using device 0: Generic RTL2832U OEM
Found Rafael Micro R820T tuner
Supported gain values (29): 0.0 0.9 1.4 2.7 3.7 7.7 8.7 12.5 14.4 15.7 16.6 19.7 20.7 22.9 25.4 28.0 29.7 32.8 33.8 36.4 37.2 38.6 40.2 42.1 43.4 43.9 44.5 48.0 49.6
[R82XX] PLL not locked!
Sampling at 2048000 S/s.

Info: This tool will continuously read from the device, and report if
samples get lost. If you observe no further output, everything is fine.

Reading samples in async mode...
Allocating 15 zero-copy buffers
lost at least 88 bytes
...
```

## Setup and install `rtl-sdr`

```bash
$ git clone git://git.osmocom.org/rtl-sdr.git && cd rtl-sdr
$ mkdir build && cd build
$ cmake ../
$ sudo make install
$ sudo ldconfig
```

## Setup and install `dump1090` (if you want to monitor airplane traffic)
- make sure that you have built and installed `rtl-sdr`. Otherwise you won't be able to compile `dump1090`
- `git://github.com/MalcolmRobb/dump1090.git && cd dump1090`
- `make`
- start: `./dump1090`

## RTL-AIS Software
- `git clone https://github.com/dgiardini/rtl-ais.git`
- `cd rtl-ais`
- `make`


## Kalibrate
```bash
# Install deps
sudo apt install build-essential libtool automake autoconf librtlsdr-dev libfftw3-dev

# Clone repo
git clone https://github.com/steve-m/kalibrate-rtl

# Compile & install
cd kalibrate-rtl/
./bootstrap && CXXFLAGS='-W -Wall -O3' ./configure && make
./configure
make
sudo make install

# Run calibartion
rtl_test -p
```

you will get similar output to this:

```
... 
real sample rate: 2048047 current PPM: 23 cumulative PPM: 23
real sample rate: 2048200 current PPM: 98 cumulative PPM: 62
real sample rate: 2047865 current PPM: -66 cumulative PPM: 18
real sample rate: 2048203 current PPM: 100 cumulative PPM: 39
real sample rate: 2048173 current PPM: 85 cumulative PPM: 14
real sample rate: 2047910 current PPM: -44 cumulative PPM: 13
real sample rate: 2048030 current PPM: 15 cumulative PPM: 13
real sample rate: 2048038 current PPM: 19 cumulative PPM: 14
real sample rate: 2048045 current PPM: 22 cumulative PPM: 14
real sample rate: 2048031 current PPM: 16 cumulative PPM: 14
real sample rate: 2048039 current PPM: 19 cumulative PPM: 14
real sample rate: 2048036 current PPM: 18 cumulative PPM: 14
...
```

Give it some time to settle on a cumulative PPM. After a minute or so, you can stop the program with CTRL + C.
In the above case, I would choose 14 as cummulative error.


Scanning for GSM-900 base stations:

`kal -s 900 -g 49.6 -e 14`

will yield

```
kal -s 900 -g 49.6 -e 23pi@raspberrypi:~/rtl-ais $ kal -s 900 -g 49.6 -e 18
Found 1 device(s):
  0:  Generic RTL2832U OEM

Using device 0: Generic RTL2832U OEM
Found Rafael Micro R820T tuner
Exact sample rate is: 270833.002142 Hz
[R82XX] PLL not locked!
Setting gain: 49.6 dB
kal: Scanning for GSM-900 base stations.
GSM-900:
    chan:    6 (936.2MHz - 28.768kHz)    power:   87038.12
    chan:   45 (944.0MHz - 29.471kHz)    power:  150421.29
    chan:   47 (944.4MHz - 29.435kHz)    power:   45369.09
```

The higher the output power, the better the signal is. In my case there aren't many base stations.

Kalibrate on channel 45 and error rate 14.

`kal -c 45 -g 49.6 -e 14`

will yield

```
Found 1 device(s):
  0:  Generic RTL2832U OEM

Using device 0: Generic RTL2832U OEM
Found Rafael Micro R820T tuner
Exact sample rate is: 270833.002142 Hz
[R82XX] PLL not locked!
Setting gain: 49.6 dB
kal: Calculating clock frequency offset.
Using GSM-900 channel 45 (944.0MHz)
Tuned to 944.000000MHz (reported tuner error: 0Hz)
average  [min, max] (range, stddev)
- 29.699kHz     [-29714, -29678]   (36, 10.022093)
overruns: 0
not found: 1
average absolute error: 49.461 ppm <--- Your overall error, that can be used as input for rtl_ais (-p parameter)
```


## Setup Udev Rules and make the device available to non-root users
- by default only `root` can read from the SDR receiver
- to fix this we need to add some `udev rules` 
- `lsusb` reports something like `Bus 001 Device 003: ID 0bda:2838 Realtek Semiconductor Corp. RTL2838 DVB-T`
- `0bda` is the **VENDOR ID** and `2838` is the **PRODUCT ID**
- Create a new udev rules file: `sudo touch  /etc/udev/rules.d/rtl-sdr.rules`
- Add the following content
```bash
SUBSYSTEM=="usb", ATTRS{idVendor}=="{VENDOR_ID}", ATTRS{idProduct}=="PRODUCT_ID", GROUP="adm", MODE="0666", SYMLINK+="rtl_sdr"
```
- and replace VENDOR_ID and PRODUCT ID with the values from above, e.g.:
```bash
SUBSYSTEM=="usb", ATTRS{idVendor}=="0bda", ATTRS{idProduct}=="2838", GROUP="adm", MODE="0666", SYMLINK+="rtl_sdr"
```
- reboot, to make sure, that the changes are applied


## Resources
- https://www.instructables.com/rtl-sdr-on-Ubuntu/
- https://cromwell-intl.com/open-source/raspberry-pi/sdr-getting-started.html
- https://www.raspberry-pi-geek.de/ausgaben/rpg/2014/06/luftraum-ueberwachen-mit-dem-raspberry-pi/
- https://github.com/dgiardini/rtl-ais
- https://www.ronan.bzh/p/ais-receiver-on-a-raspberry-pi-with-rtl-sdr/