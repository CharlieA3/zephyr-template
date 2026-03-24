Setup

1. Create a new github project using this template

2. Run this set of commands to set up wsl environment, instructions can also be found on Zephyr website

```
sudo apt update
sudo apt install python3
sudo apt install --no-install-recommends git cmake ninja-build gperf \
  ccache dfu-util device-tree-compiler wget python3-dev python3-venv python3-tk \
  xz-utils file make gcc gcc-multilib g++-multilib libsdl2-dev libmagic1
mkdir generate-firmware
cd generate-firmware
python3 -m venv .venv
source .venv/bin/activate
pip install west
cd ..
west init -m [YOUR PROJECT'S SSH GITHUB LINK] generate-firmware/
cd generate-firmware
west update
west zephyr-export
west packages pip --install
cd zephyr
west sdk install # this step will take a while to download and install
cd ..
west blobs fetch hal_espressif
```
Note: to install wsl, open powershell as admin and run " wsl --install "




Port Forwarding

Note: You will need to forward access of ports to wsl if you want to flash and use a serial monitor in wsl. This is because windows maintain permissions over the ports until you change them.

Extra note: for this to work, udev rules will need to be modified once permission is forwarded

1. Run this set of commands for port forwarding
```
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
```
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

