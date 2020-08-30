---
layout: post
title: "easyRNG: random number generators and distributions for C and Fortran"
date: 2016-08-28 16:14:02 +0100
comments: true
categories: [c++, Fortran, random number generators]
---

The C++ 2011 standard introduced a collection of templates, grouped together in the `<random>` header of the C++ standard library, with the goal of providing a number of random number generators (RNGs) as well as several commonly used random number distributions (RNDs) to sample from. Though reception appears mixed, at least they provided a welcome alternative to the very unreliable `rand` that ships with the standard C library.

However, this does not mean anything to those who are writing code in C and Fortran, and have no plans to switch to C++ in order to obtain RNG and RND functionality. Personally, I have been using for many years the RNGs and RNDs offered by the [GNU Scientific Library (GSL)](https://www.gnu.org/software/gsl/manual/html_node/) and its [Fortran bindings FGSL](https://github.com/reinh-bader/fgsl) (of which I am a major contributor).
I have found GSL and FGSL to offer both an elegant interface as well as excellent performance on all major platforms and architectures. A potential drawback for their adoption however, are their license: the [GNU General Public License](https://www.gnu.org/licenses/gpl.html), which makes it impossible to include into proprietary, closed source software, though this is no concern for me personally as all my projects are open source.

This ended up giving me the idea for a new open source project *easyRNG*, a thin wrapper around C++11's `<random>` templates, with an API inspired by GSL and FGSL. The name for the project was obvious actually:

* Easy to implement and maintain: I didn't have to write the random number generations and distributions myself (which is really, really hard)
* Easy to use: based on the established and popular APIs offered by GSL and FGSL, which should feel familiar to everybody (see [examples](#Examples))
* Easy to build and install: no dependencies required apart from a C++11 compliant compiler, and the C++ standard library to link against. Those in need of the Fortran bindings will have to install a Fortran compiler with full 2003 and partial 2008 support. GNU autotools was used for the installation script so Windows users will have to install a suitable shell (e.g. [msys2](http://msys2.github.io)) to build easyRNG. It shouldn't be hard to get it working in Visual Studio though.
* Easy to redistribute: I picked the [3-clause BSD](https://en.wikipedia.org/wiki/BSD_licenses#3-clause_license_.28.22Revised_BSD_License.22.2C_.22New_BSD_License.22.2C_or_.22Modified_BSD_License.22.29) license, which means it can be included (even with modifications) into proprietary software, provided the original copyright notices  and disclaimer 

After a couple of weeks of work I launched easyRNG on [Github](https://github.com/tschoonj/easyRNG), where one can download the code, browse the docs and of course fork the code!

<!-- more -->

## Examples

How to use easyRNG in your C/C++/Objective-C and Fortran code (compilation instructions can be found in the [docs](https://tschoonj.github.io/easyRNG/usage.html)):

``` c

#include <easy_rnh.h>
#include <easy_randist.h>
#include <time.h>

int main(int argc, char *argv[]) {
	// create a Mersenne Twister RNG 
	easy_rng *rng = easy_rng_alloc(easy_rng_mt19937);

	// seed with the time to get a unique sequence every time this program is run
	easy_rng_set(rng, (long unsigned int) time(NULL));

	// get a double precision real number in [0, 1[
	double val = easy_rng_uniform(rng);

	// sample a double precision real number from a gaussian distribution with standard deviation 5.0
	val = easy_ran_gaussian(rng, 5.0);

	// free the RNG
	easy_rng_free(rng);

	return 0;
}

```

``` fortran

PROGRAM test
  USE, INTRINSIC :: ISO_C_BINDING
  USE :: easyRNG
  IMPLICIT NONE

  TYPE (easy_rng) :: rng
  REAL (C_DOUBLE) :: val

  ! interface for libc's time function
  INTERFACE
    FUNCTION easy_time(timer) BIND(C, NAME='time') RESULT(rv)
      USE, INTRINSIC :: ISO_C_BINDING
      IMPLICIT NONE
      TYPE (C_PTR), INTENT(IN), VALUE :: timer
      INTEGER (C_LONG) :: rv
    END FUNCTION easy_time
  END INTERFACE

  ! create a Mersenne Twister RNG 
  rng = easy_rng_alloc(easy_rng_mt19937)

  ! seed with the time to get a unique sequence every time this program is run
  CALL easy_rng_set(rng, easy_time(C_NULL_PTR))

  ! get a double precision real number in [0, 1[
  val = easy_rng_uniform(rng)

  ! sample a double precision real number from a gaussian distribution with standard deviation 5.0
  val = easy_ran_gaussian(rng, 5.0)

  ! free the RNG
  CALL easy_rng_free(rng)
END PROGRAM
```

easyRNG is also thread-safe, provided each thread has its either its own unique `easy_rng` instance, or alternatively if locking is used to ensure only one thread can use the RNG at a time (not recommended).

## Checking the correctness of the results

The correctness of the random number distributions was verified by sampling large numbers of random numbers while calculating the running average and standard deviation of the generated numbers, followed by comparing the average and standard deviation with the theoretically predicted values. These were obtained using equations from the descriptions of the distributions on Wikipedia. A notable exception here is the Cauchy (AKA Lorentz) distribution for which no average and standard deviation is defined.

I added support to the easyRNG repository for both [Travis-CI](https://travis-ci.org/tschoonj/easyRNG) (Linux and Mac OS X) and [Appveyor](https://ci.appveyor.com/project/tschoonj/easyrng) (Windows). For the former, I configured a genuine barrage of combinations of compiler versions on the two platforms to ensure I could test easyRNG with as many versions of GCC and clang as possible.

This resulted in the identification of a version of clang that consistently produced wrong results: version 6.0, which shipped with Xcode 6.1 and 6.2, Apple's SDK for its Mountain Lion release. I did not manage to identify for which random number distributions they failed though.

One observation I made while running the test is that the F-distribution's standard deviation is varying rather wildly around the theoretical value. Since this appears to be the case for all platforms and compilers, it looks to me like an inherent property of the distribution.


## Performance

A very important characteristic of any library providing random number generators is its performance since in many applications such as Monte Carlo simulations, huge numbers of random numbers will have to be generated. Since easyRNG merely wraps the C++11 RNGs and RNDs, their implementation in the C++ standard library will be the key factor here. In order to establish performance, I produced 100000000 uniformly distributed random integers using both easyRNG (easy_rng_get) and GSL (gsl_rng_get), for the random number generator types that (I believe) are present in both libraries, on several platforms.

In all cases, GSL was used as installed with the default package manager, meaning that it was compiled at optimization level 2 (`CXXFLAGS="-O2"`) using the default system compiler.

These comparisons should not be taken too seriously as they were not obtained under 'lab conditions', but they do reveal that the selected compiler and optimization level do impact the result, sometimes even in surprising ways...

The code to run these tests can be found in [test2.c](https://github.com/tschoonj/easyRNG/blob/master/tests/test2.c).

### Mac OS X 10.11.6, Xcode 7.3.1, clang Apple LLVM version 7.3.0 (system default), CXXFLAGS=-O2

    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.610597 s
    GSL 0.407344 s
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 6.87464 s
    GSL 9.56388 s
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 23.3074 s
    GSL 16.3843 s
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.631972 s
    GSL 0.543443 s
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.524086 s
    GSL 0.553649 s

### Mac OS X 10.11.6, Xcode 7.3.1, clang Apple LLVM version 7.3.0 (system default), CXXFLAGS=-O3

    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.601218 s
    GSL 0.413567 s
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 6.76775 s
    GSL 9.43511 s
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 23.0925 s
    GSL 16.104 s
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.624519 s
    GSL 0.545767 s
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.515577 s
    GSL 0.548885 s

### Mac OS X 10.11.6, Xcode 7.3.1, gcc 6.1.0 (Homebrew), CXXFLAGS=-O2

    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.51174 s
    GSL 0.420167 s
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 6.76126 s
    GSL 9.55059 s
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 20.765 s
    GSL 16.3402 s
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.656705 s
    GSL 0.567046 s
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.575031 s
    GSL 0.557471 s

### Mac OS X 10.11.6, Xcode 7.3.1, gcc 6.1.0 (Homebrew), CXXFLAGS=-O3

    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.458188 s
    GSL 0.408222 s
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 4.99809 s
    GSL 9.62012 s
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 16.4211 s
    GSL 16.3106 s
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.659607 s
    GSL 0.571513 s
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.575066 s
    GSL 0.565953 s

### Linux Ubuntu 16.04 Xenial, gcc 5.4.0, CXXFLAGS=-O2

    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.487993
    GSL 0.514865
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 7.00267
    GSL 4.91427
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 21.6973
    GSL 8.139
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.51359
    GSL 0.488009
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.514706
    GSL 0.515695

### Linux Ubuntu 16.04 Xenial, gcc 5.4.0, CXXFLAGS=-O3

    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.497586
    GSL 0.50192
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 5.34764
    GSL 4.81555
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 17.0988
    GSL 8.12849
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.509686
    GSL 0.492514
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.508777
    GSL 0.505121

### Linux Ubuntu 16.04 Xenial, clang 3.8.0, CXXFLAGS=-O2
    
    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.417623
    GSL 0.498354
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 3.2451
    GSL 4.82584
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 11.4369
    GSL 8.14133
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.514472
    GSL 0.499584
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.517101
    GSL 0.513164

### Linux Ubuntu 16.04 Xenial, clang 3.8.0, CXXFLAGS=-O3

    Comparing easyRNG's mt19937 with GSL's mt19937
    easyRNG 0.545296
    GSL 0.500088
    Comparing easyRNG's ranlux24 with GSL's ranlux
    easyRNG 3.24089
    GSL 4.8667
    Comparing easyRNG's ranlux48 with GSL's ranlux389
    easyRNG 11.1006
    GSL 8.17594
    Comparing easyRNG's minstd_rand0 with GSL's minstd
    easyRNG 0.517966
    GSL 0.49886
    Comparing easyRNG's minstd_rand with GSL's fishman20
    easyRNG 0.509863
    GSL 0.517385

## Conclusion

*easyRNG* is a little library that wraps the C++11 `<random>` templates for a number of popular random number generators and distributions, using an API based on the GNU Scientific Library.
It is primarily intended for people developing in C and Fortran, but it can also be used to great effect in C++ and Objective-C. I will probably also add support for Python at some point, using SWIG.

The code is hosted on [Github](https://github.com/tschoonj/easyRNG), [documentation](https://tschoonj.github.io/easyRNG/) was produced with Doxygen and the officially released tarballs can be found [here](http://lvserver.ugent.be/easyRNG/).

Feel free to comment, fork the code and open issues!
