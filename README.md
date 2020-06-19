# DA-681C system package

## base-system
Base system for DA-681C.
It contains the necessary tools and configs for setting up DA-681C

---
## Build the necessary drivers
### Build moxa-it87-gpio-driver
```bash=
git clone git@gitlab.syssw.moxa.com:MXcore-Package/moxa-it87-gpio-driver.git
cd moxa-it87-gpio-driver
make KRELEASE=$(uname -r) modules
make install
modprobe gpio-it87
```

### Build moxa-it87-serial-driver
```bash=
git clone git@gitlab.syssw.moxa.com:MXcore-Package/moxa-it87-serial-driver.git
cd moxa-it87-serial-driver
make KRELEASE=$(uname -r) modules
make install
modprobe it87_serial
```

### Build moxa-gpio-pca953x-driver
```bash=
git clone git@gitlab.syssw.moxa.com:MXcore-Package/moxa-gpio-pca953x-driver.git
cd moxa-gpio-pca953x-driver
make KRELEASE=$(uname -r) modules
make install
modprobe gpio-pca953x
```

### Build moxa-it87-gpio-led-driver
```bash=
git clone git@gitlab.syssw.moxa.com:MXcore-Package/moxa-it87-gpio-led-driver.git
cd moxa-it87-gpio-led-driver
make KRELEASE=$(uname -r) modules
make install
modprobe leds-it87-moxa-ngs
```

### Build moxa-hid-ft260-driver
```bash=
git clone git@gitlab.syssw.moxa.com:MXcore-Package/moxa-hid-ft260-driver.git
cd moxa-hid-ft260-driver
make KRELEASE=$(uname -r) modules
make install
modprobe hid-ft260
```

---

## Export sys class gpio function
```bash=
init_gpio() {
	local gpio=${1}
	local direction=${2}
	local value=${3}
	local active_low=${4}

	if [ ! -e "/sys/class/gpio/gpio${gpio}" ]; then
		echo ${gpio} > "/sys/class/gpio/export"
	fi

	if [ "${direction}" == "out" ]; then
                echo ${direction} > "/sys/class/gpio/gpio${gpio}/direction"
                [ ! -z "${active_low}" ] && \
                        echo ${active_low} > "/sys/class/gpio/gpio${gpio}/active_low"
                [ ! -z "${value}" ] && \
                        echo ${value} > "/sys/class/gpio/gpio${gpio}/value"
	fi
}
```

## Batch export sys class gpio function
```bash=
export_batch_sysgpio() {
	local TARGET_GPIOCHIP=$1
	local GPIOCHIP_NAME=gpiochip
	local GPIO_FS_PATH=/sys/class/gpio
	local GPIO_EXPORT="export"

	if [ x"$2" == x"unexport" ]; then
		GPIO_EXPORT="unexport"
	fi

	# Export GPIOs
	ls $GPIO_FS_PATH | grep $GPIOCHIP_NAME | while read -r chip ; do
		GPIO_LABEL=$(cat $GPIO_FS_PATH/$chip/label)
		if [[ "$GPIO_LABEL" != *"$TARGET_GPIOCHIP"* ]]; then
			continue
		fi

		pinstart=$(echo $chip | sed s/$GPIOCHIP_NAME/\\n/g)
		count=$(cat $GPIO_FS_PATH/$chip/ngpio)
		for (( i=0; i<${count}; i++ )); do
			init_gpio $((${pinstart}+${i})) "out" "0"
		done
	done
}
```
---
## UART
### Bind USB-to-SMBUS driver ft260 and PCA9535 GPIO expander
```bash=
bind_ft260_driver(){
	for filename in /sys/bus/i2c/devices/i2c-*/name; do
		i2c_devname=$(cat ${filename})
		if [[ $i2c_devname == *"FT260"* ]]; then
			i2c_devpath=$(echo ${filename%/*})
			echo "pca9535 0x20" > ${i2c_devpath}/new_device
			echo "pca9535 0x21" > ${i2c_devpath}/new_device
			echo "pca9535 0x22" > ${i2c_devpath}/new_device
		fi
	done
}

export_batch_sysgpio "pca9535"
```

### GPIO table for UART ports
| Device Node | GPIO#0 | GPIO#1 | GPIO#2 | GPIO#3 |
| ----------  | ------ | ------ | ------ | ------ |
| /dev/ttyM0  |   432  |  433   |   434  |   435  |
| /dev/ttyM1  |   436  |  437   |   438  |   439  |
| /dev/ttyM2  |   440  |  441   |   442  |   443  |
| /dev/ttyM3  |   444  |  445   |   446  |   447  |
| /dev/ttyM4  |   416  |  417   |   418  |   419  |
| /dev/ttyM5  |   420  |  421   |   422  |   423  |
| /dev/ttyM6  |   424  |  425   |   426  |   427  |
| /dev/ttyM7  |   428  |  429   |   430  |   431  |
| /dev/ttyM8  |   400  |  401   |   402  |   403  |
| /dev/ttyM9  |   404  |  405   |   406  |   407  |
| /dev/ttyM10 |   408  |  409   |   410  |   411  |
| /dev/ttyM11 |   412  |  413   |   414  |   415  |

### Example for controlling UART mode
```bash=
# to switch UART port 0 (/dev/ttyM0) to RS232 mode
echo 1 > /sys/class/gpio/gpio432/value
echo 1 > /sys/class/gpio/gpio433/value
echo 0 > /sys/class/gpio/gpio434/value
echo 0 > /sys/class/gpio/gpio435/value

# to switch UART port 0 (/dev/ttyM0) to RS485-2W mode
echo 0 > /sys/class/gpio/gpio432/value
echo 0 > /sys/class/gpio/gpio433/value
echo 0 > /sys/class/gpio/gpio434/value
echo 1 > /sys/class/gpio/gpio435/value

# to switch UART port 0 (/dev/ttyM0) to RS422/RS485-4W mode
echo 0 > /sys/class/gpio/gpio432/value
echo 0 > /sys/class/gpio/gpio433/value
echo 1 > /sys/class/gpio/gpio434/value
echo 0 > /sys/class/gpio/gpio435/value
```

## Digital IO
### Initial Digital IO GPIO pins
```bash=
# setup DIO output pins (DO0~DO1)
init_gpio "467"
init_gpio "468"

# export DIO input pins (DI0~DI5)
init_gpio "506"
init_gpio "507"
init_gpio "508"
init_gpio "509"
init_gpio "470"
init_gpio "471"
```
### Example for controlling Digital IO
```bash=
# set output high for DO0
echo high > /sys/class/gpio/gpio467/direction
# set output low for DO1
echo low > /sys/class/gpio/gpio468/direction

# get input from DI0 to DI5
cat /sys/class/gpio/gpio506/value
cat /sys/class/gpio/gpio507/value
cat /sys/class/gpio/gpio508/value
cat /sys/class/gpio/gpio509/value
cat /sys/class/gpio/gpio470/value
cat /sys/class/gpio/gpio471/value
```

## Relay
### Initial Relay GPIO pins
```bash=
# setup relay output value
init_gpio "493"
```
### Example for controlling Relay
```bash=
# pull high on relay
echo high > /sys/class/gpio/gpio493/direction

# pull low on relay
echo low > /sys/class/gpio/gpio493/direction
```

## Programmable LED
LED sysfs interfaces are generated by leds-it87-moxa-ngs driver.

| LED index | SYS path |
| --------  | -------- |
| 1 | /sys/class/leds/DA681C:GREEN:PRG1 |
| 2 | /sys/class/leds/DA681C:GREEN:PRG2 |
| 3 | /sys/class/leds/DA681C:GREEN:PRG3 |
| 4 | /sys/class/leds/DA681C:GREEN:PRG4 |
| 5 | /sys/class/leds/DA681C:GREEN:PRG5 |
| 6 | /sys/class/leds/DA681C:GREEN:PRG6 |
| 7 | /sys/class/leds/DA681C:GREEN:PRG7 |
| 8 | /sys/class/leds/DA681C:GREEN:PRG8 |

### Example for controlling Programmable LED
```bash=
# turn on LED index 0
echo 1 > /sys/class/leds/DA681C\:GREEN\:PRG1/brightness
# turn off LED index 0
echo 0 > /sys/class/leds/DA681C\:GREEN\:PRG1/brightness
```
