# Deploying Atlas on Android

| Device                      	| CPU                                                  	| RAM  	| STORAGE 	| OTHER 	|
|-----------------------------	|------------------------------------------------------	|------	|---------	|-------	|
| [Google Pixel 2](https://thepihut.com/products/raspberry-pi-zero-wh-starter-kit) 	| Snapdragon 835 (AARCH64) 4+4 CPU cores 	| 4G	| 64G     	|Bluetooth, NFC, USB-C, Wi-Fi|

## Installation
Android is basically a stripped down version of Linux. Luckily for us, atlas deployment on Android is quite easy.  
The only requirement is to download the required `cross-compiler` that will convert the native C code of atlas to native Android executable.  
**NOTE** the cross-compiler **DOES NOT** produce an apk.  
It simply generates binaries (similar to Linux) that may be executed on the terminal of Android. 

### Requirements - Steps
Official Guide to setup NDK (https://developer.android.com/ndk/guides/standalone_toolchain)
(you have to download an NDK, check step 1. below)

1. First download the required Android Native Development Kit from [here](https://developer.android.com/ndk/downloads) (in my case it was version revivision 21 [from here](https://github.com/android/ndk/wiki/Unsupported-Downloads)  
2. Extract the downloaded `zip`, in my case  
```bash
unzip android-ndk-r21e-linux-x86_64.zip
```
3. This will produce a folder called `android-ndk-r21e` in my case 
4. enter the folder by running:  
`cd android-ndk-r21e`
5. At this point, we will generate the Android toolchain using api 21 of Android and arm64 as target (since Pixel 2 has 64bit CPU)
`./build/tools/make_standalone_toolchain.py --arch arm64 --api 21 android-toolchain`
6. The generated folder `android-toolchain` contains the cross-compiler
7. by running `pwd` you get the current path, in my case:  
`/home/dkarnikis/android/arm64-21-toolchain/` copy that

### Building atlas
```sh
# follow the guide on ./README.md on how to fetch and clone the atlas components
# build the interpreter for Android
cd atlas-root/quickjs/src
# the path that contains the cross-compiler binaries is 
# THE_PATH_YOU_COPIED_BEFORE/bin/arch64-linux-android- in my case
# /home/dkarnikis/work/android/arm64-21-toolchain/bin/aarch64-linux-android-
make CROSS_PREFIX=/home/dkarnikis/work/android/arm64-21-toolchain/bin/aarch64-linux-android- ANDROID=1
# This will generate the qjs binary targeted for Android aarch64!
```

### Connecting to the Android device with WiFi
Official Guide [here](https://developer.android.com/studio/command-line/adb)
1. Find the IP of your Android device in your Lan (be sure to be connected to the same WiFi)
2. For Pixel 2:
```sh
Settings > Network & internet > Wi-Fi > The gear icon on the connected Wi-Fi > Advanced (on the bottom) > Under the IP address you may find you IP address (192.168.1.3 in my case)
```
3. Then we will connect to the android device via adb connect  
4. First of all, plug your device with usb to the computer
5. run `adb tcpip 5555`
6. Unplug your mobile device
7. run
`adb connect YOUR_IP:5555`, in my case  
`adb connect 192.168.1.3:5555`
8. `adb shell` to connect to the android shell
9. `cd /data/local/tmp` is the folder where atlas will be placed
10. The next step is to install the quickjs binaries to Android
11. To do so, go to another terminal, back to the root of the atlas project (let's call it atlas-root) and do:  
`adb push atlas-root /data/local/tmp`
12. Switch back to the android terminal, and now you may run you code (assuming that you have correctly configured atlas-addresses.txt) by doing:  
```sh
# enter the root of atlas
cd /data/local/tmp/atlas-root/atlas-client
../quickjs/src/qjs --file benchmarks/macro/crypto/streaming.js --threads 2 --servers 2 --verbose
# you should see some results popping in your screen!
```
