Musings in GEMM (General Matrix Multiplication)
-----------------------------------------------

Fooling around with flops/clock in the famous SGEMM - what could be more fun? GEMM generally does `C += A * B` where A, B and C are large-ish dense matrices of single (SGEMM) or double precision (DGEMM) floats.

Usage
-----

The low-tech bash script `build_sgemm.sh` will try to build the test for a recognized host architectures - substitute the compiler for one of your choice. Macros of interest, passed with `-D` on the command line:

* `ALT` - implementation alternatives
	* -1 - scalar version
	*  0 - 16-element-wide version suitable for autovectorizers
	*  1 - 2x16-element-wide SSE2 (x86/amd64) version
	*  2 - 64-element-wide AVX256 (x86/amd64) version
	*  3 - 2x64-element-wide AVX256 (x86/amd64) version
	*  4 - 16-element-wide ASIMD2 (arm/aarch64) version
	*  5 - 32-element-wide ASIMD2 (arm/aarch64) version
	*  6 - 2x16-element-wide ASIMD2 (arm/aarch64) version
	*  7 - 2x32-element-wide ASIMD2 (arm/aarch64) version
	*  8 - 2x16-element-wide MSA (mips32/mips64) version
* `PREFETCH` - distance, in floats, to prefetch in the innermost loop (0 for no prefetch; unused in the scalar version)
* `MATX_SIZE` - dimension of the square matrices A, B & C
* `REP_EXP` - exponent of the number of repetitions of the test, ie. 1eEXP
* `PRINT_MATX` - print out C on the standard output (for debugging)

Tips
----

To tell what prefetch works best on a given CPU and matrix dimension, use something along the following (pick ALT wisely):

	for i in {0..10} ; do ./build_sgemm.sh -DALT=1 -DPREFETCH=`echo "512 + 512 * $i" | bc` -DMATX_SIZE=512 -DREP_EXP=1 ; ./sgemm ; done

Results
-------

Best results measured in SP flops/clock by the formula:

	MATX_SIZE^3 * 2 * 10^REP_EXP / (CPU_freq * duration)

| CPU (single thread only)   | width of SIMD ALU | RAM GB/s  | LLC visible per core | 64x64    | 512x512  | remarks [^1]                                                                |
| -------------------------- | ----------------- | --------- | -------------------- | -------- | -------- | --------------------------------------------------------------------------- |
| AMD C60 (Bobcat)           | 2-way             | 8.53      | 512 KB               | 1.94     | 1.47     | g++     4.8, ALT = 1, PREFETCH = 2560, SSE2 intrinsics, 1.33GHz             |
| Intel Core2 T5600          | 4-way             | 5.33      | 2 MB                 | 3.31     | 2.82     | clang++ 3.4, ALT = 1, PREFETCH = 4096, SSE2 intrinsics, 1.83GHz             |
| Intel Core2 P8600          | 4-way             | 8.53      | 3 MB            [^2] | 4.86     | 4.14     | apple clang 8.1, ALT = 1, PREFETCH = 2048, SSE2 intrinsics, 2.40GHz         |
| Intel E5-2687W (SNB)       | 8-way             | 25.6      | 20 MB           [^2] | 13.79    | 10.17    | clang++ 3.6, ALT = 3, PREFETCH = 3584, AVX256 intrinsics, 3.1GHz            |
| Intel E5-2687W (SNB)       | 8-way             | 25.6      | 20 MB           [^2] | 14.27    | 10.25    | g++     4.8, ALT = 3, PREFETCH = 3584, AVX256 intrinsics, 3.1GHz            |
| Intel E3-1270v2 (IVB)      | 8-way             | 25.6      | 8 MB            [^2] | 13.40    | 11.05    | clang++ 3.6, ALT = 3, PREFETCH = 3072, AVX256 intrinsics, 1.6GHz            |
| Intel E3-1270v2 (IVB)      | 8-way             | 25.6      | 8 MB            [^2] | 14.01    | 11.22    | g++     4.8, ALT = 3, PREFETCH = 3072, AVX256 intrinsics, 1.6GHz            |
| Intel i7-4770 (HSW)        | 8-way             | 25.6      | 8 MB            [^2] | 22.72    | 11.65    | g++     5.1, ALT = 3, PREFETCH = 2560, AVX256+FMA3 intrinsics, 3.9GHz       |
| AMD Ryzen 1700X (Zen)      | 4-way             | 37.5      | 16 MB           [^2] | 14.15    | 10.22    | clang++ 3.8, ALT = 3, PREFETCH = 3072, AVX256 intrinsics, 3.4GHz            |
| RK3368 (Cortex-A53)        | 2-way             | 6.4       | 512 KB          [^3] | 3.12     | 1.39     | clang++ 3.6, ALT = 7, PREFETCH = 1536, ASIMD2 intrinsics, 1.51GHz           |
| Allwinner A64 (Cortex-A53) | 2-way             | 4.42      | 512 KB               | 3.18     | 1.38     | clang++ 3.6, ALT = 6, PREFETCH = 2560, ASIMD2 intrinsics, 1.152GHz [^4]     |
| MT8163A (Cortex-A53)       | 2-way             | 6.4       | 512 KB               | 3.09     | 1.65     | clang++ 3.6, ALT = 7, PREFETCH = 1536, ASIMD2 intrinsics, 1.5GHz            |
| MT8173C (Cortex-A53) *A32* | 2-way             | 6.4       | 512 KB               | 1.62     | 1.01     | clang++ 6.0, ALT = 6, PREFETCH = 2560, ASIMD intrinsics, 1.7GHz [^5]        |
| MT8173C (Cortex-A53)       | 2-way             | 6.4       | 512 KB               | 2.68     | 1.44     | clang++ 6.0, ALT = 6, PREFETCH = 2560, ASIMD2 intrinsics, 1.7GHz [^4]       |
| MT8173C (Cortex-A72) *A32* | 4-way             | 6.4       | 1 MB                 | 3.23     | 1.81     | clang++ 6.0, ALT = 7, PREFETCH = 2560, ASIMD intrinsics, 2.1GHz [^5]        |
| MT8173C (Cortex-A72)       | 4-way             | 6.4       | 1 MB                 | 6.82     | 2.30     | clang++ 6.0, ALT = 7, PREFETCH = 2560, ASIMD2 intrinsics, 2.1GHz [^6]       |
| Marvell 8040 (Cortex-A72)  | 4-way             | 12.8      | 1 MB                 | 6.52     | 2.91     | clang++ 3.5, ALT = 7, PREFETCH = 1536, ASIMD2 intrinsics, 1.3GHz [^6]       |
| AWS Graviton (Cortex-A72)  | 4-way             | 12.8      | 2 MB                 | 6.13     | 3.33     | clang++ 3.8, ALT = 7, PREFETCH = 2560, ASIMD2 intrinsics, 2.28GHz [^6] [^9] |
| AWS Graviton (Cortex-A72)  | 4-way             | 12.8      | 2 MB                 | 6.81     | 4.12     | clang++ 6.0, ALT = 7, PREFETCH = 1024, ASIMD2 intrinsics, 2.28GHz [^6] [^9] |
| Baikal-T1 (MIPS P5600)     | 4-way             | 6.4       | 1 MB                 | 3.85     | 2.00     | g++     7.3, ALT = 8, PREFETCH = 4096, MSA intrinsics, 1.2GHz [^7]          |
| Baikal-T1 (MIPS P5600)     | 4-way             | 6.4       | 1 MB                 | 3.74     | 2.09     | g++     7.3, ALT = 8, PREFETCH = 4096, MSA intrinsics, 1.2GHz [^7] [^8]     |

[^1]: Prefetch applies only to 512x512 and is tuned for the given core clock; 64x64 is not prefetched.  
[^2]: The entirety of 512x512 matrices fit in LLC; LLC runs in the clock domain of the cores on SNB & IVB, but in its own clock domain on HSW.  
[^3]: Amount of shared L2 in the 'big' cluster.  
[^4]: Small dataset (64x64) uses ALT=7, big dataset (512x512) uses ALT=6.  
[^5]: Target arch set to 32-bit A32.  
[^6]: Non-native compiler tuning -mtune=cortex-a57.  
[^7]: Large variance in the 512x512 times -- best result listed.  
[^8]: Non-native compiler tuning -mtune=mips32r5.  
[^9]: Core part of AWS EC2 instance.  
