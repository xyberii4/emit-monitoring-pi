﻿# Emit Solar Monitoring Pi

By directly accessing customers’ inverters, it removes the need to rely on third-party software and also provides greater insight and analysis into the solar PV system. Not only is this solution cost-effective, it is relatively easy to implement, requiring simple hardware and minimal knowledge.

# Hardware
- **Raspberry Pi 3A+**

- **WaveShare RS485 Can Hat**

   This allows the Pi to access the inverter via the RS485 port

- **Miscellaneous**

   - 5V/2.5A micro-USB power supply

      A 5V/2.5A power supply is important, as a weaker supply may cause the Pi to underperform or not turn on at all, while a stronger supply may cause heating issues that will reduce its lifespan and damage the Pi in the long run.

   - TP-Link TL-WN727N Wi-Fi antenna (optional)


      If the Pi is unable to connect to the client's Wi-Fi due to location or the material of the combiner box, a Wi-Fi antenna may help with this.
   - Micro-SD card
   - Case with DIN rail attachment


# Installation
## Manual installation
Required software:

- PuTTy (https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
- Raspberry Pi Imager (https://www.raspberrypi.com/software/)
- Nmap (https://nmap.org/download#windows)
---

### OS Setup
Attach the can hat to the Pi’s pins as shown below.\
<img src="assets/hardware.jpg" alt="drawing" width="200"/>

Insert the MicroSD car into your laptop and laucnh the Raspberry Pi Imager.
Select Raspberry Pi 3 for the model and Raspberry Pi OS Lite, which can be found by going to `Choose OS -> Raspberry Pi OS (Other) -> Raspberry Pi OS Lite (64-bit)` and select the microSD card's model for Storage.\
<img src="assets/rpi.png" alt="drawing" width="400"/>

Select `Next -> Edit Settings` and configure the settings as shown below.\
<img src="assets/rpi_settings.png" alt="drawing" width="400"/>\
>**NOTE**: Change the hostname from "tutorial" to the customer's name, and ensure it is unique. Wi-Fi is the office router's SSID and password. Remember the username and password as it will be used to login into the Pi later on.

Enable SSH with password authentication.\
<img src="assets/rpi_ssh.png" alt="drawing" width="400"/>

Apply the changes and allow the program to write to the SD card. Once complete, insert the SD card into its designated slot under the Pi and boot it up with the micro-USB adapter. Give it a few minutes to setup.

---
### Connecting to the Pi

To connect to the Pi via SSH, you will need the Pi's IP address. \
First launch Command Prompt and enter this command:
```
ipconfig
```
Under the `Wireless LAN adapter Wi-Fi` section, copy down the first 3 numbers of the IPv4 address, which is the network ID.
> **EXAMPLE:** If the IPv4 address is 10.69.0.58, the network ID is 10.69.0

Next, enter this command:
```
nmap -sn <NETWORK_ID>.1/24
```
This will scan for all devices connected to the same network. It should output a list of devices and their address. Find the entry with the name "Raspberry Pi" and note down its address.

**EXAMPLE**
```
$ nmap -sn 10.69.0.1/24
Nmap scan report for 10.69.0.58
Host is up (0.066s latency).
MAC Address: AC:67:5D:B1:36:18 (Intel Corporate)
Nmap scan report for 10.69.0.59
Host is up (0.0056s latency).
MAC Address: 6C:24:08:AF:85:EF (LCFC(HeFei) Electronics Technology)
Nmap scan report for 10.69.0.61
Host is up (0.13s latency).
MAC Address: 2C:CF:67:36:10:5F (Raspberry Pi (Trading))
```
The Pi's address is then 10.69.0.61.

Launch PuTTy and connect to this address. A pop-up should appear. Select Accept and continue.

A login prompt should appear. Enter the username and password you entered previously when installing the OS.

---
### RPI-Connect
If you have already configured `rpi-connect` on the Pi, it can also be remotely accessed from anywhere from https://connect.raspberrypi.com/devices.
> **NOTE** This software is still in development, and there will be times when it cannot be accessed.

---

### Configuring the Pi

Due to restrictions on the TIME network, you must first manually set the time and date.
```
sudo date -s "<DAY> <MONTH> <DATE> <HH>:<MM>:<SS> UTC <YYYY>"
```
>**NOTE:** The time must be in UTC

---

Update the system and enter the raspi-config screen.
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install git
sudo raspi-config
```
A blue screen with options should appear. \
Using your arrow keys, choose `System Options -> Boot/ Auto Login -> Console AutoLogin`.\
Next choose `Interface Options -> Serial Port -> No -> Yes`.\
Finally, choose `Interface Options -> RPi Connect -> Yes`. Follow any on-screen instructions.\
Choose `Finish -> Yes` to reboot. 

---
### Post-boot configuration
Once it has rebooted, reconnect to it and update the system.\
Setup rpi-connect by running `rpi-connect signin` and follow the displayed instrcuctions.\
Next, install git, then clone and navigate to this repository.
```
git clone https://github.com/xyberii4/emit-monitoring-pi.git
cd emit-monitoring-pi
```
Next, you must modify the telegraf configuration file to match the customer's details.\
There should be two files, starting with either `solis` or `growatt`. Open the correct file with this command:
```
nano <INVERTER>_telegraf.conf
```
Replacing \<INVERTER\> with the corresponding brand.

At the top of the file, under the `[global_tags]` section, there are two variables, `station` and `capacity`. Enter the customer's name in between the quotation marks for `station`, and replace any spaces with `_`. Do the same for capacity, in watts.

**EXAMPLE**
```toml
[global_tags]
station = "test_123"
capacity = "9860"
```
Save the file with `Ctrl-O` + `Enter` + `Ctrl-X`.

Next, execute the `setup.sh` file.
```
sudo ./setup.sh
```
This will setup the CAN HAT and install telegraf.
A prompt will also ask you if the customer's inverter is Huawei or Growatt.
Once complete, the Pi will reboot.

---

Reconnect to the Pi using the same steps above.\
Check telegraf is running successfully
```
systemctl status telegraf.service
```
If you see an `Error in plugin: could not open /dev/ttyS0: no such file or directory`, check the serial number of the CAN HAT.
```
$ ls -l /dev/serial*
lrwxrwxrwx 1 root root 5, 0 Sep  4 11:46 /dev/serial0 -> ttyAMA0
```
`ttyAMA0` is the new serial name. Open the telegraf configuration with
```
sudo nano /etc/telegraf/telegraf.conf
```
and modify `controller` line, replacing `ttyS0` with `ttyAMA0`
```toml
  controller = "file:///dev/ttyAMA0"
```
Restart telegraf
```
sudo systemctl restart telegraf
```
Proceed to [Wi-Fi Configuration](#wi-fi-configuration) to change the saved network to the customer's network. When installing on-site, it should then automatically connect to their network.

---

## Another installation method
Another way to install on a new Pi is by copying an existing SD card onto another.\
Insert the SD card to be cloned to into the USB port of the Pi and run this command.
```
sudo rpi-clone -s <HOST> sda
```
Replace `<HOST>` with the new hostname you want for the Pi. `sda` is usually the name of the inserted SD card, but you can check with `sudo fdisk -l` and look for the last entry.\
Once complete, it will tell you to press `Enter` to finish. Do so and remove your new SD card.\
Insert it into your new Pi, and connect to it through [SSH](#connecting-to-the-pi).
>**NOTE** You must be connected to the same network the original Pi is connected to.

Next, [configure your Pi again](#post-boot-configuration), following ALL steps.


## Connecting to inverter
The CAN HAT has 4 ports, labelled H, L, B and A. You will only need the A and B ports. \
A is the negative port, whilst B is the postive.\
For Growatt models, connect port 3 on the inverter to port A on the Pi, and port 4 and port B together.

# Wi-Fi Configuration
Changing the WiFi network requires the new connection to be added to the Pi before the customer changes their network.\
Connect to the Pi and access the Network Manger terminal interface.
```
sudo nmtui
```
Your terminal should now look like this.\
<img src="assets/nmtui_home.png" alt="drawing" width="400"/>

Choose `Edit a connection -> preconfigured`.\
Change the `SSID` and `Password` field with the new SSID and password then choose `Ok`.\
<img src="assets/nmtui_con.png" alt="drawing" width="400"/>\
Exit back to the terminal, and the next time the Pi boots up, it should connect to the updated network.
To connect to the Pi again, refer to [this section](#connecting-to-the-pi).
