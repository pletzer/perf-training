---
layout: post
title: Calling C/C++ extensions with ctypes
permalink: /python-scatter/ctypes
chapter: python-scatter
---

## Objectives

You will learn:

* how to call C/C++ compiled code from Python using the `ctypes` module
* how to compile a C++ extension using `setuptools`

We'll use the code in directory `cext`. Start by
```
cd cext
```

## Why extend Python with C/C++

 1. you want to call a function that is implemented in C, C++ or Fortran. This gives you access to a vast collection of libraries so you won't have re-invent the wheel.
 2. you have identified a performance bottleneck - reimplementing some parts of your Python code in C, C++ or Fortran will give you a performance boost
 3. make your code type safe. In contrast to C, C++ and Fortran, Python is not a typed language - you can pass any object to any Python function.  This can cause runtime failures in Python which cannot occur in C, C++ or Fortran, as the error would be caught by the compiler. 

### Pros

 * a good way to glue Python with an external library, no need to compile interface code
 * can be used to incrementally migrate code to C/C++
 * very flexible, you can pass most priitive types, array types and struct's from Python to C and back
 * simpler and easier to maintain than custom C extensions 

### Cons

 * has a learning curve, one must understand how Python and C work
 * mistakes often lead to segmentation faults, which can be hard to debug

## Learn the basics 

As an example, we'll assume that you have to compute the sum of all the elements of an array. Let's assume you have written a C++ extension for that purpose
```cpp
/** 
 * Compute the sum an array
 * @param n number of elements
 * @param array input array
 * @return sum
 */
extern "C"
double mysum(const int n, double* array) {
    double res = 0;
    for (int i = 0; i < n; ++i) {
        res += array[i];
    }
    return res;
}
```
in file *mysum.cpp*. The `extern "C"` line ensures that function "mysum" can be called from "C".

The easiest way to compile *mysum.cpp* is to write a *setup.py* file, which lists the source file to compile:
```python
from setuptools import setup, Extension

# Compile mysum.cpp into a shared library 
setup(
    #...
    ext_modules=[Extension('mysum', ['mysum.cpp'],),],
)
```
You might have to add include directories and libraries if your C++ extension depends on external packages. 
An example of a `setup.py` file can be found [here](https://raw.githubusercontent.com/pletzer/scatter/master/cext/setup.py). 

Calling 
```
python setup.py build
```
will compile the code and produce a shared library under *build/lib.linux-x86_64-3.6*, something like *mysum.cpython-36m-x86_64-linux-gnu.so*.

The extension *.so* indicates that the above is a shared library (also called dynamic-link library or shared object). The advantage of creating a shared library over a static library is that in the former the Python interpreter needs not be recompiled. The good news is that `setuptools` knows how to compile shared library so you won't have to worry about the details.

To call  `mysum` from Python we'll use the `ctypes` module. The steps are described below:

 1. use function `CDLL` to open the shared library. `CDLL` expects the path to the shared library and returns a shared library object.
 2. tell the argument and result types of the function. The argument types are listed in members `argtypes` (a list) and `restype`, respectively. Use for instance `ctypes.c_int` for a C `int`. See table below to find out how to translate other C types to their corresponding `ctypes` objects.
 3. call the function, casting the Python objects into ctypes objects. For instance, to pass `double` 1.2, call the function with argument `ctypes.c_double(1.2)`.

### Translation of some Python types into objects that can be passed to a C/C++ function

The following summarises the translation between Python and C for some common data types: 

| Python casting                            | C type            | Comments                                      |
|-------------------------------------------|-------------------|-----------------------------------------------|
| `None`                                    | `NULL`            |                                               |
| `str(...).encode('ascii')`                | `char*`           |                                               |
| `ctypes.c_int(...)`                       | `int`             | No need to cast                               |
| `ctypes.c_double(...)`                    | `double`          |                                               |
| `(...).ctypes.POINTER(ctypes.c_double)`   | `double*`         | pass a numpy array of type float64            |
| `ctypes.byref(...)`                       | `&`               | pass by reference (suitable for arguments returning results)                             |      

For a complete list of C to ctypes type mapping see the Python [documentation](https://docs.python.org/3/library/ctypes.html).

### Passing NumPy arrays to `ctypes` functions

An alternative to using the Python casting from the above table every time you want to pass an array to a C/C++ function (i.e. `(...).ctypes.POINTER(ctypes.c_double)`), is to use `numpy.ctypeslib.ndpointer` in the `argtypes` list to specify that an array should be passed to the function:

```python
# define the interface (this dummy function takes 1 argument, a float64 array)
mylib.myfunction.argtypes = [np.ctypeslib.ndpointer(dtype=np.float64)]

# call the function (no casting required)
myarray = np.zeros(100, np.float64)
mylib.myfunction(myarray)
```

With this approach it is possible to specify extra restrictions on the NumPy arrays at the interface level, for example the number of dimensions the array should have or its shape. If an array passed in as an argument does not meet the specified requirements and exception will be raised. A full list of possible options can be found in the `numpy.ctypeslib.ndpointer` [documentation](https://docs.scipy.org/doc/numpy-1.15.0/reference/routines.ctypeslib.html#numpy.ctypeslib.ndpointer).

### Working example

Let's return to our `mysum` C++ function, which we would like to call from Python:
```python
import ctypes
import numpy
import glob

# find the shared library, the path depends on the platform and Python version
libfile = glob.glob('build/*/*.so')[0]

# 1. open the shared library
mylib = ctypes.CDLL(libfile)

# 2. tell Python the argument and result types of function mysum
mylib.mysum.restype = ctypes.c_double
mylib.mysum.argtypes = [ctypes.c_int, numpy.ctypeslib.ndpointer(dtype=numpy.float64)]

array = numpy.linspace(0., 1., 100000)

# 3. call function mysum
array_sum = mylib.mysum(len(array), array)

print('sum of array: {}'.format(array_sum))
```

### Additional explanation

By default, arguments are passed by value. To pass an array of doubles (`double*`), specify `numpy.ctypeslib.ndpointer(dtype=numpy.float)` in the `argtypes` list. You can declare `int*` similarly by using `numpy.ctypeslib.ndpointer(dtypes=numpy.int)`. Then the numpy arrays can be passed directly to the function with no other casting required.

Strings will need to be converted to byte strings in Python 3 (`str(mystring).encode('ascii')`).

Passing by reference, for instance `int&` can be achieved using `ctypes.byref(myvar_t)` with `myvar_t` of type `ctypes.c_int`. 

The C type `NULL` will map to None.


## Exercises

We've created a version of `scatter.py` that builds and calls a C++ external function *src/wave.cpp*. Compile the code using
```
python setup.py build
```
(Make sure you have the `BOOST_DIR` environment set as described [here.](https://nesi.github.io/perf-training/python-scatter/introduction))

 1. profile the code and compare the timings with those obtained by running *scatter.py* under `original/` and `vect/`
 2. rewrite Python function `isInsideContour` defined in `scatter.py` in C++ and update file `setup.py` to compile your extension. 
 3. profile the code with your version of `isInsideContour` written in C++


