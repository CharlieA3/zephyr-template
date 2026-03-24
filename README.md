# Setup

1. Create a new github project using this template

2. Run this set of commands to set up wsl environment, instructions can also be found on Zephyr website: https://docs.zephyrproject.org/latest/develop/getting_started/index.html

```c
sudo apt update
sudo apt install python3
sudo apt install --no-install-recommends git cmake ninja-build gperf \
  ccache dfu-util device-tree-compiler wget python3-dev python3-venv python3-tk \
  xz-utils file make gcc gcc-multilib g++-multilib libsdl2-dev libmagic1
mkdir project-name
cd project-name
python3 -m venv .venv
source .venv/bin/activate
pip install west
cd ..
west init -m [YOUR PROJECT'S SSH GITHUB LINK] project-name/
cd project-name
// some of these install steps could take a while, maybe up to 30 minutes
west update
west zephyr-export
west packages pip --install
cd zephyr
west sdk install
cd ..
// this is specific to espressif SoCs, to find the corct one for your hardware, go to <project-name>/zephyr/west.yml and ctrl+f your vendor or whatever else you are looking to install
west blobs fetch hal_espressif
```
Note: to install wsl, open powershell as admin and run " wsl --install "




# Port Forwarding

Note: You will need to forward access of ports to wsl if you want to flash and use a serial monitor in wsl. This is because windows maintain permissions over the ports until you change them.

Extra note: for this to work, udev rules will need to be modified once permission is forwarded


1. Run this set of commands for port forwarding
```c
// In powershell: 
	winget install --interactive --exact dorssel.usbipd-win

// In WSL2 bash:
	sudo apt update
	sudo apt install linux-tools-generic hwdata
	sudo update-alternatives --install /usr/local/bin/usbip usbip /usr/lib/linux-tools/*-generic/usbip 20
	
// In powershell as admin:
	usbipd list (this will list out the different busids that you could forward)
	usbipd bind --busid <BUSID> (you only need to do this once)
	usbipd attach --wsl --busid <BUSID> (this gives the access of the port to WSL, and you cannot use it in windows until you detach access)
	
// In WSL2 bash:
	lsusb (to verify that something has been added)
	
// Flash or do whatever you need with this port (after makeing udev rule changes.

// In powershell as admin:
	usbipd wsl detach --busid <BUSID> (You should run this every time you are done or unplug to give access back to windows)
```
2. Run this set of commands for modifying udev rules
```c
// (go to this file in wsl)
sudo nano /etc/udev/rules.d/49-stlinkv2.rules

// (and add this line)
SUBSYSTEM=="usb", ATTR{idVendor}=="0483", ATTR{idProduct}=="374b", MODE="0666"

// (replace idVendor and idProduct values) -> Note: to find the correct value run " usbipd list " in powershell, or " lsusb " in wsl, you will see differnet values listed for each hardware device under VID:PID or next to ID
1d6b:0003

// (reload udev rules and replug the board)
sudo udevadm control --reload-rules
sudo udevadm trigger
```
3. Now you should be ready to flash


# Navigating the devicetree

Note: The Devicetree HOWTOs is a more formal collection of tips: https://docs.zephyrproject.org/latest/build/dts/howtos.html

1. Build for your board - this will put everything into your build directory, including the zephyr.dts file
2. Go in build/zephyr/zephyr.dts to see the entire compiled device tree
    a. this includes everything defined from the architecture to the SoC, to the board, then eventually your overlay file
    b. When you are using a dev board you only need to define application specific devicetree nodes, but when we have our custom PCBs you will need to define the board level device tree as well
3. Find the node you want to use in the zephyr.dts file
	a. scrolling through the dts, you will see a lot of listing that look like flash0: flash@0000 {}, where in this case flash0 is the node label, and flash@0000 is the node name with address
4. Grab the node you want (use #define in the source code)
```c
// in a .c source file
#define LED2 DT_ALIAS(led0)
```
Note: there are many ways to access a node, I've just happened to use node label mostly:
```c
/* Option 1: by node label */
#define MY_SERIAL DT_NODELABEL(serial0)

/* Option 2: by alias */
#define MY_SERIAL DT_ALIAS(my_serial)

/* Option 3: by chosen node */
#define MY_SERIAL DT_CHOSEN(zephyr_console)

/* Option 4: by path */
#define MY_SERIAL DT_PATH(soc, serial_40002000)
```

5. Access the specific property you want to modify (‘gpios’ property is an example but it could be others)
```c
static const struct gpio_dt_spec led_2 = GPIO_DT_SPEC_GET(LED2, gpios);
```

6. You are free to set the extracted property in your code like I did with this LED
```c
int state = 0;
ret = gpio_pin_set_dt(&led_2, state);
```

Note: look at this for understanding one node definition on the devicetree:
```c
# 'leds' before the colon is the node label, and 'leds' after the colon is node name
leds: leds {
	compatible = "gpio-leds"; /* in zephyr/boards/st/nucleo_l476rg/nucleo_l476rg.dts:26 */

	/* node '/leds/led_2' defined in zephyr/boards/st/nucleo_l476rg/nucleo_l476rg.dts:28 */
	# 'green_led_2' before the colon is the node label, and 'led_2' after the colon is node name
	green_led_2: led_2 {
		gpios = < &gpioa 0x5 0x0 >; /* in zephyr/boards/st/nucleo_l476rg/nucleo_l476rg.dts:29 */
		label = "User LD2";         /* in zephyr/boards/st/nucleo_l476rg/nucleo_l476rg.dts:30 */
	};
};
```

