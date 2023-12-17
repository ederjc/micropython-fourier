Single precision FFT written in ARM assembler
=============================================

V0.52 6th Oct 2019  
Author: Peter Hinch  
Requires: ARM platform with FPU (e.g. Pyboard 1.x, Pyboard D). Any firmware
version dated 2018 or later.  

# Contents

 1. [Overview](./README.md#1-overview)  
 2. [Design](./README.md#2-design)  
  2.1 [Future development](./README.md#21-future-development)  
 3. [Getting Started](./README.md#3-getting-started)  
 4. [The DFT class](./README.md#4-the-dft-class)  
  4.1 [Conversion types](./README.md#41-conversion-types)  
  4.2 [The populate function](./README.md#42-the-populate-function)  
  4.3 [The window function](./README.md#43-the-window-function)  
  4.4 [FORWARD transform](./README.md#44-forward-transform)  
  4.5 [REVERSE transform](./README.md#45-reverse-transform)  
  4.6 [POLAR transform](./README.md#46-polar-transform)  
  4.7 [DB transform](./README.md#47-db-transform)  
 5. [The DFTADC class](./README.md#5-the-dftadc-class)  
 6. [Implementation](./README.md#6-implementation)  
 7. [Note for beginners](./README.md#7-note-for-beginners)  
 8. [Performance](./README.md#8-performance)  
 9. [Whimsical observations](./README.md#9-whimsical-observations)  

# 1. Overview

The `DFT` class is intended to perform a fast discrete fourier transform on an
array of data typically received from a sensor. Apart from initialisation most
of the code is written in ARM assembler for speed. It uses the floating point
coprocessor and does not allocate heap storage: it can therefore be called from
a MicroPython interrupt handler. See [section 8](./README.md#8-performance) for
benchmark results on the various Pyboard options.

The DFT is performed using the Cooley-Tukey algorithm. This requires arrays of
length 2**N where N is an integer. The "twiddle factors" are precomputed in
Python. The algorithm performs the conversion in place to minimise RAM usage
(i.e. the results appear in the same arrays as the source data).

Inverse transforms and fast Cartesian to polar conversion are supported, as is
the use of window functions. There is an option to convert polar magnitudes to
dB.

The principal purpose of this library is for processing signals acquired from a
Pyboard ADC. Features are targeted at typical engineering applications. It can
be used with arbitrary data in other applications, but such users may also want
to consider [ulab](https://github.com/v923z/micropython-ulab.git) which is a
"micro" version of NumPy implemented as a C module.

# 2. Design

This code obsoletes my integer based converter which was written before the
inline assembler supported floating point instructions: the ARM FPU is so fast
that integer code offers no speed advantage. The use of floating point avoids
problems with scaling and loss of precision which become apparent when integers
are used for transforms with more than 256 bins.

`adc.read_timed()` may be used for data acquisition. It blocks until completion
but is designed to work up to about a 750KHz sample rate.

Conversion from Cartesian to polar is performed in assembler using an
approximation to the `math.atan2()` function. Its accuracy is of the order of
+-0.085 degrees.

Calculations use single precision floating point on all platforms.

## 2.1 Future development

Modern compilers mean that the traditional performance benefit of assembler is
nonexistent unless the assembler code is hand crafted and optimised to an
extreme level. This is because the compiler exploits details of the internal
design of the CPU in ways which are difficult for the programmer to achieve.

The benefit of the inline assembler (compared to C modules) is that the code
may be run on a standard firmware build. When dynamically loaded C modules
arrive this will no longer apply. The drawback of assembler is that it is not
portable.

I may rewrite this library as a dynamically loadable C module.

# 3. Getting Started

The first step is to determine how to populate the real data array. If you are
using the Pyboard and intend to use an onboard ADC, one option is to use the
ADC's `read_timed()` method. This can acquire data at high speed but has the
drawack of blocking for the duration of the read. Its use is supported by the
`DFTADC` class:

```python
from dftclass import DFTADC, POLAR
mydft = DFTADC(128, "X7")
 # Acquire data for 0.1 secs and convert
mydft.run(POLAR, 0.1)  # values are in mydft.re and mydft.im
```

Where the data is to be acquired by other means you will need to instantiate a
`DFT` object and provide a function to populate its `re` array. For synthetic
data this is straightforward. Data from sensors is usually in the form of
integers which will need to be converted to floats. While this is trivial in
Python, if speed is critical the `window.icopy` function can copy and convert
an integer array to one of floats (see the `DFTADC` class). The test programs
`dftadc.py`, `dftadc_tests.py` and `dfttest.py` provide examples, the latter
showing the use of synthetic data.

File | Purpose |
-----|-------- |
dftadc.py   | Demo program using the DAC to generate analog data passed to the ADC. |
dftadc_tests.py | Further ADC demos showing window function etc. |
dfttest.py  | Demo with synthetic data. |
dft.py      | The fft implementation. |
dftclass.py | Python interface. Requires `polar.py`, `window.py`, `dft.py`. |
window.py   | Assembler code to initialise an array and to multiply two 1D arrays. |
polar.py    | Cartesian to polar conversion. Includes fast atan2 approximation. |
ctrlmap.ods | Describes structure of the control array. |
algorithms.py | Pure Python DFT used as basis for asm code. |
dftbench.py | Benchmark times a 1024-point forward transform. |

Test programs `dftadc.py` and `dfttest.py` provide means of demonstrating the
code with ADC and synthetic data respectively. `dftadc_tests.py` also
illustrates the use of a window function. For ease of reading the test programs
print phase angles in degrees.

Test programs require `dft.py`, `dftclass.py`, `polar.py`, and `window.py`.
Note that `dft.py` cannot be frozen as bytecode because of it use of assembler.

###### [Top](./README.md#contents)

# 4. The DFT class

This is the interface to the conversion. The constructor takes the following
arguments:  
 1. `length` Mandatory. Integer. The conversion length. Must be an integer
 power of 2.
 2. `popfunc=None` An optional function to populate the real array.
 3. `winfunc=None` An optional window function.

Method:  
 * `run` Mandatory arg: `conversion`. Specifies the conversion type. See below.
 Returns the time in μs taken by the raw conversion.

Properties:  
 * `scale` Integer. Read/write. The consructor initialises this with the
 default scaling factor `1/length`. This may be modified prior to executing
 `run()`.
 * `length` Integer. Read only. The transform length.

User-accessible bound variables:  
 * `re` Real data array. Elements are of type `float`.
 * `im` Imaginary data array. Elements are of type `float`.
 * `dboffset` Float. Offset for dB conversion. Default 0. See section 4.7.

## 4.1 Conversion types

These constants in `dftclass.py` are passed to `DFT.run()` and define the
conversion to be performed. The following are the options, described in detail
below:

Option | Result |
-------|------- |
FORWARD | Normal forward transform. See 4.4 below. |
REVERSE | Perform a reverse transform. See 4.5 below. |
POLAR | Forward transform with results as polar coordinates. See 4.6. |
DB | As per POLAR but magnitude is converted to dB. See 4.7. |

## 4.2 The populate function

This optional function is called each time `run` is executed. Its purpose is to
populate the `re` data array, possibly by accessing hardware. It receives the
`DFT` instance as its arg. Any return value is ignored. Any windowing is
applied after it returns.

## 4.3 The window function

A discussion of the purpose of window functions is outside the scope of this
document. See:  
[Mathematical background](https://en.wikipedia.org/wiki/Window_function)  
[Engineer's guide](http://www.bores.com/courses/advanced/windows/files/windows.pdf)

This optional function takes two arguments:
 * `x` Point number (0 <= number < length).
 * `length` Transform length.

It should return the window function value for the specified point. Commonly
this is in range 0-1.0. A typical window function is the Hanning (Hann) function:

```python
def hanning(x, length):
    return 0.5 - 0.5*math.cos(2*math.pi*x/(length-1))
```

This has a -6dB coherent gain which may be offset by multiplying by 2 to
preserve signal amplitude:

```python
def hanning(x, length):
    return 1 - math.cos(2*math.pi*x/(length-1))
```

## 4.4 FORWARD transform

Forward transforms assume real data: you only need to populate the real array.
The imaginary array is zeroed by `DFT.run()` before a conversion is performed.
By default values are scaled by the transform length to produce mathematically
correct values. The `forward()` function in `dfttest.py` provides an example of
this.

The result comprises complex data in the DFT object's `re` and `im` arrays.

## 4.5 REVERSE transform

These accept complex data in the DFT object's `re` and `im` arrays. If you use
a `populate()` function it must initialise both arrays. The `trev()` function
in `dfttest.py` provides an example of this.

The conversion result comprises complex data in the DFT object's `re` and `im`
arrays.

## 4.6 POLAR transform

This is a forward transform with results converted to polar coordinates.

On completion the magnitude is in the DFT object's `re` array and the phase is
in `im`. Phase is in radians with the same conventions as `math.atan2()`. The
`test()` function in `dfttest.py` provides an example of this.

For performance only the first half of `re` and `im` arrays are converted. The
complex conjugates are ignored.

## 4.7 DB transform

This is a forward transform with results converted to polar coordinates. The
magnitude is converted to dB. The 0dB level defaults to 1VRMS. Magnitudes are
scaled by subtracting the `dboffset` bound variable. Magnitudes <= 0.0 are
returned as -80dB. The `dbtest()` function in `dfttest.py` provides an example
of this.

On completion the magnitude is in the DFT object's `re` array and the phase is
in `im`. Phase is in radians in a form compatible with `math.atan2()`.

For performance only the first half of `re` and `im` arrays are converted. The
complex conjugates are ignored.

As noted above the 0dB reference voltage is determined by the bound variable
`DFTADC.dboffset`. An explanation of the calculation of its value may be found
in comments in `dftclass.py`. The value may be changed prior to performing a DB
transform to change the reference voltage.

###### [Top](./README.md#contents)

# 5 The DFTADC class

This supports input from a Pyboard ADC using `pyb.Timer.read_timed`. Its base
class is `DFT`.

Costructor. This takes the following args:
 1. `length` Mandatory. Integer defining transform length.
 2. `adcpin` Mandatory. This may take an ADC instance or an object capable of
 defining one e.g. `'X7'` or `pyb.Pin.board.X19`.
 3. `winfunc=None` Window function. See section 4.3.
 4. `timer=6` Can take a `pyb.Timer` instance or a timer no. Defines the timer
 used for data acquisition.

The constructor sets the `dboffset` bound variable so that the scaling is such
that 0dB corresponds to a 1V RMS sinewave applied to the Pyboard ADC (with
suitable DC bias). This only affects `DB` conversions.

Method.  
 * `run` Mandatory args: `conversion`, `duration`.
 Returns the time in μs taken by the conversion from the time of completion of
 data acquisition to the completion of conversion.

`conversion` must be one of the forward conversion types defined in 
[section 4.1](./README.md#41-conversion-types).  
`duration` Integer or float. Acquisition duration in seconds.

`run` will block for the duration.

In the case of `DB` conversions scaling may be modified by altering the
`dboffset` bound variable.

# 6. Implementation

The DFT constructor creates and initialises three member float arrays, `re`,
`im`, and `cmplx` and an integer array `ctrl`. The first two store the real
and imaginary parts of the input and output data: for a 256 bin transform
each will use 1KB of RAM. The `ctrl` and `cmplx` arrays are small (total size
of the order of 120 bytes, size of the latter varies slightly with transform
length) and contains data used by the transform itself, including a one-off
calculation of the roots of unity (twiddle factors). There is no need to
access this data, but for those wishing to understand the code the structure of
the `ctrl` and `cmplx` arrays is documented in `ctrlmap.ods`.

The constructor is pure Python as it is assumed that the speed of
initialisation is not critical. The `run()` member function which performs the
transform uses assembler for iterative routines in an attempt to optimise
performance. The one exception is dB conversion of the result which is in
Python.

###### [Top](./README.md#contents)

# 7. Note for beginners

This README does assume some familiarity with sampling theory and the DFT. It
is worth noting that, in any sampled data system, precautions need to be taken
to prevent a phenomenon known as aliasing. If you read the ADC at 1mS
intervals, the maximum frequency which can be extracted from the set of samples
is 500Hz. If signals above this frequency are present in the input analog
signal, these will incorrectly appear as signals below this frequency. This is
a fundamental property of all sampled data systems and you need to ensure that
such signals are removed. Typically this is performed by a combination of
analog and digital filtering.

Assume you read the ADC over one second and do a 1024 point DFT. Samples will
be acquired at a rate of 1.024KHz. However as described above the maximum
frequency which can be acquired at that rate is 512Hz. The output data from the
conversion occupies 1024 frequency "bins". bin[0] contains the DC component.
bin[1] contains the 1Hz component up to bin[511] with 511Hz.

Bins from 1023 downwards contain complex conjugates of the lower bins, so
bin[1023] contains 1Hz conjugate, bin[1022] 2Hz and so on. Frequencies are
represented as two contra-rotating phasors with the same magnitude but opposite
phase (complex conjugates) which add to produce a real voltage.

As such these higher bins contain no information and can be ignored: simply
double the absolute value of the lower order bins to retrieve the voltage.

# 8. Performance

The script `dftbench.py` times a 1024 point forward transform in a way that
mimics a typical application. In such an application the `DFTADC` would be
instantiated at the start. Data would be acquired repeatedly from an ADC at an
application dependent rate. Critical timing is from the end of data acquisition
to the availability of transform data. This is the interval that `dftbench`
measures. Results were:

Board | Time (ms) |
|-----|-----------|
| Pyboard 1.x | 12.9 |
| Pyboard D SF2W |  3.6 |
| Pyboard D SF6W |  3.6 |

# 9. Whimsical observations

At one time a 1024 point DFT was widely used as a computer benchmark. As such
they were implemented in highly optimised assembler. I can't make this claim:
my code could be significantly improved. It does it in 12.9mS on a Pyboard. It
costs £28.

One of the first supercomputers, a Cray 1, took 9mS. It cost a king's ransom.

My own introduction to DFT involved punching cards, handing them in to the
computer operator, and retrieving a listing (often with only an error code)
the following day...

###### [Top](./README.md#contents)
