# A Short Atlas Tutorial
Quick jump: [Introduction](#introduction) | [Installation](#installation) | [Repo Structure](#structure) | [Running Scripts](#running-scripts) | [What Next?](#what-next)

This short tutorial covers the `atlas`'s main functionality.

## Introduction

Atlas is a system for identifying and offloading bottlenecked/critical Javascript code with trusted execution in mind.
Atlas is based on two main components:
* Atlas-client
    Modified QuickJS engine with enhanced with networking, cryptography and other components 
* Atlas-worker
    Modified QuickJS engine packed with an Intel SGX enclave enhanced with various security attributes
### High level overview of Atlas

Consider the following Javascript code  
```js
import lib_a as la
import atlas as atlas
atlas.wrap(la)
# Init the atlas execution 
atlas.init()
# start offloading the functions to the cloud 
foreach (let func in la) 
    atlas.schedule(func, [args]) 
```
The `wrap` calls detect and hooks all the potential code (either functions or objects) from a Javascript source file.  
The `init` function prepares and setup the `atlas` ecosystem:

 - Parse `atlas-addesses.txt` (contains all the SGX nodes [Port IP]) and setup sockets
 - Allocate and create secure connections (crypto key allocation and handshake) for all the servers
 - Allocate local workers (that communicate with the remote SGX workers)
 - Distribute and assign the remote SGX workers to the local workers
At this point, we can start offloading the functions (by calling the `schedule` function) and wait the results (async)

## Installation

### Natively on Linux

On any Linux distribution, installing and setting up `atlas` is easy and is split into two sections:

 1. Install the client code
 2. Install and deploy on the cloud/local with SGX hardware/simulation mode

#### Setting up the client environment
```sh
# enter the source code folder of QuickJS for the client 
$ cd $ATLAS_ROOT/quickjs/src
# build the client code, it will generate qjs binary 
$ make qjs; cp qjs ../
# return to the root directory and go to atlas-client 
$ cd $ATLAS_ROOT/atlas/client 
# edit the `atlas-addresses.txt` with your configuration in format `PORT IP`
```

#### Setting up the SGX environment
Be sure to install  [Intel SGX SDK](https://github.com/intel/linux-sgx) and [Intel SGX Driver](https://github.com/intel/linux-sgx-driver)
```sh
$ source the your SGX environment
source /opt/intel/sgxsdk/environment
# enter the source code folder of SGX worker 
$ cd $ATLAS_ROOT/sgx-worker
# build the atlas worker enclave binary
# If you have Intel SGX enabled and running, you may use hardware mode `SGX_MODE=HW`
# If you want to try atlas without the underlying SGX capabilities you may use simulated mode `SGX_MODE=SIM`
$ make SGX_MODE=HW/SIM
```
<!---
### Docker

**TODO SETUP OUR DOCKER**

Atlas on Docker is useful when native installation is not an option -- for example, to allow development on Windows and OS X.
Note that Atlas on Docker may or may not be able to exploit all available hardware resources.
There are several options for installing Atlas via Docker.
**TODO**
The easiest is to `pull` the docker image 
```sh
docker pull 
```
We refresh this image on every major release.

[//]: # (TODO: Need to automate this per commit.)

Alternatively, one can built the latest Docker container from scratch by running `docker build` in the repo:
```sh
docker build -t atlas-artifact .
```
This will build a fresh Docker image using the latest commit---recommended for development.


In all the above cases, launching the container is done via:
```sh
docker run --name atlas-docker -it atlas-artifact
```
-->

## Running Scripts

All scripts in this guide assume that are being executed from  `$ATLAS_ROOT/atlas-client`

### Hash Example

The simplest test to try out `atlas` is SHA512, that performs simple has calls to input data.  
The file is located at `$ATLAS_ROOT/atlas-client/macro/crypto/streaming.js`  
To run it **locally and sequentially**, you would call it using `--local` flag:  
```sh
$ATLAS_ROOT/quickjs/src/qjs daemon.sh --local --file macro/crypto/streaming.js
```
To run it **remotely** and assuming you have 2 servers and 2 local workers: 
```sh
# On the remote workers on the ports of your choice (7000, 7001 in my case)
./app -p 7001
./app -p 7002
# so my $ATLAS_ROOT/atlas_client/atlas-addresses.txt config looks like this
$ cat $ATLAS_ROOT/atlas_client/atlas-addresses.txt
# 7000 127.0.0.1
# 7001 127.0.0.1
##################
$ATLAS_ROOT/quickjs/src/qjs daemon.js --local --threads 2 --servers 2 --file macro/crypto/streaming.js
```


## Repo Structure

Atlas consist of four main components and a few additional "auxiliary" files and directories. 
The three main components are:

* [analyses](../analyses/): **TODO**

* [atlas-client](../atlas-client): The orchestrating component of the eco-system.  It is responsible for conneting to the servers,  identifying the critical functions of a library/module, allocating cryptographic keys, offloading client requests to atlas servers and parsing the results. Combines the `analyses` and the `quickjs` module. 

* [sgx-worker](../runtime):  Stripped down-optimized Javascript Interpreter packed inside the SGX enclave. Setups trusted connection with the client and  upon it request receival, it evaluates and processes them inside the enclave. Code that may tamper/leak the contets of the enclave is removed/unsupported such as system-calls, calls to untrusted part of the application or signals. 
* [quickjs](../quickjs): Javascript Interpreter that runs on the client device. Enhanced version of [QuickJS](https://bellard.org/quickjs/quickjs.html) with networking, cryptographic capabilities and other `atlas` functionality.

## What next?
* Configurable Scheduler
* Android Port
* BAttery Management