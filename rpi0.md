# Raspberry Zero Deployment
## Hardware
| Device                      	| CPU                                                  	| RAM  	| STORAGE 	| OTHER 	|
|-----------------------------	|------------------------------------------------------	|------	|---------	|-------	|
| [Raspberry Pi Zero W Rev 1.1](https://thepihut.com/products/raspberry-pi-zero-wh-starter-kit) 	| ARMv6-compatible processor rev 7 (v6l), 1Ghz, 1 Core 	| 512M 	| 16G     	|Bluetooth/WIFI|


## Operating System
Atlas should be capable of running to any available Linux distribution.  
For this tutorial, we will be installing [Raspbian OS](https://www.raspbian.org/) but this guide may also be applied to the install different Linux distros/environments. Additionally, this guide uses Ubuntu 20.10 as host machine.

## Installation
We support two different ways of setting up Raspbery PI.
1.  Vanilla raspbian installation (Atlas components will be built later) 
2.  Custom raspbian image packed with Atlas and its dependencies **TODO**

### Vanilla Installation
#### Requirements
1. Download the [latest](https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-05-28/2021-05-07-raspios-buster-armhf-lite.zip) version of Raspberry Pi OS Lite (https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit)
2. Download [balena etcher](https://www.balena.io/etcher/) for your Operating System. This tool will help us installing Raspbian to the sd card.
3. In my case, both files are downloaded to `~/Downloads` or `/home/your_username/Downloads`
4. Unzip `etcher` by doing:
    `cd ~/Downloads` (you may change to the path you downloaded your file)
    `unzip balena-etcher-electron-YOUR_VERSION.zip` (to extract the executable program)
5. At this point, we are ready to start flashing Raspbian OS

#### Flashing Vanilla OS
1. Insert the micro sd card to your computer (or use the provided micro sd card adapter if needed)
2. Go to `etcher` location and execute it with sudo privileges by doing:
`sudo ./balenaEtcher-YOUR_VERSION.AppImage` 
(We need sudo since we will alter the micro sd card which is a device. Enter your password if needed)
3. A new window will pop up and might look like this:  
![](imgs/etcher.png?raw=true)  
4. Click `Flash from file` to start installing the OS to the sd card
5. A new window will pop up and should look like this:  
![](imgs/etcher_os.png?raw=true)  
6. Select the downloaded [Raspian image at step 1.](#Requirements) in my case, `2021-05-07-raspios-buster-armhf-lite.zip`
7. On the main screen click `Change` if the selected device is not your sd card (the provided sd card is 16GB, so you should see something similar on your program)  
![](imgs/etcher.png?raw=true)  
8. Now you may click `Flash!` to start installing the Operating System (It might take some take to complete).
Your screen should now look like this:  
![](imgs/etcher_flash.png?raw=true)  
9. Finally, when it completes, your screen should look like this  
![](imgs/etcher_complete.png?raw=true)  
10. At this point, you are ready to eject the micro sd card from your computer, and insert it to the Raspberry PI!

#### Raspberry PI Overview
In order to properly config your system you will need to:
1. Insert the micro sd card to the Raspberry PI
2. Connect the Raspberry to a screen (a small HDMI converter is provided (a white dongle) that will be connected with a HDMI cable (not provided))
3. Connect a keyboard to it (a converter is provided in the bundle (a black cable), that should be connected with a keyboard (not provided))
4. Connect the Raspberry to the charger (It is provided)
5. When all the components are connected, my setup looks like this (make sure to connect the charger on the port that says **PWR**):
![](imgs/rpi.png?raw=true)  

The green light means your device is on!

#### OS Configuration
1. Switch to your HDMI screen now. After some time (when no new text is being printed) your screen should look like this:
![](imgs/screen.jpg?raw=true)  
2. The default credentials for Raspbian OS are:
Username: `pi`  
Password: `raspberry`  
(**for other distros the credentials might be different**)
Type in `pi` and press enter. Then when it asks for password type `raspberry` and press enter.  
3. After succesfully logging in, you are using a fully fledged Linux environment with `bash`!  
4. The last step here is to setup the Wifi. To do so, type  
`sudo raspi-config`.  
A new screen should pop up that looks like this:  
![](imgs/raspi.jpg?raw=true)  
5. Select the Option: `1 System Options` and press enter.  
Then select `S1 Wireless LAN` and press enter.  
A new window will pop up with countries. `Do select your country` and press enter.  
Then the SSID of your WiFI will be asked. `Insert the correct name` and press enter.  
![](imgs/ssid.png?raw=true)  
Then the password for the WiFi will be asked. `Insert the correct password` and press enter.  
![](imgs/pass.jpg?raw=true)  
6. Then select `<Finish>` on the main screen (of step 4) and you should be be back to the shell!
7. To verify you are connected to the network, try running `ifconfig` and find the entry containing `wlan0`:  
![](imgs/wlan0.png?raw=true)  
In my case, my IP is `192.168.1.11`. 
If you cannot find such an entry, you might have miss-configured your SSID/Password on previous steps.

At this point, you whole environment is set! However, if you want to deploy an SSH server and connect remotely your Raspberry PI, do the following:  
1. Enable the ssh service so it runs every time you reboot your Raspberry PI.  
`sudo systemctl enable ssh.service`  
2. Restart the ssh service to use it now.  
`sudo systemctl restart ssh.service`  
If it asks for your Username/Password, enter:  
Username: `pi`  
Password: `raspberry`  
The operations should complete without a fail.  
Now you may connect to the remote pi from your host machine by connecting to it.  
3. To do so run on your computer (which must be connected on the same router):  
`ssh pi@IP_OF_THE_RASPBERRY` in my case  
`ssh pi@192.168.1.11`  
4. The first time you connect, you should see something like this.  
![](imgs/ssh.png?raw=true)  
5. Type `yes` and then write `raspberry` when it asks for the password.  
6. Finally you will be connected to the Raspberry PI remotely, running on bash!  
7. You may now even disconnect the HDMI screen and the usb keyboard from the Raspberry, since you may do everything remotely (you only need the power cable)!  


#### Additional Settings
Due to the limited available RPI0 memory (~430MB), running memory-intensive applications might be an issue. To overcome this problem, we need to increase the max available swap memory (currently is set to 100MB).  
To do so (on raspbian):  
```sh
# disable the swap
sudo dphys-swapfile swapoff
# modify the /etc/dphys-swapfile as root and edit the CONF_SWAPSIZE to 800MB --- you can change it
CONF_SWAPSIZE=800
# restart the swap file
/etc/init.d/dphys-swapfile restart
```
