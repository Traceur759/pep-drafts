PEP-C


PEP: XXX
Title: Module State Access from C Extension Methods
Version: $Revision$
Last-Modified: $Date$
Author: Petr Viktorin <encukou@gmail.com>,
        Nick Coghlan <ncoghlan@gmail.com>,
        Eric Snow <ericsnowcurrently@gmail.com>
        Marcel Plch <gmarcel.plch@gmail.com>
Discussions-To: import-sig@python.org
Status: Active
Type: Process
Content-Type: text/x-rst
Created: 02-Jun-2016
Python-Version: 3.8
Post-History:


Abstract
========

This PEP proposes to add a way for CPython extension methods to access context such as
the state of the modules they are defined in.

This will allow extension methods to use direct pointer dereferences
rather than PyState_FindModule for looking up module state, reducing or eliminating the
performance cost of using module-scoped state over process global state.

This fixes one of the remaining roadblocks for adoption of PEP 3121 (Extension
module initialization and finalization) and PEP 489
(Multi-phase extension module initialization).

Additionaly, support for easier creation of immutable exception classes is added.
This removes the need for keeping per-module state if it would only be used
for exception classes.

While this PEP takes an additional step towards fully solving the problems that PEP 3121 and PEP 489 started
tackling, it does not attempt to resolve *all* remaining concerns. In particular, accessing the module state from slot methods (``nb_add``, etc) remains slower than accessing that state from other extension methods.


Terminology
===========

Process-Global State
--------------------

C-level static variables. Since this is very low-level
memory storage, it must be managed carefully.

(XXX Use this term everywhere)

Per-module State
----------------

State local to a module object, allocated dynamically as part of a
module object's initialization. This isolates the state from other
instances of the module (including those in other subinterpreters).

Accessed by ``PyModule_GetState()``.


Static Type
-----------

A type object defined as a C-level static variable, i.e. a compiled-in type object.

A static type needs to be shared between module instances and has no
information of what module it belongs to.
Static types do not have ``__dict__`` (although their instances might).

Heap Type
---------

A type object created at run time.


Rationale
=========

PEP 489 introduced a new way to initialize extension modules, which brings
several advantages to extensions that implement it:

    * The extension modules behave more like their Python counterparts.
    * The extension modules can easily support loading into pre-existing
      module objects, which paves the way for extension module support for
      ``runpy`` or for systems that enable extension module reloading.
    * Loading multiple modules from the same extension is possible, which
      makes testing module isolation (a key feature for proper sub-interpreter
      support) possible from a single interpreter.

The biggest hurdle for adoption of PEP 489 is allowing access to module state
from methods of extension types.
Currently, the way to access this state from extension methods is by looking up the module via
``PyState_FindModule`` (in contrast to module level functions in extension modules, which
receive a module reference as an argument).
However, ``PyState_FindModule`` queries the thread-local state, making it relatively
costly compared to C level process global access and consequently deterring module authors from using it.

Also, ``PyState_FindModule`` relies on the assumption that in each
subinterpreter, there is at most one module corresponding to
a given ``PyModuleDef``.  This does not align well with Python's import
machinery.  Since PEP 489 aimed to fix that,  the assumption does
not hold for modules that use multi-phase initialization, so
``PyState_FindModule`` is unavailable for these modules.

A faster, safer way of accessing module-level state from extension methods
is needed.


Immutable Exception Types
-------------------------

For isolated modules to work, any class whose methods touch module state
must be a heap type, so that each instance of a module can have its own
type object.  With the changes proposed in this PEP, heap type instances will
have access to module state without global registration.  But, to create
instances of heap types, one will need the module state in order to
get the type object corresponding to the appropriate module.
In short, heap types are "viral" – anything that “touches” them must itself be
a heap type.

Curently, most exception types, apart from the ones in ``builtins``, are
heap types.  This is likely simply because there is a convenient way
to create them: ``PyErr_NewException``. 
Heap types generally have a mutable ``__dict__``.
In most cases, this mutability is harmful.  For example, exception types
from the ``sqlite`` module are mutable and shared across subinterpreters.
This allows "smuggling" values to other subinterpreters via attributes of
``sqlite3.Error``.

Moreover, since raising exceptions is a common operation, and heap types
will be "viral", ``PyErr_NewException`` will tend to "infect" the module
with "heap-type-ness" – at least if the module decides play well with
subinterpreters/isolation.
Many modules could go without module state
entirely if the exception classes were immutable.

To solve this problem, a new function for creating immutable exception types
is proposed.


Background
===========

The implementation of a Python method may need access to one or more of
the following pieces of information:

   * The instance it is called on (``self``)
   * The underlying function
   * The class the method was defined in
   * The corresponding module
   * The module state

In Python code, the Python-level equivalents may be retrieved as::

    import sys

    class Foo:
        def meth(self):
            instance = self
            module_globals = globals()
            module_object = sys.modules[__name__]
            underlying_function = Foo.meth
            defining_class = Foo

.. note::

    The defining class is not ``type(self)``, since ``type(self)`` might
    be a subclass of ``Foo``.

Implicitly, the last three of those rely on name-based lookup via the function's ``__globals__``:
either the ``Foo`` attribute to access the defining class and Python function object, or ``__name__`` to find the module object in ``sys.modules``.
In Python code, this is feasible, as ``__globals__`` is set appropriately when the function definition is executed, and
even if the namespace has been manipulated to return a different object, at worst an exception will be raised.

By contrast, extension methods are typically implemented as normal C functions.
This means that they only have access to their arguments and C level thread-local
and process-global states. Traditionally, many extension modules have stored
their shared state in C-level process globals, causing problems when:
    
    * running multiple initialize/finalize cycles in the same process
    * reloading modules (e.g. to test conditional imports)
    * loading extension modules in subinterpreters

PEP 3121 attempted to resolve this by offering the ``PyState_FindModule`` API, but this still has significant problems when it comes to extension methods (rather than module level functions):

    * it is markedly slower than directly accessing C-level process-global state
    * there is still some inherent reliance on process global state that means it still doesn't reliably handle module reloading

It's also the case that when looking up a C-level struct such as module state, supplying
an unexpected object layout can crash the interpreter, so it's significantly more important to ensure that extension
methods receive the kind of object they expect.

Proposal
========

Currently, a bound extension method (``PyCFunction`` or ``PyCFunctionWithKeywords``) receives only
``self``, and (if applicable) the supplied positional and keyword arguments. 

While module-level extension functions already receive access to the defining module object via their
``self`` argument, methods of extension types don't have that luxury: they receive the bound instance
via ``self``, and hence have no direct access to the defining class or the module level state.

The additional module level context described above can be made available with two changes.
Both additions are optional; extension authors need to opt in to start
using them:

    * Add a pointer to the module to heap type objects.

    * Pass the defining class to the underlying C function.

      The defining class is readily available at the time built-in
      method object (``PyCFunctionObject``) is created, so it can be stored
      in a new struct that extends ``PyCFunctionObject``.

The module state can then be retrieved from the module object via
``PyModule_GetState``.

Note that this proposal implies that any type whose method needs to access
per-module state must be a heap type, rather than a static type.

This is necessary to support loading multiple module objects from a single
extension: a static type, as a C-level global, has no information about
which module it belongs to.


Slot methods
------------

The above changes don't cover slot methods, such as ``tp_iter`` or ``nb_add``.

The problem with slot methods is that their C API is fixed, so we can't
simply add a new argument to pass in the defining class.
Two possible solutions have been proposed to this problem:

    * Look up the class through walking the MRO.
      This is potentially expensive, but will be useful if performance is not
      a problem (such as when raising a module-level exception).
    * Storing a pointer to the defining class of each slot in a separate table,
      ``__typeslots__`` [#typeslots-mail]_.  This is technically feasible and fast,
      but quite invasive.

Due to the invasiveness of the latter approach, this PEP proposes adding a MRO walking helper for use in slot method implementations, deferring the more complex alternative as a potential future optimisation.


Immutable Exception Types
-------------------------

To faciliate creating static exception classes, a new function is proposed:
``PyErr_PrepareImmutableException``. It will work similarly to ``PyErr_NewExceptionWithDoc``
but will take a ``PyTypeObject **`` pointer, which points to a ``PyTypeObject *`` that is
either ``NULL`` or an initialized ``PyTypeObject``.
This pointer may be declared in process-global state. The function will then
allocate the object and will keep in mind that already existing exception
should not be overwritten.

(XXX: Update function name in the implementation)

The extra indirection makes it possible to make ``PyErr_PrepareImmutableException``
part of the stable ABI by having the Python interpreter, rather than extension code,
allocate the ``PyTypeObject``.


Specification
=============

Adding module references to heap types
--------------------------------------

The ``PyHeapTypeObject`` struct will get a new member, ``PyObject *ht_module``,
that can store a pointer to the module object for which the type was defined.
It will be ``NULL`` by default, and should not be modified after the type
object is created.

A new factory method will be added for creating modules::

    PyObject* PyType_FromModuleAndSpec(PyObject *module,
                                       PyType_Spec *spec,
                                       PyObject *bases)

This acts the same as ``PyType_FromSpecWithBases``, and additionally sets
``ht_module`` to the provided module object.

Additionally, an accessor, ``PyObject * PyType_GetModule(PyTypeObject *)``
will be provided.
It will return the ``ht_module`` if a heap type with module pointer set
is passed in, otherwise it will set a SystemError and return NULL.

Usually, creating a class with ``ht_module`` set will create a reference
cycle involving the class and the module.
This is not a problem, as tearing down modules is not a performance-sensitive
operation (and module-level functions typically also create reference cycles).
The existing "set all module globals to None" code that breaks function cycles
through ``f_globals`` will also break the new cycles through ``ht_module``.


Passing the defining class to extension methods
-----------------------------------------------

A new style of C-level functions will be added to the current selection of
``PyCFunction`` and ``PyCFunctionWithKeywords``::

    PyObject *PyCMethod(PyObject *self,
                        PyTypeObject *defining_class,
                        PyObject *args, PyObject *kwargs)

A new method object flag, ``METH_METHOD``, will be added to signal that
the underlying C function is ``PyCMethod``.

To hold the extra information, a new structure extending ``PyCFunctionObject``
will be added::

    typedef struct {
        PyCFunctionObject func;
        PyTypeObject *mm_class; /* Passed as 'defining_class' arg to the C func */
    } PyCMethodObject;

Method construction and calling code and will be updated to honor
``METH_METHOD``.


Argument Clinic
---------------

To support passing the defining class to methods using Argument Clinic,
a new converter will be added to clinic.py: ``defining_class``.

Each method may only have one argument using this converter, and it must
appear after ``self``, or, if ``self`` is not used, as the first argument.
The argument will be of type ``PyTypeObject *``.

When used, Argument Clinic will select ``METH_METHOD`` as the calling
convention.
The argument will not appear in ``__text_signature__``.

This will be compatible with ``__init__`` and ``__new__`` methods, where an
MRO walker will be used to pass the defining class from clinic generated
code to the user's function.


Slot methods
------------

To allow access to per-module state from slot methods, an MRO walker
will be implemented::

    PyTypeObject *PyType_DefiningTypeFromSlotFunc(PyTypeObject *type,
                                                  int slot, void *func)

The walker will go through bases of heap-allocated ``type``
and search for class that defines ``func`` at its ``slot``.

The ``func`` needs not to be inherited by ``type``, only requirement
for the walker to find the defining class is that the defining class
must be heap-allocated.

On failure, exception is set and NULL is returned.


Static exceptions
-----------------

A new function will be added::

    int PyErr_PrepareImmutableException(PyTypeObject **exc,
                                     const char *name,
                                     const char *doc,
                                     PyObject *base)

Creates an immutable exception type which can be shared
across multiple module objects.
If the type already exists (determined by a process-global pointer,
``*exc``), skip the initialization and only ``INCREF`` it.

If ``*exc`` is NULL, the function will
allocate a new exception type and initialize it using given parameters
the same way ``PyType_FromSpecAndBases`` would.
The ``doc`` and ``base`` arguments may be ``NULL``, defaulting to a
missing docstring and ``PyExc_Exception`` base class, respectively.
The exception type's ``tp_flags`` will be set to values common to
built-in exceptions and the ``Py_TPFLAGS_HEAP_IMMUTABLE`` flag (see below)
will be set.
On failure, ``PyErr_PrepareImmutableException`` will set an exception
and return -1.

If called with an initialized exception type (``*exc``
is non-NULL), the function will do nothing but incref ``*exc``.

(XXX: Support and test multiple inheritance)

A new flag, ``Py_TPFLAGS_HEAP_IMMUTABLE``, will be added to prevent
mutation of the type object. This makes it possible to
share the object safely between multiple interpreters.
This flag is checked in setattr (XXX: name of C function?) and blocks
setting of attributes when set, similar to built-in types.

A new pointer, ``ht_moduleptr``, will be added to heap types to store ``exc``.

(XXX: better name? Update implementation)

On deinitialization of the exception type, ``*exc`` will be set to ``NULL``.
This makes it safe for
``PyErr_PrepareImmutableException`` to check if the exception
was already initialized.


Helpers
-------

XXX: I'd like to port a bunch of modules to see what helpers would be convenient

XXX: Helper for getting module state directly from defining class


Modules Converted in the Initial Implementation
-----------------------------------------------

To validate the approach, several modules will be modified during
the initial implementation:

The ``zipimport``, ``_io``, ``_elementtree``, and ``_csv`` modules
will be ported to PEP 489 multiphase initialization.


Summary of API Changes and Additions
====================================

XXX, see above for now


Backwards Compatibility
=======================

Two new pointers are added to all heap types.
All other changes are adding new functions, structures and a type flag.

The new ``PyErr_PrepareImmutableException`` function changes encourages
modules to switch from using heap-type Exception classes to static ones,
and a number of modules will be switched in the initial implementation.
This change will prevent adding class attributes to such types.
For example, the following will raise AttributeError::

    sqlite.OperationalError.foo = None

Instances and subclasses of such exceptions will not be affected.

Implementation
==============

An initial implementation is available in a Github repository [#gh-repo]_;
a patchset is at [#gh-patch]_.


Possible Future Extensions
==========================

Easy creation of types with module references
---------------------------------------------

It would be possible to add a PEP 489 execution slot type to make
creating heap types significantly easier than calling
``PyType_FromModuleAndSpec``.
This is left to a future PEP.


Optimization
------------

CPython optimizes calls to methods that have restricted signatures,
such as not allowing keyword arguments.

As proposed here, methods defined with the ``METH_METHOD`` flag do not support
these optimizations.


References
==========

.. [#typeslots-mail] [Import-SIG] On singleton modules, heap types, and subinterpreters
   (https://mail.python.org/pipermail/import-sig/2015-July/001035.html)

.. [#gh-repo]
   https://github.com/Traceur759/cpython/commits/pep-c

.. [#gh-patch]
   https://github.com/Traceur759/cpython/compare/master...Traceur759:pep-c.patch


Copyright
=========

This document has been placed in the public domain.



..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
