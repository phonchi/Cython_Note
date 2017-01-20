# Cython Basic Types/Function

Why Cython works, it can be attributed to two differences.

1. Run‐ time interpretation versus ahead-of-time compilation.
2. Dynamic versus static typing.

## Interpreted Versus Compiled Execution

* Python code is automatically compiled to Python bytecode. Bytecodes are fundamental instructions to be executed, or interpreted, by the Python virtual machine \(PVM\). 
* In fact, the Python interpreter can run compiled C code directly and transparently to the end user. This code must be compiled into a specific kind of   dynamic library known as an **extension module**.
* We can expect around a   **10 to 30 percent** speedup from converting Python code into an equivalent extension   module written by Cython.

## Dynamic Versus Static Typing

* When running a Python program, the interpreter spends most of its time figuring out what low-level operation to perform, and extracting the data to give to this low-level operation. 
* Python interpreter always has to determine the low-level operation in a completely general way, because a variable **can have any type at any time**. This is known as **dynamic dispatch.**
* Before Cython, we could only benefit from static typing** by reimplementing our Python code in C**. Cython makes it easy to keep our Python code as is and tap into C’s static type system

## Static Type Declaration with cdef

```py
def integrate(a, b, f):
    cdef int i
    cdef int N=2000
    cdef float dx, s=0.0
    dx = (b-a)/N
    for i in range(N):
        s += f(a+i*dx)
    return s * dx
```

* To statically type variables in Cython, we use the `cdef` keyword with a type and the   variable name. The static variables with C types **have C semantics. \(Except division/modulus?\)**
* The C `static` keyword is used to declare a variable whose lifetime extends _to the entire lifetime of a program_. It is not a valid Cython keyword.
* Besides above notes, Cython supports the full range of C declarations. Cython **does not limit the C-level types that we can use**, which is especially useful when we are wrap‐ ping external C libraries.

### Automatic Type Inference in Cython

```py
cimport cython

@cython.infer_types(True)
def automatic_inference():
    i = 1
    d = 2.0
    c = 3+4j
    r = i * d + c
    return r
```

* By default, Cython infers variable types only when doing so cannot change the semantics of the code.  In the code snippet, it will automatic infer that the 2.0 literal, and hence the variable d, are C doubles.\(if the decorator does not added\)
* By means of the `infer_types` compiler directive or the decorator form of  `infer_types` , we can give Cython more leeway to infer types in cases that may possibly change   semantics.
* When enabling `infer_types`, we are taking responsibility **to ensure that integer operations do not overflow** and that semantics do not change from the untyped version. 

### C Pointers in Cython

```py
cdef int *a, *b

cdef double golden_ratio
cdef double *p_double
p_double = &golden_ratio
p_double[0] = 1.618

from cython cimport operator
print operator.dereference(p_double)

cdef st_t *p_st = make_struct()
cdef int a_doubled = p_st.a + p_st.a
```

* We have to use an asterisk   with **each** variable declared.
* Dereferencing pointers in Cython is different than in C. Because the Python language already uses the `*args` and `**kwargs `syntax. Instead, **we index into the pointer at location 0** to dereference a pointer in Cython. This syntax _also works to dereference a pointer in C_, although that’s rare.
* Alternatively, we can use the `cython.operator.dereference `function-like operator to dereference a pointer. 
* Another difference is that Cython uses **dot access whether we have a nonpointer struct variable or a pointer to a struct.**

### Mixing Statically and Dynamically Typed Variables

* It allows us to use dynamic Python objects for the majority of our code base, and easily convert them into fast, statically typed analogues for the performance-critical sections.

| Python type | C type |
| :--- | :--- |
| bool | bint |
| int, long | \[unsinged\] char, \[unsinged\] short \[unsinged\] int, \[unsinged\] long, \[unsinged\] long, long |
| float | float, double, long, double |
| complex | float complex, double complex |
| bytes, str, unicode | char \*, std::string |
| dict | struct |

* The full list of correspondences between built-in Python types and Cython C types.
* When converting integral types from Python to C, Cython generates code that checks   for overflow.

### Statically Declaring Variables with a Python Type

```py
cdef list particles, modified_particles
cdef dict names_from_particles
cdef str pname
cdef set unique_particles
```

* We can also use `cdef` to statically declare variables with a built-in Python type. Since they are implemented in C and
  Cython have access to the declaration.
* In cases where Python built-in types like` int `or` float` have the same name as a Cython C type, the** C type takes precedence. **

```py
# cython: cdivision=True

cimport cython
@cython.cdivision(True)
def divides(int a, int b):
    return a / b
    
cimport cython
def remainder(int a, int b):
    with cython.cdivision(True):
        return a % b
```

* C and Python have markedly different behavior when computing the **modulus and division**, and Cython **uses Python semantics by default** for division and modulus **even when the operands are statically typed C scalars.**
*  To   obtain C semantics, we can use the compiler directive, decorator, or context manager.

### Static Typing for Speed

* A general Cython principle: the more static type information we provide, the better Cython can optimize the result. 
* `C double` and `C long` are prefered over static declare python type.

### Reference Counting and Static String Types

* One of Python’s major features is _automatic memory management_. CPython implements this via straightforward reference counting, with an automatic garbage collector that runs periodically to clean up unreachable reference cycles. Cython also implements it.

## Cython’s Three Kinds of Functions

```py
#--------------(1)-----------------
def py_fact(n):
    """Computes n!"""
    if n <= 1:
        return 1
    return n * py_fact(n - 1)


#--------------(2)-----------------
cdef long c_fact(long n):
    """Computes n!"""
    if n <= 1:
        return 1
    return n * c_fact(n - 1)
def wrap_c_fact(n):
    """Computes n!"""
    return c_fact(n)
    
#--------------(3)-----------------   
cpdef long cp_fact(long n):
    """Computes n!"""
    if n <= 1:
        return 1
    return n * cp_fact(n - 1)
```

* Because `n` is a function argument, we can omit the ` cdef` keyword.
* The primary difference being the ` long` return type in second code snippet.
* A function declared with  cdef can be called by any other function—def or  cdef,  However, Cython does not allow a cdef function to be called from external Python   code.Thus we need a wrapper.
* In the third code snippet, we get a C-only version   of the function and a Python wrapper for it, both with the same name. 
* Any Python object can be represented at the C level \(e.g., by using a dynamically typed argument, or by statically typing a built-in type\), **but not all C types can be represented in Python**. So, **we cannot use void, C pointers, or C arrays indiscriminately as the argument types or return type of `cpdef` functions**. 

```py
cpdef int divide_ints(int i, int j) except? -1:
    return i / j
```

* Cython also provides an `except` clause to allow a `cdef   `or `cpdef` function to communicate to its caller.

## Type Coercion and Casting

```py
def print_address(a):
    cdef void *v = <void*>a
    cdef long addr = <long>v
    cdef list cast_list = <list>a
    cdef list cast_list = <list?>a
```

* Cython provides a casting operator that is very similar to C’s casting operator, **except that it replaces parentheses with angle brackets**.
* When we are less than certain and want Cython to check the type before casting, we can use the checked casting operator instead.

## Declaring and Using structs, unions, and enums

```py
cdef struct mycpx:
    float real
    float imag
cdef union uu:
    int a
    short b, c
ctypedef struct mycpx:
    float real
    float imag
ctypedef union uu:
    int a
    short b, c
    
#--------------(1)-----------------
cdef mycpx a = mycpx(3.1415, -1.0)
cdef mycpx b = mycpx(real=2.718, imag=1.618034)

#--------------(2)-----------------
cdef mycpx zz
zz.real = 3.1415
zz.imag = -1.0

#--------------(3)-----------------
cdef mycpx zz = {'real': 3.1415, 'imag': -1.0}

ctypedef double real
ctypedef long integral
def displacement(real d0, real v0, real a, real t):
    """Calculates displacement under constant acceleration."""
    cdef real d = d0 + (v0 * t) + (0.5 * a * t**2)
    return d
```

* The general way that Cython blends Python with C is to use **Python-like semantic to deal with C-level constructs.**
* Three ways to initialized: Function-like syntax, individually, and using Python dictionary.
* The `ctypedef` feature is particularly useful for C++, when typedef aliases can significantly shorten long templated types.
* Cython provides three built-in **fused types** that we can use directly: `integral` ,` floating` , and `numeric` . All are accessed via the special cython namespace, which must be `cimport`  \(see Chapter 6\).

## Cython for Loops and while Loops

## The Cython Preprocessor

```py
DEF E = 2.718281828459045
DEF PI = 3.141592653589793
def feynmans_jewel():
    """Returns e**(i*pi) + 1.  Should be ~0.0"""
    return E ** (1j * PI) + 1.0

IF UNAME_SYSNAME == "Windows":
    # ...Windows-specific code...
ELIF UNAME_SYSNAME == "Darwin":
    # ...Mac-specific code...
ELIF UNAME_SYSNAME == "Linux":
    # ...Linux-specific code...
ELSE:
    # ...other OS...
```

* Cython has a` DEF` keyword that creates a macro, which is like `#define,` it also supports conditional compilation.
* By default, Cython assumes the source language version \(the version of Python in   the `.pyx `or `.py` file\) uses Python 2 syntax and semantics.



