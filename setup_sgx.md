# A Short SGX Tutorial

This short tutorial covers the installation of SGX for `atlas`'s deployment.

# Setting up the SGX environment
## Simulation Mode
(If you want to install with hardware support, skip this step)  
If you want to try atlas **without** using real SGX hardware you do not have to setup the whole SGX infrastructure
```sh
# goto SGX folder of this repo
cd SGX
# execute the binary that setups the SGX SDK
./sgx_linux_x64_sdk_2.14.100.2.bin
# when it asks if you want to install to current directory select `yes` (or choose the directory you want)
# finally a `source` command will appear that you need to execute, in my case:  
source /home/dkarnikis/SGX/sgxsdk/environment
# You are now set to run atlas on simulation mode!
```
## Hardware Mode
(If you have installed the simulated software, skip this step)  
**SUGGUESTED** Be sure to follow the official Intel guide and install  [Intel SGX SDK](https://github.com/intel/linux-sgx) and [Intel SGX Driver](https://github.com/intel/linux-sgx-driver)  
if you want to use SGX hardware.

# Running atlas-workers with SGX Hardware/Simulation
```sh
# source the your SGX environment, in my case
$ source /home/dkarnikis/SGX/sgxsdk/environment
# fetch the atlas worker
git clone git@github.com:atlas-runtime/atlas-worker.git
# enter the source code folder of the atlas-worker 
$ cd atlas-worker
# build the atlas worker enclave binary
# If you have Intel SGX enabled and running, you may use hardware mode `SGX_MODE=HW`
# If you want to try atlas without the underlying SGX capabilities you may use simulated mode `SGX_MODE=SIM`
$ make SGX_MODE=HW/SIM
# run the enclave binary listening to port 7000
./app -p 7000
# the next step would be to get the server IP (maybe using ifconfig/ip addr) and put in atlas-addresses.txt in the atlas-client
# for example, my server IP is 127.0.0.1 and the port my APP is listening is 7000
# so I have to put in atlas-addresses.txt the following:
# 7000 127.0.0.1
```
After this point, you may start offloading requests to the remote SGX worker (after you have properly edited the `atlas-addresses.txt` to contain the PORT IP of your server) from the client.
In order to run and test `atlas` follow [this](https://github.com/atlas-runtime/atlas-guides/blob/main/atlas_pi.md) guide 
after finishing SGX installation.
