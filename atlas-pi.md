# Deploying Atlas on Raspberry PI Zero
Quick jump: [Introduction](#introduction) | [Installation](#installation) | [Repo Structure](#repo-structure) | [Running Scripts](#running-scripts) | [What Next?](#what-next)

This tutorial focuses strictly on installing and deploying atlas on Raspberry PI.

## Introduction

Two main componenents are needed for this guide:

* **atlas-client**
    The orchistrating component written in JavaScript and is responsible for offloading execution requests.
* **atlas-qjs**
    Modified QuickJS engine packed with cryptographic and networking capabilities. Also, it performs transformations to the user-code at parsing phase, to automatically wrapp the user-code.

## High level overview of Atlas

Consider the following Javascript [math](https://github.com/atlas-runtime/atlas-client/blob/master/benchmarks/math/math.js) library
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
And we want to perform some basic computation using this library such as:
```js
import {math} from 'benchmarks/math/math.js';
math.add(12,34).then(console.log);  // should print 12 + 34 = 46
math.sub(1,6).then(console.log);    // 1/6 = 0.16666666666
math.div(1,0).then(console.log);    // Infinity
math.mult(1,5).then(console.log);   // 5
```

Since all the calls to atlas are asynchronous (using worker-threads), the results will also be returned asynchronously.
Thus, that's why we are using the ```then```, to evaluate/resolve the ```async```/```promise``` calls.
What really happens under the hood, is the following:
1. We provide our driver code to atlas-interpreter
2. While the code is being parsed, additional code is crafted on-the-fly and is injected to the original code
3. Atlas detects all the imports and logs all the imports of the code
4. After the module detection and the code injection is done, start the normal execution of Javascript
5. Atlas offloads to the remote party the dependencies and the imports, so the server may setup the original context

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

## Installation

On any Linux distribution, installing and setting up atlas is easy and is split into two sections:  
 1. Install the client code
 2. Install and deploy on the cloud/local with SGX hardware/simulation mode

### Setting up the client environment
```sh
# create the root folder for our project
mkdir atlas_root && cd atlas_root
# fetch the repo of atlas-client
git@github.com:atlas-runtime/atlas-client.git --depth 1
# fetch the atlas interpreter
git clone git@github.com:atlas-runtime/atlas-qjs.git --depth 1 quickjs
# export the required environment variable
# replace the path to match to your local atlas installation
$ export ATLAS_ROOT=/home/dkarnikis/atlas_root/
# enter the source code folder of QuickJS for the client where $ATLAS_ROOT is the root folder of atlas 
$ cd $ATLAS_ROOT/quickjs/src
# build the client code, it will generate qjs binary 
$ make qjs
# return to the root directory and go to atlas-client 
$ cd $ATLAS_ROOT/atlas-client/
# edit the `atlas-addresses.txt` with your configuration in format `PORT IP`
# For the purpose of this demo, I have deployed already deployed a remote worker, ready to handle requests
```

## Running Scripts
All scripts in this guide assume that are being executed from `$ATLAS_ROOT/atlas-client`

### Math Example

The simplest way to try out atlas is executing `math` library, that performs simple arithmetic operations.
The file JavaScript file is located at `$ATLAS_ROOT/atlas-client/benchmarks/math/math.js`  
To run it **locally and sequentially**, you would call it using `--local` flag:  
```sh
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
#Start  Battery:80.16666666666669
#Start  Duration                   Latency  Bytes      Interval  End    Mode   Thread_ID  Type  Function   Request_ID  Battery_Status
0.482   0                          0.482    undefined  -1        0.482  local  -1         exec  math.add   0           80.16666666666669
0.959   0                          0.959    undefined  -1        0.959  local  -1         exec  math.sub   0           80.16666666666669
1.438   0                          1.438    undefined  -1        1.438  local  -1         exec  math.div   0           80.16666666666669
1.918   0.001                      1.918    undefined  -1        1.918  local  -1         exec  math.mult  0           80.16666666666669    
```

To run it **remotely** and using our deployed SGX server:
```sh
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --servers 1 --file benchmarks/math/run.js --log log.dat
```
The expected output log should be something similar to this (depending also on your hardware, network and offloading function)  
```sh
$ cat log.dat
######################################################################################################################
#Start Battery:80.16666666666669                                                 
#Start Duration Latency Bytes Interval End Mode Thread_ID Type Function Request_ID Battery_Status
0.481   0   0.481   undefined   -1  0.481   local   -1  exec    math.add    0   80.16666666666669
0.958   0   0.958   undefined   -1  0.958   local   -1  exec    math.sub    0   80.16666666666669
1.436   0   1.436   undefined   -1  1.436   local   -1  exec    math.div    0   80.16666666666669
1.915   0   1.915   undefined   -1  1.915   local   -1  exec    math.mult   0   80.16666666666669
```

### Encrypt and Sign Example

For our second demo application, we will be using a program fragment that performs a simple AES encrypt and HMAC Sign. Using atlas we should only provide a data buffer (to be encrypted) but also a pair of cryptographic keys (one used for the encryption and the second for the signing). 

Our application is based on [crypto-es](https://github.com/entronad/crypto-es).

The source code for this JavaScript benchmark is located at 
`$ATLAS_ROOT/atlas-client/benchmarks/crypto-benchmark/run.js` 

and our custom crypto wrapper for encrypt and sign

`$ATLAS_ROOT/atlas-client/benchmarks/crypto_benchmark/crypto-wrapper.js`

To run it **locally and sequentially**, you would call it using `--local` flag:  
```sh
# enter the atlas client
$ cd $ATLAS_ROOT/atlas-client/
# execute the code
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --local --file benchmarks/crypto_benchmark/run.js --log crypto_l.dat
```
The expected output should be something like this:
```sh
$ cat crypto_l.dat
#Start  Duration  Latency  Bytes      Interval  End     Mode   Thread_ID  Type  Function                 Request_ID  Battery_Status
0.936   0.615     1.551    undefined  800       1.551   local  -1         exec  AES.encrypt              0           80.66666666666666
2.034   1.002     3.036    undefined  800       3.037   local  -1         exec  sha.HmacSHA512           0           80.33333333333333
0.936   2.584     3.521    undefined  800       3.521   local  -1         exec  benchmarks.encrypt_sign  0           80.33333333333333
4.813   0.644     5.457    undefined  800       5.457   local  -1         exec  AES.encrypt              1           80.33333333333333
5.942   1.002     6.944    undefined  800       6.944   local  -1         exec  sha.HmacSHA512           1           80.33333333333333
4.812   2.616     7.428    undefined  800       7.428   local  -1         exec  benchmarks.encrypt_sign  1           80.33333333333333
8.716   0.659     9.375    undefined  800       9.375   local  -1         exec  AES.encrypt              2           80.33333333333333
9.862   0.999     10.862   undefined  800       10.862  local  -1         exec  sha.HmacSHA512           2           80.33333333333333
8.715   2.631     11.346   undefined  800       11.346  local  -1         exec  benchmarks.encrypt_sign  2           80.33333333333333
12.638  0.651     13.289   undefined  800       13.289  local  -1         exec  AES.encrypt              3           80.33333333333333
13.772  0.997     14.769   undefined  800       14.769  local  -1         exec  sha.HmacSHA512           3           80.33333333333333
12.637  2.615     15.252   undefined  800       15.252  local  -1         exec  benchmarks.encrypt_sign  3           80.33333333333333
16.542  0.667     17.209   undefined  800       17.209  local  -1         exec  AES.encrypt              4           80.33333333333333
17.696  1         18.696   undefined  800       18.696  local  -1         exec  sha.HmacSHA512           4           80.33333333333333
16.542  2.636     19.178   undefined  800       19.178  local  -1         exec  benchmarks.encrypt_sign  4           80.33333333333333
20.47   0.672     21.142   undefined  800       21.142  local  -1         exec  AES.encrypt              5           80.33333333333333
21.628  1.005     22.633   undefined  800       22.633  local  -1         exec  sha.HmacSHA512           5           80.33333333333333
20.469  2.65      23.119   undefined  800       23.119  local  -1         exec  benchmarks.encrypt_sign  5           80.33333333333333
24.409  0.654     25.063   undefined  800       25.063  local  -1         exec  AES.encrypt              6           80.33333333333333
25.552  1.008     26.56    undefined  800       26.56   local  -1         exec  sha.HmacSHA512           6           80.33333333333333
24.408  2.644     27.052   undefined  800       27.052  local  -1         exec  benchmarks.encrypt_sign  6           80.33333333333333
28.346  0.637     28.983   undefined  800       28.983  local  -1         exec  AES.encrypt              7           80.33333333333333
29.467  0.994     30.461   undefined  800       30.461  local  -1         exec  sha.HmacSHA512           7           80.33333333333333
28.345  2.602     30.947   undefined  800       30.947  local  -1         exec  benchmarks.encrypt_sign  7           80.33333333333333
32.237  0.659     32.896   undefined  800       32.896  local  -1         exec  AES.encrypt              8           80.33333333333333
33.38   1.004     34.384   undefined  800       34.384  local  -1         exec  sha.HmacSHA512           8           80.33333333333333
32.236  2.636     34.872   undefined  800       34.872  local  -1         exec  benchmarks.encrypt_sign  8           80.33333333333333
....
```

```sh
# goto atlas-client folder
$ cd $ATLAS_ROOT/atlas-client;
# this specific test, starts generating packets dynamically at different intervals
$ $ATLAS_ROOT/quickjs/src/qjs atlas.js --file benchmarks/macro/eval/streaming.js --servers 1 --log crypto_r.log
```

The execution log should similar to this:

```sh
$ cat crypto_r.dat
#Start  Duration  Latency  Bytes   Interval  End     Mode    Thread_ID  Type  Function                 Request_ID  Battery_Status
1.929   1.86      1.942    117317  800       3.871   remote  0          exec  benchmarks.encrypt_sign  0           82.16666666666669
2.741   1.098     2.312    39142   800       5.053   remote  0          exec  benchmarks.encrypt_sign  1           82.16666666666669
3.547   1.085     2.674    39142   800       6.221   remote  0          exec  benchmarks.encrypt_sign  2           82.16666666666669
4.445   0.949     2.786    39142   800       7.231   remote  0          exec  benchmarks.encrypt_sign  3           82.16666666666669
5.623   0.953     2.625    39142   800       8.248   remote  0          exec  benchmarks.encrypt_sign  4           82.16666666666669
6.768   0.934     2.49     39142   800       9.258   remote  0          exec  benchmarks.encrypt_sign  5           82.16666666666669
7.796   0.942     2.463    39142   800       10.259  remote  0          exec  benchmarks.encrypt_sign  6           82.16666666666669
8.822   0.94      2.439    39142   800       11.261  remote  0          exec  benchmarks.encrypt_sign  7           82.16666666666669
9.81    1.062     2.585    39142   800       12.395  remote  0          exec  benchmarks.encrypt_sign  8           82.16666666666669
10.83   1.1       2.735    39142   800       13.565  remote  0          exec  benchmarks.encrypt_sign  9           82.33333333333334
11.83   0.929     2.725    39144   800       14.555  remote  0          exec  benchmarks.encrypt_sign  10          82.16666666666669
12.947  0.94      2.609    39144   700       15.556  remote  0          exec  benchmarks.encrypt_sign  11          82.16666666666669
14.136  0.941     2.446    39144   700       16.582  remote  0          exec  benchmarks.encrypt_sign  12          82.16666666666669
15.126  0.937     2.465    39144   700       17.591  remote  0          exec  benchmarks.encrypt_sign  13          82.16666666666669
16.13   0.985     2.518    39144   700       18.648  remote  0          exec  benchmarks.encrypt_sign  14          82.16666666666669
17.143  1.113     2.693    39144   700       19.836  remote  0          exec  benchmarks.encrypt_sign  15          82.16666666666669
18.139  1.156     2.942    39144   700       21.081  remote  0          exec  benchmarks.encrypt_sign  16          82.33333333333334
19.223  0.962     2.884    39144   700       22.107  remote  0          exec  benchmarks.encrypt_sign  17          82.16666666666669
20.412  1.179     2.944    39144   700       23.357  remote  0          exec  benchmarks.encrypt_sign  18          82.16666666666669
21.654  0.804     2.583    39144   700       24.237  remote  0          exec  benchmarks.encrypt_sign  19          82.16666666666669
22.679  0.932     2.556    39144   700       25.235  remote  0          exec  benchmarks.encrypt_sign  20          82.33333333333334
23.92   0.935     2.317    39144   700       26.237  remote  0          exec  benchmarks.encrypt_sign  21          82.16666666666669
24.787  0.946     2.463    39144   700       27.25   remote  0          exec  benchmarks.encrypt_sign  22          82.16666666666669
25.812  0.937     2.462    39144   700       28.275  remote  0          exec  benchmarks.encrypt_sign  23          82.16666666666669
....
````