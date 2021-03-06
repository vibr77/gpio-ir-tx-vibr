# gpio-ir-tx-vibr
Linux Kernel module GPIO-IR with multi GPIO management.
The new gpio-ir-tx that replace the lirc_rpi is not managing multiple transmitters (via SET_TRANSMITTERS). this proposes version enable the multi gpio with tx_mask.

This has been successfully testing on
Rasperry 3B+ & 4B

Note that I am struggling with Raspberry pi zero, where the original and this new version are not working (pulse issue)

## How to make it works

### Step 1 - first update to the last linux kernel version
```
apt-get update
apt-get upgrade
rpi-update
```

### Step 2 - Get the Linux  Kernel source
```
cd /home/pi
sudo apt-get install git bc bison flex libssl-dev
sudo wget https://raw.githubusercontent.com/notro/rpi-source/master/rpi-source -O /usr/local/bin/rpi-source && sudo chmod +x /usr/local/bin/rpi-source && /usr/local/bin/rpi-source -q --tag-update
rpi-source
```

### Step 3 - Compile the New driver version
```
cd gpio-ir-tx
make
```

### Step 4 - Add the overlay in boot file to enable driver at startup
```
sudo vi /boot/config
```
and add at the end this line
```
dtoverlay=gpio-ir-tx-vibr,gpio_pin=13
```
Note : gpio_pin=13, is not manage since gpio_pin is setup in the dts file 

### Step 5 - Configuration the gpio pin

Edit the DTS file, the section below is where to put your pin configuration,
```
vibr-gpios = 	<&gpio 21 0>,
		<&gpio 23 0>,
		<&gpio 24 0>,
		<&gpio 25 0>;
```              
First number is for the pin number, the second is for the state 0 => LOW => Output

Note: The MAX gpio is 8 but feel free to increase the Define MAX_GPIO in the source

Compile and install the DTS 
```
sudo  dtc -@ -I dts -O dtb -o gpio-ir-tx-vibr.dtbo gpio-ir-tx-vibr-overlay.dts
sudo cp gpio-ir-tx-vibr.dtbo /boot/overlays/
sudo reboot 
```

After reboot check the dts is loaded
```
sudo vcdbg log msg
```
You should see the log regarding gpio-ir-tx-vibr



### Step 6 - Now load the new kernel driver in the kernel 
```
sudo rmmod gpio-ir-tx
sudo insmod gpio-ir-tx

tail -f /var/log/kern
```
You should see the message of load


then we your lirc configured

### Step 7 - Test lirc

Assuming that your lircd is configured correctly, 
Please not that in version 0.94c the default.c plugin does not enable to manage SET_TRANSMITTER
to enable it 

change the end of the default.c
with 
```
	case  LIRC_SET_TRANSMITTER_MASK:
		return default_ioctl( LIRC_SET_TRANSMITTER_MASK, arg);

	default:
		return DRV_ERR_NOT_IMPLEMENTED;

```

 
```
irsend SET_TRANSMITTERS 1 2 // to send to First transmitter and second
irsend SET_TRANSMITTERS 2 // for the second transmitter
```
Note there is a bug in the current version, when sending to multiple transmitters there is a lock issue and irsend is not giving back hand, will manage that later.

No issue with 1 transmitter at a time

Enjoy the new module

### Step 8 - Optionnal Make the new module permanent after Reboot

```
uname -r
sudo cp gpio-ir-tx/gpio-ir-tx.ko /lib/modules/[KERNEL Version]/kernel/drivers/media/rc/
sudo update-initramfs -u
```



