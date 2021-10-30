# Numerical Computing on Apple M1 : Study and Implementations

Study and Implementations of Numerical Algorithms on Apple M1 and A* Devices

# Table of Contents (Sections)

* [Memcpy](01_memcpy/README.md) 
* [Saxpy](02_saxpy/README.md)
* [Dot Product](03_dot/README.md)
* [Prefix-Sum/Prefix-Scan](04_prefix_sum/README.md)
* [Sort (GPU radix Sort on int and float)](05_radix_sort/README.md)
* [N-Body (particle simulation)](06_nbody/README.md)
* [2D convolution with 5x5 kernel](07_2d_filter/README.md)
* [Sparse Matrix-Vector Multiplication](08_sparse_matrix_vector_mul/README.md)
* [Dense Matrix-Vector Multiplication](09_dense_matrix_vector_mul/README.md)
* [Cholesky Decomposition](10_cholesky_decomp/README.md)
* [Jacobi Iterative Solver](11_jacobi_solver/README.md)
* [Gauss-Seidel Iterative Solver](12_gauss_seidel_solver/README.md)
* [512-point FFT](13_fft/README.md)

# Background Context

- Apple M1 is an awesome chip for interactive apps with heavy numerical computation

- Lacking study material and examples. They are not as rich as the ones for CUDA.

This is a collection of technical reports about my study, implementation and experiments for numerical computation on Apple M1.
When the first Mac Mini M1 was released in 2020, I was super excited about the ARM-based architecture with the unified memory access to both CPUs and GPUs.
It seemed to be the best consumer computing device for the modern interactive applications with graphics rendering, audio processing, physics simulation, and neural networks, etc.
As we are apparently approaching the end of Moore's law, it is important to take advantage of parallelism more than ever, either in better pipelining, SIMD, fine grain multithreading, or GPU, and
the Apple M1 provides probably the best solution with the clean ARM architecture, rich libraries like vDSP and BLAS in Accelerate framework, and Metal GPGPU.

However, I have quickly realized that there are relatively few references, tutorials, and articles about the GPGPU computing on Apple devices,
after I started studying the numerical computing as a preparation work for my interactive applications on iOS and Macos that involve heavy numerical calculation.
For CUDA there are rich resources, such as GPU Gems by NVIDIA, and CUDA Handbook etc.

Since then, I have started applying to Metal what I had learned from my experience in CUDA GPGPU programming.
Especially I have started applying the idioms such as reduction and prefix-sum presented in the later chapters of CUDA Handbook, and wanted to see how it shakes out on Metal.
At the same time, I have also started working on some basic numerical operations for both GPU and CPUs as a basis for more elaborated work planned for some real applications.
The solutions on CPU,  either with the existing libraries/frameworks or in my own implementations,  are as important as the solutions on GPU.
There are already excellent collection of libraries, notably vDSP & BLAS, and the running time on CPU is usually faster when the problem size is small,
and the overhead of launching the GPU kernels can not be amortized.

After a while I have accumulated a pretty good collection of implementations and experimental results enough to be shared in public.
I have reorganized them into 13 sections above, and made it more presentable, and this repository has come out in the end.


# Purpose

* To target realtime interactive applications

* To provide various implementations for 13 representative types of computation

* To provide a Metal equivalent to Chap. 11-15 of CUDA Handbook

* To present real running time information on MacOS (Apple Mac Mini 2020 8GB, for iOS devices planned later).

* To give an indication as to what type of technology/implementation works best for a particular type of problem in a given data size

* To give a guideline on how to implement own solution if there is no existing framework or library available

The target application is the realtime interactive applications on iOS and Macos that execute heavy numerical computation.
For now, it provides implementations and results for MacOS BigSur on Mac Mini. An edition for the iOS devices is planned to be released later.

The collection is split into 13 sections. About half of them are idioms in GPGPU computing, and the others are representative types of numerical computing.
On CPU, each section first implements the algorithm in a plain C++ code, tests the running times for the problems in various sizes, and establish the baseline.
If there are any existing solutions provided by some libraries, such as BLAS, then it makes an implementation with those routines, and tests the running times.
Then it tries to improve the baseline C++ implementation by reviewing the algorithm design, the data organization, and a few techniques, notably multithreading, NEON intrinsics, and loop unrolling. It then tests the running time and analyze the results.
On GPU, if there is an idiom of GPGPU described in CUDA Handbook, it tries to implement the algorithms presented in the book to Metal. Some changes have to be made to absorb the differences between CUDA and Metal. Otherwise, it tries to make implementations according to the publicly available literature in GPGPU computing.
In some cases I had to design the algorithms based on common knowledge in GPGPU computing.

With those implementations and test results on the running time, it provides insight into what type of technology/implementation works best for particular type of problem in a given data size on the particular device (at the time of writing Mac Mini M1 8G BigSur 11.6).
It can also be used as a guideline on how to implement custom numerical solution if there is no existing framework or library, either by combining low-level routines in the libraries, or by writing in C++ with NEON intrinsics.
For example, if you have to implement a variant of the projected Gauss-Seidel iterative solver, for which there is no direct routine is provided by either LAPACK, BLAS, or GNU Scientific Library, you can use this as a guideline for your own implementation in C++.

# Status

* **Oct 2021** : The version (this repo) for Apple M1 using Mac mini (M1, 2020), 8GB Memory has been converged.

* **Jan 2022** : Another version for Apple A15 Bionic using an iPhone 13 planned.

# Technical Summary

## Main Technologies and Aspects Considered

* CPU
  * Apple Accelerate Framework (vDSP,BLAS,LAPACK)
  * Other Apple Frameworks (CIImage)
  * 3rd-party libraries (BOOST, GNU Scientific library, & Eigen3)
  * NEON SIMD intrinsics
  * Explicit loop unrolling
  * Cache prefetch
  * Multithreading

* GPU
  * Metal Perfomance Shaders ( MPSImageConvolution, MPSMatrixVectorMultiplication, MPSMatrixDecompositionCholesky, & MPSMatrixSolveTriangular)
  * Own Implementations in Metal Compute Kernels

    * Coalesced load & store to the device memory
    * Managed device memory vs shared device memory
    * Use of threadgroup memory vs device memory
    * Bank conflict in the threadgroup memory
    * The simd (warp-level) instructions
    * Atomic operations on int and float
    * Multiple thread-groups vs loops in one thread-group
    * Explicit loop unrolling

**NOTE1: Assembler not considered** 

Coding directly in assembly language is not considered, as I'm not capable of writing optimized assembly code.

**NOTE2: Multithreading with CondVar**

For multithreaded implementations, I used [ThreadSynchronizer](https://github.com/ShoYamanishi/ThreadSynchronizer)
to minimize the overhead of synchronizing threads.
This repo utilizes mainly condition variables, and provides OpenMP-like functionalities as well as CUDA's __synchthreads() equivalent.


## How to Run the Experiments

To run all the experiments, type `make all` on the top directory.
To run the experiments for one section, `make all` in the directory. 
The target *all* of Make does the following:
1. Build the test executable, and also metal libraries if necessary.
2. Run the executable and perform experiments according to the top-level C++-code 'test_*.cpp'.
3. The executable generates a log in `doc/make_log.txt`
4. A python script [common/process_log.py](common/process_log.py) scan the make_log.txt and plots charts according to the specification given in 'doc/plot_spec.json'.


## General Findings on CPU

* BLAS & vDSP routines are highly optimized per architecture, and they should be used in general wherever applicable.

* Explicit use of NEON intrinsics with explicit loop unrolling of factor 4 or 8 usually works well.

* Multithreading are usually beneficial, and usually 4-8 threads are the sweet spots.

## General Findings on GPU

* The more thread-groups, the faster it executes. Please see [DOT](03_dot/README.md) for details.

* The last block detection technique can not be used for Metal. Please see [DOT](03_dot/README.md) for details.

* There is no severe penalty in uncoalesced loads from the device memory. Please see 'METAL TWO_PASS_DEVICE_MEMORY 0 0' and 'METAL TWO_PASS_SHARED_MEMORY 0 0' in [DOT](03_dot/README.md) for details.

* There is no severe penalty in uncoalesced writes to the device memory. Please see 'METAL UNCOALESCED_WRITE' and 'METAL COALESCED_WRITE' in [RadixSort](05_radix_sort/README.md) for details.

* Multiple *threadgroup* function parameters can not be used for Metal kernels. Please see [Prefix-Scan](04_prefix_sum/README.md) for details.

* Explicit loop unrolling in the kernels does not seem to improve the running time. CUDA has a pragma *unroll <factor>*, and some improvements with it are reported in Chapter 14 of CUDA Handbook. However, Metal does not have such a compiler directive, and according to my study in [NBody](06_nbody/README.md),
explicit unrolling of the loop body does not only improve performance, but also causes runtime error if the factor is greater than 4.

* Apparently there is a limit in the number of *thread (local)* variables defined in one kernel even for *consts*. This is observed in [2DConvolution](07_2d_filter/README.md). Please see the kernels used in 'METAL TWO_STAGE'. It uses many 'const float' in the kernel `convolution_5x5_stage2()`.
To make it work, the number of threads per thread-group has to be reduced from 1024 to 768. Otherwise, the output from this kernel is wrong, and there is no compiler or runtime error for it.


# File Organization
This repo consists of 13 independent sections at the top directory, numbered from '01_*' to '13_*'.
Each section is dedicated to a particular type of numerical computation, and has an accompanying README as listed above, and the main source file named `test_*.cpp`

Each README consists of the following format:
* Key Points
* Background, Context, and Purpose
* Results on Running Time and Analyses
* Implementations


The commonly used functionalities are placed under [./common](./common).
They are mainly for the test framework, sample data generation, output inspection, and time measurements.


# Give Me Your Feedback!

* Any corrections, advice, and suggestions are appreciated.

* Indy developers, small businesses, and students, please use the materials provided here under MIT license.
I also try to answer questions and give support as much as possible, but without any warranty.

* Further study on particular device, for particular type of problem and applications in a commercial setting can be arranged upon request for fee.

* Donation of the latest Apple devices are very welcome.

# License

MIT License

# References
The references are listed in the README.md in each section (subdirectory).




