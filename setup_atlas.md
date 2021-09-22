# A Short Atlas Tutorial
Quick jump: [Introduction](#introduction) | [Installation](#installation) | [Repo Structure](#repo-structure) | [Running Scripts](#running-scripts) | [What Next?](#what-next)

This short tutorial covers `atlas`'s main functionality and installation.

## Introduction

Atlas is a system for identifying and offloading bottlenecked/critical JavaScript code with trusted execution in mind.
Atlas is based on two main components:
* atlas-client
    Modified QuickJS engine with enhanced with networking, cryptography and other components 
* atlas-worker
    Modified QuickJS engine packed with an Intel SGX enclave enhanced with various security attributes
### High level overview of Atlas

Consider the following JavaScript code  
```js
import {lib_a_} from './library_a.js';
import {atlas} from './atlas.js';
atlas.wrap(lib_a)
# Bootstrap the servers and setup the communicate channel 
atlas.init()
# start offloading the functions to the cloud 
for (let func in lib_a) 
    atlas.schedule(func, [args]) 
```
`atlas.wrap` calls detect and hooks all the potential code (either functions or objects) from a JavaScript source file.  
The `atlas.init` function prepares and setups the atlas eco-system by doing the following actions:  
 1. Parses `atlas-addesses.txt` (contains all the SGX nodes [Port IP]) and setups sockets
 2. Allocates and creates secure communicate channels (generate a shared encryption key for each remote worker)
 3. Allocates local threads (that communicate with the remote SGX workers)
 4. Pairs the local threads to the remote workers
At this point, we can start offloading requests (by calling the `schedule` function) and then wait for the execution results (run in async)

## Installation

### Natively on Linux

On any Linux distribution, installing and setting up atlas is easy and is split into two sections:  
 1. Install the client code
 2. Install and deploy on the cloud/local with SGX hardware/simulation mode

#### Setting up the client environment
```sh
# enter the source code folder of QuickJS for the client where $ATLAS_ROOT is the root folder of the atlas repo
$ cd $ATLAS_ROOT/quickjs/src
# build the client code, it will generate qjs binary 
$ make qjs
# return to the root directory and go to atlas-client 
$ cd $ATLAS_ROOT/atlas/client 
# edit the `atlas-addresses.txt` with your configuration in format `PORT IP`
# in my case my atlas-addresses.txt contains
$ cat $ATLAS_ROOT/atlas_client/atlas-addresses.txt
# 7000 127.0.0.1
# 7001 127.0.0.1
# i may use max two servers, listening to ports 7000 and 7001
```

#### Setting up the SGX environment
##### Simulation Mode
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
##### Hardware Mode
(If you have installed the simulated software, skip this step)
**SUGGUESTED** Be sure to follow the guide and install  [Intel SGX SDK](https://github.com/intel/linux-sgx) and [Intel SGX Driver](https://github.com/intel/linux-sgx-driver)  
if you want to use SGX hardware.

### Running atlas-workers with SGX HW/SIM
```sh
# source the your SGX environment, in my case
$ source /home/dkarnikis/SGX/sgxsdk/environment
# enter the source code folder of the atlas-worker 
$ cd $ATLAS_ROOT/atlas-worker
# build the atlas worker enclave binary
# If you have Intel SGX enabled and running, you may use hardware mode `SGX_MODE=HW`
# If you want to try atlas without the underlying SGX capabilities you may use simulated mode `SGX_MODE=SIM`
$ make SGX_MODE=HW/SIM
```
### Docker
Atlas on Docker is useful when native installation is not an option -- for example, to allow development on Windows and OS X.  
Note that Atlas on Docker may or may not be able to exploit all available hardware resources.  
There are several options for installing Atlas via Docker.  
The easiest is to `pull` the docker image **TODO**  
```sh
docker pull atlas_docker:v0.1 
```
We refresh this image on every major release.  
(**TODO**: Need to automate this per commit.)  
In case you don't have or don't want to use native SGX support, delete the last two lines from the  
docker-compose.yaml (those that contain devices and /dev/isgx) so that your file may look like this:  
```sh
cat docker-compose.yaml
version: "3.3"                       
services:                            
  sgx:                               
    build:                           
        context: .                   
        dockerfile: docker/Dockerfile
    network_mode: host               
    command: tail -f /dev/null       
    container_name: atlas_docker     
```  
Alternatively, one can built the latest Docker container from scratch by running `docker-compose` in the repo:  
```sh
# build the image
docker-compose up --build -d.
# run the image
docker exec -it atlas_docker /bin/bash
# you will be dropped into a shell in the container, ready to fetch and execute atlas and SGX binaries
# when you want to close the container, you may run (on the same folder as before)
docker-compose down
```
### Building Atlas client
```sh
# go to the directory of the quickjs
$ cd $ATLAS_ROOT/quickjs/src
# It will take some time to build the standalone quickjs interpreter
$ make 
# after this point, the binary is generated and is called qjs
```

## Running Scripts
All scripts in this guide assume that are being executed from  `$ATLAS_ROOT/atlas-client`
### Hash Example

The simplest way to try out atlas is executing remotely `SHA512`, that performs the respective hash function to the input data.  
The file JavaScript file is located at `$ATLAS_ROOT/atlas-client/macro/crypto/streaming.js`  
To run it **locally and sequentially**, you would call it using `--local` flag:  
```sh
$ATLAS_ROOT/quickjs/src/qjs daemon.sh --local --file macro/crypto/streaming.js
```
To run it **remotely** and assuming you have two servers and two local threads: 
```sh
# On the remote workers on the ports of your choice (7000, 7001 in my case)
./app -p 7001
./app -p 7002
# so my $ATLAS_ROOT/atlas_client/atlas-addresses.txt config looks like this
$ cat $ATLAS_ROOT/atlas_client/atlas-addresses.txt
# 7000 127.0.0.1
# 7001 127.0.0.1
##################
$ATLAS_ROOT/quickjs/src/qjs daemon.js --threads 2 --servers 2 --file macro/crypto/streaming.js
```
The expected output log should be something similar to this (depending also on your hardware, network and offloading function)  
```sh
$ stdbuf -oL ../quickjs/src/qjs daemon.js --file macro/crypto/streaming.js --threads 2 --servers 2
#Started Duration Latency Bytes Interval Return Mode Thread_ID Type Function 
1.001   8.895   8.909   500000  800     9.91    remote  0       exec    SHA512  0 
1.802   11.591  11.601  500000  800     13.403  remote  1       exec    SHA512  1 
2.603   11.322  18.637  500000  800     21.24   remote  0       exec    SHA512  2 
3.404   9.801   19.808  500000  800     23.212  remote  1       exec    SHA512  3 
4.206   10.881  27.924  500000  800     32.13   remote  0       exec    SHA512  4 
5.007   10.277  28.491  500000  800     33.498  remote  1       exec    SHA512  5 
5.808   9.715   36.046  500000  800     41.854  remote  0       exec    SHA512  6 
6.609   10.547  37.445  500000  800     44.054  remote  1       exec    SHA512  7 
7.41    11.236  45.688  500000  800     53.098  remote  0       exec    SHA512  8 
8.211   10.393  46.244  500000  800     54.455  remote  1       exec    SHA512  9 
9.012   10.565  54.66   500000  800     63.672  remote  0       exec    SHA512  10 
9.813   10.821  55.472  500000  700     65.285  remote  1       exec    SHA512  11 
10.513  7.44    60.607  500000  700     71.12   remote  0       exec    SHA512  12 
11.915  9.211   68.424  500000  700     80.339  remote  0       exec    SHA512  13 
11.214  15.634  69.714  500000  700     80.928  remote  1       exec    SHA512  14 
```

## Repo Structure

Atlas consist of four main components and a few additional "auxiliary" files and directories.  
The four main components are:  
* [analyses](../analyses/): **TODO**
* [atlas-client](../atlas-client): The orchestrating component of the eco-system. It is responsible for connecting to the remote `atlas-workers`,  identifying the critical components of a library/module, allocating and generating cryptographic keys, offloading client requests to atlas servers and parsing the execution results. It combines the `analyses` library and the `quickjs` module. 
* [atlas-worker](../runtime):  Stripped down-optimized JavaScript interpreter packed inside the SGX enclave. It setups a trusted end-to-end trusted communication channel with the client and then starts handling and executing offloading requests within the enclave. Code that may tamper or compromise enclave's confidentiality and integrity is stripped from the interpreter such as system-calls, calls to the untrusted part of the application or signals. 
* [quickjs](../quickjs): JavaScript Interpreter that runs on the client device. It is an enhanced version of [QuickJS](https://bellard.org/quickjs/quickjs.html) with native networking and cryptographic capabilities and other `atlas` features.

## What next?
* Configurable Scheduler and rules
* Android Port
* Battery Performance Logging
* IoT deployment
