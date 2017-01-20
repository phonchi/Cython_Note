# Compiling and Runing

## The Cython Compilation Pipeline

1. The first stage is handled by the _cython compiler,_ which transforms Cython source into optimized and platform-independent C or C++. 
2. The second stage compiles the generated C or C++ source into a **shared library** with a standard C or C++ compiler. The resulting shared library is platform dependent. It is a shared-object file with a `.so` extension on Linux or Mac OS X, and is a dynamic library with a `.pyd` extension on Windows. Some flags pass to the C or C++ compiler to ensure this shared library is a full-fledged Python module. 

> We call this compiled module an **extension module**, and it can be imported and used as if it were written in pure Python.

## Using distutils with cythonize

1. The first stage is handled by `cythonize` command which is included in cython.
2. The `distutils` package which is in the standard library of python for building, packaging, and distributing Python projects is use for the second stage.

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(ext_modules=cythonize('fib.pyx'))

$ python setup.py build_ext --inplace
```

* The cythonize command returns** a list of distutils Extension objects** that the `setup` function knows how to turn into Python extension modules.
* The optional ` --inplace` flag instructs   `distutils` to place each extension module next to its respective Cython` .pyx `source file.

```py
from distutils.core import setup, Extension
from Cython.Build import cythonize

# First create an Extension object with the appropriate name and sources.
ext = Extension(name="wrap_fib", sources=["cfib.c", "wrap_fib.pyx"])
# Use cythonize on the extension object.
setup(ext_modules=cythonize(ext))
```

* Instead of wrapping cython code, when wrapping C and C++ code, which we will cover in detail in Chapters 7 and 8, we must include other source files in the compilation step.

```py
from distutils.core import setup, Extension
from Cython.Build import cythonize

ext = Extension(name="wrap_fib",
                sources=["wrap_fib.pyx"],
                library_dirs=["/path/to/libfib.so"],
                libraries=["fib"])
setup(ext_modules=cythonize(ext))
```

If we are provided a precompiled dynamic library `libfib.so` rather than source code, we can instruct distutils to link against `libfib.so `at link time:

## Interactive Cython with IPython’s %%cython Magic

## Compiling On-the-Fly with pyximport

* It retrofits the` import `statement to recognize` .pyx `extension modules, sends them through the compilation pipeline automatically, and then imports the extension module for use.
* A `.pyxdeps` extension in the same directory as the Cython source file is used to control the dependency. It should contain a listing of files that the `.pyx` file depends on, one file per line. If any file that matches a pattern in the` .pyxdeps` file is newer than the` .pyx` file, then pyximport will recompile on import.
* That role of `.pyxbld` file is used to tell `pyximport` to compile and link several source files into one extension module.

## Rolling Our Own and Compiling by Hand

```bash
# First step
$ cython fib.pyx

# Second step
$ CFLAGS=$(python-config --cflags)
$ LDFLAGS=$(python-config --ldflags)
$ cython fib.pyx # --> outputs fib.c
$ gcc -c fib.c ${CFLAGS} # outputs fib.o
$ gcc fib.o -o fib.so -shared ${LDFLAGS} # --> outputs fib.so
```

* To compile our `fib.c` into a Python extension module, we need to first compile` fib.c` into an object file with the proper includes and compilation flags, and then compile `fib.o` into a dynamic library with the right linking flags. Fortunately, Python provides the `python-config` command-line utility to help with this process.
* It is strongly recommended to use **the same compiler that was used to compile the Python interpreter**. The` python-config `command gives back configuration flags that are **tailored to this compiler/Python version combination. **It is thus recommended to query the CPython interpreter.

## Using Cython with Other Build Systems

### CMake and Cython

### SCons and Cython

### Make and Cython

## Compiler Directives



