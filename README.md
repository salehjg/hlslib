_Include errors with latest commits:_ in an effort to support both Xilinx Vivado HLS and Intel FPGA OpenCL, the project has been restructured. If you are using `hlslib` includes, please change the include path from `hlslib/<Name>.h` to `hlslib/xilinx/<Name>.h`.

# Overview

## What is included

This repository includes the following features for use with Vivado HLS and/or SDAccel projects:

* `DataPack` class to ease SIMD parallelization in HLS kernels (`include/hlslib/xilinx/DataPack.h`) 
* Utility classes for SDAccel host side integration, located at `include/hlslib/xilinx/SDAccel.h`, including support for specifying FPGA memory banks
* Thread-safe stream implementation, with blocking semantics to mimic hardware behavior (`include/hlslib/xilinx/Stream.h`)
* Macros to easily simulate dataflow functions using multithreading when compiled as C++, while being compatible with high-level synthesis when building for hardware (`include/hlslib/xilinx/Simulation.h`)
* CMake module files that locate SDAccel, Vivado, Vivado HLS and Vivado Lab on the host machine, setting relevant variables for build integration with CMake
* Classes to flatten loop nests and keep track of indices (`include/hlslib/xilinx/Flatten.h`), both for bounds known at runtime (`hlslib::Flatten`) and bounds known at compile-time (`hlslib::ConstFlatten`). Example usage can be found in `xilinx_test/kernels/Flatten.cpp`.
* Various compile-time functions commonly used when designing hardware, such as log2, in `include/hlslib/xilinx/Utility.h`
Some of these headers are interdependent, while others can be included standalone. Refer to the source code for details.
* A template tcl-file that can be used with CMake or another templating engine to produce a high-level synthesis script
* `xilinx_test/CMakeLists.txt` that builds a number of tests to verify hlslib functionality, doubling as a reference for how to integrate HLS projects with CMake using the provided module files 
* An example of how to use the Simulation and Stream headers, at `xilinx_test/kernels/MultiStageAdd.cpp`, both as a host-only simulation (`xilinx_test/test/TestMultiStageAdd.cpp`), and as a hardware kernel (`xilinx_test/host/RunMultiStageAdd.cpp`) 
* `include/hlslib/xilinx/Accumulate.h`, which includes a streaming implementation of accumulation, including for type/operator combinations with a non-zero latency (such as floating point addition). Example kernels of usage for both integer and floating point types are included as `xilinx_test/kernel/AccumulateInt.cpp` and `xilinx_test/kernel/AccumulateFloat.cpp`, respectively. 
* `include/hlslib/xilinx/Operators.h`, which includes some commonly used operators as functors to be plugged into templated functions such as `TreeReduce` and `Accumulate`.
* `include/hlslib/xilinx/Axi.h`, which implements the AXI Stream interface and the bus interfaces required by the DataMover IP, enabling the use of a command stream-based memory interface for HLS kernels if packaged as an RTL kernel where the DataMover IP is connected to the AXI interfaces.

## Examples

For some real life example of using the core features of `hlslib`, take a look at our matrix-matrix multiplication and stencil codes:

- https://github.com/spcl/gemm_hls
- https://github.com/spcl/stencil_hls

## Installation

The source code provided in this repository can be used on a file-by-file basis, and as such does not need to be compiled/installed. However, installation with CMake is supported, which you can use to install headers and CMake files directly into your project.

When an hlslib header is included, compilation must allow C++11 features, and the macro `HLSLIB_SYNTHESIS` must be set whenever high-level synthesis is run, using `-cflags "-std=c++11 -DHLSLIB_SYNTHESIS"` and `--xp prop:kernel.<entry function>.kernel_flags="-std=c++11 -DHLSLIB_SYNTHESIS"` for Vivado HLS and SDAccel, respectively. See the included `CMakeLists.txt` for reference.

## Compatibility

The code within has been tested with SDx 2018.2 and Vivado HLS 2018.2 for the HLS code, and GCC 7.3.0 for host/simulation code. There is no plan to accommodate older versions of any of these tools, although there is no principal reason why older versions should not work. Newer versions of the tools are expected to work. If this is not the case, it should be considered a bug. 

# Useful information

### Running SDAccel

It is beneficial to make a custom script for setting up the SDAccel environment, as the default one generated by Xilinx sets `LD_LIBRARY_PATH`, which generally is bad practice, and will conflict with binaries compiled under a different environment.

The following is sufficient as of SDx 2018.2:

```shell
#!/bin/sh
export XILINX_OPENCL=<path to installed DSA>/xbinst
export PATH=$XILINX_OPENCL/runtime/bin:$PATH
unset XILINX_SDACCEL
unset XILINX_SDX
unset XCL_EMULATION_MODE
```

### Bundled libc++

When linking against the Xilinx OpenCL libraries, the linker must add the runtime library folder to the library search path. This folder contains an ancient version of libc++.so, which will break compilation when using a newer compiler.
To avoid this, backup or delete the library file located at:
```
<SDx or SDAccel folder>/runtime/lib/<architecture>/libstdc++.so
```

### relax\_ii\_for_timing

SDAccel sets the option `relax_ii_for_timing`, and a conservative clock uncertainty of 27% of the target timing. This means that it will silently increase the initiation interval if this more conservative constraint is not met, resulting a slowdown of 2x to the resulting performance. Check the HLS log file to see if your design was throttled, located at:
```
_xocc_<source file>_<kernel file>.dir/impl/build/system/<kernel file>/bitstream/<kernel file>_ipi/ipiimpl/ipiimpl.runs/impl_1/<"vivado_hls" or "runme">.log
```
In newer versions of SDAccel, the final initiation interval determined by Vivado HLS is printed to standard output when running xocc. However, this output is suppressed after 100 messages, so for large kernels it is necessary to verify either the `runme.log` or `vivado_hls.log` file.

### Ubuntu packages

On Ubuntu, the following package might need to be installed to run hardware emulation:

```
sudo apt install libc6-dev-i386
```
