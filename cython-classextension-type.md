# Cython: Class and Extension Type

## Comparing Python Classes and Extension Types

* In Python everything is an object. An Object has identity, value, and type.

  * Identity: distinguishes it from all others and is provided by the id built-in function

  * Value: data associated with it, accessible via dot notation.  Python places an object’s data inside an internal instance dictionary named `__dict__`.

  * Type: specifies the behaviors that an object of that type exhibits. These behaviors are accessible via special functions, called **methods**.

* A type is responsible for creating and destroying its objects, initializing them, and up‐ dating their values when methods are called on the object. **Python allows us to create new types, in Python code, with the **`class`** statement**.

* We can also create our own types at the C level directly using the Python/C API; these  
   are known as **extension types**. The extension type has fast C-level access to the type’s methods and the instance’s data.  **Cython makes creating and using extension types as straightforward as working with pure-Python classes**. Extension types are created in Cython with the `cdef class` statement.

## Extension Types in Cython

```py
cdef class Particle:
    """Simple Particle extension type."""
    cdef double mass, position, velocity
    def __init__(self, m, p, v):
        self.mass = m
        self.position = p
        self.velocity = v
    def get_momentum(self):
        return self.mass * self.velocity
```

* The generated code uses the \_Python/C API \_heavily and makes the same calls that the interpreter would if this class were defined in pure Python.
* The `cdef` type declarations in the class body **are not class-level attributes**. They are C-level instance attributes; this style of attribute declaration is similar to languages like C++ and Java.** All instance attributes must be declared with **`cdef`**at the class level in this way for extension types. **
* For an object of a regular Python class, its underlying dictionary is modifiable and open to new key/value pairs, Extension type attributes on the other hand are **private** by default, and are accessible by the methods of the class. 
* Cython will translate any accesses like `self.mass`or`self.velocity` into low-level accesses to **C-struct fields**. This _bypasses the general lookup process for pure- Python classes._

## Type Attributes and Access Control

```py
cdef class Particle:
    """Simple Particle extension type."""
    cdef public double mass
    cdef readonly double position
    cdef double velocity
```

* When calling the `get_momentum` method, Cython still uses fast C-level direct access, and extension type methods essentially ignore the`readonly` and `public` declarations. **These exist only to allow and control access from Python**.

## C-Level Initialization and Finalization

```py
cdef class Matrix:
    cdef:
        unsigned int nrows, ncols
        double *_matrix
    def __cinit__(self, nr, nc):
        self.nrows = nr
        self.ncols = nc
        self._matrix = <double*>malloc(nr * nc * sizeof(double))
        if self._matrix == NULL:
            raise MemoryError()
    def __dealloc__(self):
        if self._matrix != NULL:
            free(self._matrix)
```

* Cython adds a special method named `__cinit__` whose responsibility is to perform C- level allocation and initialization.

* Cython guarantees that`__cinit__`is called** exactly once** and that it is** called before**`__init__, __new__`, or alternative Python-level constructors. Cython also supports C-level finalization through the  
   \_\_dealloc\_\_ special method.

## cdef and cpdef Methods

```py
cdef class Particle:
    """Simple Particle extension type."""
    cdef double mass, position, velocity
    # ...
    def add_momentums(self)      
    cpdef add_momentums_typed (self):
    cdef add_momentums_typed_c (self):


def add_momentums(particles):
    """Returns the sum of the particle momentums."""
    total_mom = 0.0
    for particle in particles:
        total_mom += particle.get_momentum()
    return total_mom

cpdef add_momentums_typed(list particles):
    """Returns the sum of the particle momentums."""
    cdef:
        double total_mom = 0.0
        Particle particle
    for particle in particles:
        total_mom += particle.get_momentum()
    return total_mom
cdef add_momentums_typed_c(list particles):
    """Returns the sum of the particle momentums."""
    cdef:
        double total_mom = 0.0
        Particle particle
    for particle in particles:
        total_mom += particle.get_momentum_c()
    return total_mom
```

* The basic understanding about `def, cdef`, and `cpdef` functions also apply to extension type methods. 
* The last version has the best performance: approximately 4.6 microseconds, another 40 percent boost over the add\_momentums\_typed version. The downside is that `get_momentum_c` is not callable from Python code, only Cython. 
* Because a`cdef` method is not accessible or overrideable from Python, it does not have to cross the language boundary, so it has less call overhead than a `cpdef`equivalent. **This is a relevant concern only for small functions where call overhead is non-negligible.**

## Inheritance and Subclassing

```py
cdef class CParticle(Particle):
    cdef double momentum
    def __init__(self, m, p, v):
        super(CParticle, self).__init__(m, p, v)
        self.momentum = self.mass * self.velocity
    cpdef double get_momentum(self):
        return self.momentum

class PyParticle(Particle):
    def __init__(self, m, p, v):
        super(PyParticle, self).__init__(m, p, v)
    def get_momentum(self):
        return super(PyParticle, self).get_momentum()
```

* An extension type can subclass **a single base type**, and that base type must itself be a type implemented in C—either a built-in type or another extension type.
* We can also subclass `Particle` in pure Python as well. The `PyParticle` class cannot access **any private C-level attributes or **`cdef`** methods**. It can override `def` and `cpdef` methods defined on `Particle.`

### Casting and Subclasses

```py
cdef Particle static_p = p
print static_p.get_momentum()
print static_p.velocity

print (<Particle?>p).get_momentum()
print (<Particle?>p).velocity
```

* When working with a dynamically typed object, Cython cannot access any C-level data or methods on it. **All attribute lookup must be done via the Python/C API**, which is slow. If we know the dynamic variable is or may possibly be an instance of a built-in type or an extension type, then it is worth** casting to the static type**. Further, Cython can also access Python-level attributes and `cpdef` methods directly without going through the Python/C API. 
* Two ways to perform this casting: either by creating a statically typed variable of the desired type and assigning the dynamic variable to it, or by using Cython’s casting operator.
* The casting method removes the need to create a temporary variable. The cast is enclosed in parentheses due to Cython’s precedence rules. Note checked   cast calls into the Python/C API will incurs runtime overhead.

### Extension Type Objects and None

```py
def dispatch(Particle p not None):
    print p.get_momentum()
    print p.velocity

# cython: nonecheck=True
$ cython --directive nonecheck=True source.pyx
```

* Cython treats `None` specially—even though it is not an instance of Particle, Cython allows it to be passed in as if it were. This is analogous to the NULL pointer in C. However, `None` object essentially has no C interface, so trying to call a method on it or access an attribute is not valid. To make these operations safe, we could check if it is `None` first.
* Cython also provides a  nonecheck compiler directive.

## Extension Type Properties in Cython

```py
class Particle(object):
    # ...
    def _get_momentum(self):
        return self.mass * self.velocity
    momentum = property(_get_momentum)

cdef class Particle:
    """Simple Particle extension type."""
    cdef double mass, position, velocity
    # ...
    property momentum:
        """The momentum Particle property."""
        def __get__(self):
            """momentum's getter"""
            return self.mass * self.velocity
        def __set__(self, m):
            """momentum's setter"""
            self.velocity = m / self.mass
```

* Python` properties` are handy and very powerful, allowing** precise control over attribute access and on-the-fly computation**. Python programmer would berate using a getter method like previous; the right way to do it is to e**ither expose momentum directly or make a property instead.**
* Cython has different syntax for extension type properties, but it achieves the same end

## Special Methods Are Even More Special

```py
cdef class E:
    """Extension type that supports addition."""
    cdef int data
        def __init__(self, d):
        self.data = d
    def __add__(x, y):
        # Regular __add__ behavior
        if isinstance(x, E):
            if isinstance(y, int):
                return (<E>x).data + y
        # __radd__ behavior
        elif isinstance(y, E):
            if isinstance(x, int):
                return (<E>y).data + x
        else:
            return NotImplemented

from cpython.object cimport Py_LT, Py_LE, Py_EQ, Py_GE, Py_GT, Py_NE
cdef class R:
    """Extension type that supports rich comparisons."""
    cdef double data
    def __init__(self, d):
        self.data = d
    def __richcmp__(x, y, int op):
        cdef:
            R r
            double data
        # Make r always refer to the R instance.
        r, y = (x, y) if isinstance(x, R) else (y, x)
        data = r.data
        if op == Py_LT:
            return data < y
        elif op == Py_LE:
            return data <= y
        elif op == Py_EQ:
            return data == y
        elif op == Py_NE:
            return data != y
        elif op == Py_GT:
            return data > y
        elif op == Py_GE:
            return data >= y
        else:
           assert False
 
 cdef class I:
    cdef:
        list data
        int i
    def __init__(self):
        self.data = range(100)
        self.i = 0
    def __iter__(self):
        return self
    def __next__(self):
        if self.i >= len(self.data):
            raise StopIteration()
        ret = self.data[self.i]
        self.i += 1
        return ret
```

* When providing support for operator overloading with a Cython extension type, we have to define a special method; that is, a method of a specific name with leading and trailing double underscores.
* To support the in-place `+` operator for a pure-Python class C, we define an `__add__(self, other)` special method. The operation `c + d` is transformed into` C.__add__(c, d)` when c is an instance of the C class. If C does not know how to add itself to the other argument, then it returns NotImplemented. In this case, the Python interpreter then calls `type(d).__radd__(d, c) `to give d’s class a chance to add itself to a C instance.
* For extension types, the situation is different. Extension types do not support `__radd__`; instead, they \(effectively\) overload `__add__ `to do the job of both the regular` __add__ `and `__radd__` in one special method.
* Cython extension types do not support the individual comparison special methods like` __eq__`, `__lt__`, and `__le__`. Instead, Cython provides a single \(some would say cryptic\) method,` __richcmp__(x, y, op).`
* To make an extension type iterable, we define` __iter__` on it, just as in regular Python. To make an extension type an iterator, we define a `__next__` special method on it, as we would in Python 3. This is different from a pure-Python object, where we would define a next method instead. Cython will expose `__next__` as next to Python.



