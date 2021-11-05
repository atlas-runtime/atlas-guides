# Deploying Atlas on Raspberry PI Zero

Quick jump: [Introduction](#introduction) | [Installation](#installation) | [Running](#running) | [Remote Offloading](#remote-offloading) | [How Atlas works](#how-atlas-works)

This guide focuses strictly on installing and deploying atlas on Raspberry PI without the need
of installing Intel SGX.

If you're interested in testing remote offloading, you can follow the instructions in the
[Remote Offloading](#remote-offloading) section below.

## Introduction

Two components are needed to run ATLAS:

* **atlas-client**
    The orchestrating component written in JavaScript that is responsible for offloading execution requests.
* **atlas-qjs**
    Modified [QuickJS](https://bellard.org/quickjs/) engine packed with
    cryptographic and networking capabilities. It also performs code
    transformations during parsing phase to provide secure guarantees about the
    code being executed.

### Important Note on Battery Measurements

We have found that every Raspberry PI battery has a different API to get usage
statistics.  In our evaluation, we are a using a Raspberry PI Zero Wireless
with headers, a UPS HAT: Waveshare and Batteries: 2 x 18650 Li-ion 3400mAh
3.7V.

If you are using a different battery module, ATLAS' battery functionality
will need to be modified to match the vendor's battery API. In case atlas does
not detect any battery module, the value -1 is returned to the battery status.

## Installation

We assume you already have a Raspberry PI ready to go. If you need help setting
this up, you can follow [this guide](https://github.com/atlas-runtime/atlas-guides/blob/main/rpi0.md).

Installing and setting up ATLAS is easy and is split into two sections:
 1. Install the client environment
 2. Install the atlas interpreter

### Setting up the client environment

```sh
# create the root folder for our project
$ mkdir ~/atlas_root && cd ~/atlas_root
$ echo "export ATLAS_ROOT=`pwd`" >> ~/.bashrc
$ source ~/.bashrc
```

Fetch the repo of atlas-client:
```sh
$ cd $ATLAS_ROOT
$ git clone git@github.com:atlas-runtime/atlas-client.git --depth 1
```

### Setting up atlas interpreter

```sh
$ cd $ATLAS_ROOT
# fetch the atlas interpreter
$ git clone git@github.com:atlas-runtime/atlas-qjs.git --depth 1 quickjs
# enter the source code folder of QuickJS for the client where $ATLAS_ROOT is the root folder of atlas
$ cd $ATLAS_ROOT/quickjs/src
# build the client code, it will generate qjs binary.
# This will take a while since the rpi 0 is so low
# power. You will see some compiler warnings, this is okay.
$ make qjs
```

## Running

All scripts in this guide assume that are being executed from `$ATLAS_ROOT/atlas-client`

**IMPORTANT**

As mentioned before, we have found that different battery modules have
different APIs for measuring battery consumption. Our testbed uses a Raspberry
PI Zero Wireless with headers, a UPS HAT: Waveshare and Batteries: 2 x 18650
Li-ion 3400mAh 3.7V. If you are using a different battery module, we will need
to augment ATLAS to support that vendor's battery API. In case atlas does not
detect any battery module, the value -1 is printed to the battery status column.

### Math Example

The simplest way to try out atlas is by using the `math` library which performs simple arithmetic operations.
The JavaScript file for this example is located at `$ATLAS_ROOT/atlas-client/benchmarks/math/math.js`
To run it **locally and sequentially**, pass the `--local` flag:
```sh
$ cd $ATLAS_ROOT/atlas-client
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --local --file benchmarks/math/run.js
```
The expected output is:
```sh
46
-5
Infinity
5
```

To view each request's information and metadata, you may use the `--log` flag:

```sh
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --local --file benchmarks/math/run.js --log log.dat
# view the file log
$ cat log.dat
######################################################################################################################
#Start  Battery:72.83333333333334
#Start  Duration                   Latency  Bytes  Interval  End    Mode   Thread_ID  Type  Function   Request_ID  Battery_Status
0.731   0                          0.731    7      -1        0.731  local  -1         exec  math.add   0           72.83333333333334
1.206   0                          1.206    5      -1        1.206  local  -1         exec  math.sub   0           72.83333333333334
1.683   0                          1.683    5      -1        1.683  local  -1         exec  math.div   0           72.83333333333334
2.159   0                          2.159    5      -1        2.159  local  -1         exec  math.mult  0           72.83333333333334
```

### Encrypt and Sign Test

For our second demo application, we will be using a program fragment that
performs a simple AES encrypt and HMAC Sign. Using atlas we should only provide
a data buffer (to be encrypted) but also a pair of cryptographic keys (one used
for the encryption and the second for the signing).

In this benchmark, we are using a function called `generate_traffic` that
dynamically generates and issues 120 `encrypt_sign` requests at different time
intervals. We hold a global array of promises to store every returned remote
request. After ~120 requests have been received, the program runs
`Promise.allSettled` on the global promise array to gather the execution
results and then terminates.

Our application is based on [crypto-es](https://github.com/entronad/crypto-es).

The source code for this JavaScript benchmark is located at
`$ATLAS_ROOT/atlas-client/benchmarks/crypto-benchmark/run.js`

and our custom crypto wrapper for encrypt and sign

`$ATLAS_ROOT/atlas-client/benchmarks/crypto_benchmark/crypto-wrapper.js`

To run it **locally and sequentially**, pass the `--local` flag:
```sh
# enter the atlas client
$ cd $ATLAS_ROOT/atlas-client/
# execute the code
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --local --file benchmarks/crypto_benchmark/run.js --log crypto_l.dat
```

This should take about 2-3 minutes.  The expected output should be something like this:

```sh
$ cat crypto_l.dat
#Start  Battery:72.33333333333333
#Start  Duration                   Latency  Bytes  Interval  End     Mode   Thread_ID  Type  Function                 Request_ID  Battery_Status
0.643   0.668                      1.311    10038  800       1.311   local  -1         exec  benchmarks.encrypt_sign  1           72.33333333333333
2.59    0.701                      3.291    10038  800       3.291   local  -1         exec  benchmarks.encrypt_sign  2           72.50000000000001
4.572   0.703                      5.275    10038  800       5.275   local  -1         exec  benchmarks.encrypt_sign  3           72.50000000000001
6.557   0.702                      7.259    10038  800       7.259   local  -1         exec  benchmarks.encrypt_sign  4           72.50000000000001
8.542   0.707                      9.249    10038  800       9.249   local  -1         exec  benchmarks.encrypt_sign  5           72.50000000000001
10.536  0.707                      11.243   10038  800       11.243  local  -1         exec  benchmarks.encrypt_sign  6           72.50000000000001
12.526  0.697                      13.224   10038  800       13.224  local  -1         exec  benchmarks.encrypt_sign  7           72.50000000000001
14.505  0.707                      15.212   10038  800       15.212  local  -1         exec  benchmarks.encrypt_sign  8           72.50000000000001
16.494  0.695                      17.189   10038  800       17.189  local  -1         exec  benchmarks.encrypt_sign  9           72.50000000000001
18.471  0.705                      19.176   10038  800       19.176  local  -1         exec  benchmarks.encrypt_sign  10          72.50000000000001
20.457  0.701                      21.158   10038  800       21.158  local  -1         exec  benchmarks.encrypt_sign  11          72.50000000000001
22.443  0.709                      23.152   10038  700       23.152  local  -1         exec  benchmarks.encrypt_sign  12          72.33333333333333
24.334  0.699                      25.032   10038  700       25.032  local  -1         exec  benchmarks.encrypt_sign  13          72.33333333333333
26.212  0.701                      26.913   10038  700       26.913  local  -1         exec  benchmarks.encrypt_sign  14          72.50000000000001
28.098  0.69                       28.788   10038  700       28.788  local  -1         exec  benchmarks.encrypt_sign  15          72.50000000000001
29.97   0.711                      30.681   10038  700       30.681  local  -1         exec  benchmarks.encrypt_sign  16          72.50000000000001
31.861  0.713                      32.574   10038  700       32.574  local  -1         exec  benchmarks.encrypt_sign  17          72.50000000000001
....
```

## Remote Offloading

We have already deployed a pre-configured Intel SGX worker-server ready to handle requests to make testing easier.
If you're interested in deploying your own SGX server, follow
[this guide](https://github.com/atlas-runtime/atlas-guides/blob/main/setup_atlas.md) first.

### Set Up

```sh
# Go to the atlas-client folder
$ cd $ATLAS_ROOT/atlas-client/
# edit the `atlas-addresses.txt` with your configuration in format `PORT IP`
# For the purpose of this guide, we have already deployed a remote worker, ready to handle requests
```

### Math Example script

```sh
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --servers 1 --file benchmarks/math/run.js --log log.dat
```

The expected output log should be something similar to this (depending also on your hardware, network and offloading function)
```sh
$ cat log.dat
######################################################################################################################
#Start  Battery:72.83333333333334
#Start  Duration                   Latency  Bytes  Interval  End    Mode    Thread_ID  Type  Function   Request_ID  Battery_Status
1.392   0.49                       0.509    513    -1        1.901  remote  0          exec  math.add   0           72.83333333333334
1.401   0.477                      0.996    186    -1        2.397  remote  0          exec  math.sub   1           72.83333333333334
1.403   0.534                      1.545    186    -1        2.948  remote  0          exec  math.div   2           72.83333333333334
1.405   0.538                      2.092    187    -1        3.497  remote  0          exec  math.mult  3           72.83333333333334
```

### Encrypt and Sign Test

```sh
# goto atlas-client folder
$ cd $ATLAS_ROOT/atlas-client
# this specific test, starts generating packets dynamically at different intervals
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --file benchmarks/crypto_benchmark/run.js --servers 1 --log crypto_r.log
```

This should take about 1 minute (compare with the 2-3 minutes that it took in the local case).
The execution log should look similar to this:

```sh
$ cat crypto_r.dat
#Start  Battery:72.50000000000001
#Start  Duration                   Latency  Bytes  Interval  End     Mode    Thread_ID  Type  Function                 Request_ID  Battery_Status
1.6     1.448                      1.521    88248  800       3.121   remote  0          exec  benchmarks.encrypt_sign  0           72.50000000000001
2.418   0.553                      1.308    10252  800       3.726   remote  0          exec  benchmarks.encrypt_sign  1           72.50000000000001
3.676   0.534                      0.609    10252  800       4.285   remote  0          exec  benchmarks.encrypt_sign  2           72.50000000000001
4.765   0.549                      0.567    10252  800       5.332   remote  0          exec  benchmarks.encrypt_sign  3           72.50000000000001
5.847   0.552                      0.569    10252  800       6.416   remote  0          exec  benchmarks.encrypt_sign  4           72.50000000000001
6.914   0.548                      0.566    10252  800       7.48    remote  0          exec  benchmarks.encrypt_sign  5           72.50000000000001
7.992   0.556                      0.573    10252  800       8.565   remote  0          exec  benchmarks.encrypt_sign  6           72.50000000000001
9.078   0.553                      0.57     10252  800       9.648   remote  0          exec  benchmarks.encrypt_sign  7           72.50000000000001
10.168  0.563                      0.58     10252  800       10.748  remote  0          exec  benchmarks.encrypt_sign  8           72.50000000000001
11.258  0.548                      0.565    10252  800       11.823  remote  0          exec  benchmarks.encrypt_sign  9           72.50000000000001
12.34   0.561                      0.579    10254  800       12.919  remote  0          exec  benchmarks.encrypt_sign  10          72.50000000000001
13.424  0.556                      0.576    10254  700       14      remote  0          exec  benchmarks.encrypt_sign  11          72.50000000000001
14.517  0.551                      0.568    10254  700       15.085  remote  0          exec  benchmarks.encrypt_sign  12          72.50000000000001
15.604  0.556                      0.574    10254  700       16.178  remote  0          exec  benchmarks.encrypt_sign  13          72.50000000000001
16.69   0.562                      0.58     10254  700       17.27   remote  0          exec  benchmarks.encrypt_sign  14          72.50000000000001
17.785  0.587                      0.605    10254  700       18.39   remote  0          exec  benchmarks.encrypt_sign  15          72.50000000000001
18.886  0.556                      0.574    10254  700       19.46   remote  0          exec  benchmarks.encrypt_sign  16          72.50000000000001
19.976  0.556                      0.573    10254  700       20.549  remote  0          exec  benchmarks.encrypt_sign  17          72.50000000000001
....
````

In the remote case, the Request with ID = 17, returns to the user at 20.549s whereas in the local execution it needs 32.574s, which is 58.51% slower!
We could even increase this difference when using much faster network connections but also changing the transmitted buffers!

## How Atlas works

Consider the following Javascript math library
```js
function add(a, b) {
    return a + b;
}
function sub(a, b) {
    return a - b;
}
function mult(a, b) {
    return a * b;
}
function div(a, b) {
    return a / b;
}
let math = {};
math.add = add;
math.sub = sub;
math.mult = mult;
math.div = div;
export {math};
```
We want to perform some basic computations using this library, so we do the following:
```js
import {math} from 'benchmarks/math/math.js';
math.add(12,34).then(console.log);  // should print 12 + 34 = 46
math.sub(1,6).then(console.log);    // 1/6 = 0.16666666666
math.div(1,0).then(console.log);    // Infinity
math.mult(1,5).then(console.log);   // 5
```

Since all the calls to atlas are asynchronous (using worker-threads), the results will also be returned asynchronously.
Thus, we need to use the `then` keyword to evaluate/resolve the `async/promise` calls.

Here's what happens under the hood:
1. We provide our driver code to the atlas-interpreter
2. While the code is being parsed, additional code is crafted on-the-fly and is injected to the original code
3. Atlas detects all the imports and logs them.
4. After the module detection and the code injection is done, normal execution of Javascript starts.
5. If enabled, Atlas offloads to the remote party the dependencies and the imports, so the server may replicate the original context.
6. When JavaScript execution reaches to a wrapped library function call, atlas offloads each request.

The new generated code after step 3 will look similar to this:
```js
import {math} from 'benchmarks/math/math.js';
if (globalThis.mathiswrapped === false) {  // have we already wrapped the library?
    math = atlas_wrapper(math);            // wrap the library using atlas, so it can be automatically scaled-out
    globalThis.mathiswrapped = true;       // math is wrapped
}
// then, all the next calls to math, we be offloaded using atlas!
math.add(12,34).then(console.log);  // should print 12 + 34 = 46
math.sub(1,6).then(console.log);    // 1/6 = 0.16666666666
math.div(1,0).then(console.log);    // Infinity
math.mult(1,5).then(console.log);   // 5
```
