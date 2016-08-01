# Coprocess (Python)

This feature makes it possible to write Tyk middleware using [Python](https://www.python.org/), the current binding supports Python 3.x.
The purpose of this README is to provide an overview of the architecture and a few implementation notes.

## Build requirements

* [Python 3.x](https://www.python.org/)
* [Go](https://golang.org)
* [Cython](http://cython.org/) (required if you need to modify and re-compile the gateway API binding)

## Build steps

To build Tyk with the Coprocess + Python support, use:

```
go build -tags 'coprocess python'
```

To compile the gateway API binding (assuming you're on the repository root):

```sh
cd coprocess/python
./cythonize gateway
```

This will "cythonize" `gateway.pyx`, generating `gateway.c` and `gateway.h`.

To compile some other binding (where `mybinding.pyx` is your Cython input file):

```sh
cd coprocess/python
./cythonize mybinding
```

[cythonize](cythonize) is a helper script for compiling Python source files with Cython and patching the resulting source with the specific build tags used by this Coprocess feature.
This is important in order to keep Tyk build-able when a standard build is needed (and make the Go compiler ignore C binding files based on the specified build tags).

The top of a standard Cython binding file will look like this:
```
/* Generated by Cython 0.24.1 */

/* BEGIN: Cython Metadata
{
    "distutils": {
        "depends": []
    },
    "module_name": "gateway"
}
END: Cython Metadata */
```

After running `cythonize` the binding will have the correct build tags, and will be ignored if you don't build Tyk with these (`go build -tags 'coprocess python'`):

```
// +build coprocess
// +build python

/* Generated by Cython 0.24.1 */

/* BEGIN: Cython Metadata
{
    "distutils": {
        "depends": []
    },
    "module_name": "gateway"
}
END: Cython Metadata */
```

After re-compiling a binding, the C source code will change and you may want to build Tyk again.

## CPython

[Python](https://www.python.org/) has a very popular and well-documented [C API](https://docs.python.org/3/c-api/index.html), this coprocess feature makes a heavy use of it.

## Interop

## Built-in modules

All the standard Python modules are available and it's also possible to load additional ones, if you add them to your local Python installation (for example, using pip).

### Coprocess Gateway API

There's a Python binding for the [Coprocess Gateway API](../README.md), this is written using the Cython syntax, it's basically a single file: [`gateway.pyx`](tyk/gateway.pyx).

This binding exposes some functions, like a storage handler that allows you to get/set Redis keys:

```python
from tyk.decorators import *
from gateway import TykGateway as tyk

@Pre
def SetKeyOnRequest(request, session, spec):
    tyk.store_data( "my_key", "expiring_soon", 15 )
    val = tyk.get_data("cool_key")
    return request, session
```

### Cython bindings

Cython takes a `.pyx` file and generates a C source file and its corresponding header, after this process, we use these two files as part of the `cgo` build process. This approach has been used as an alternative to `cffi`, which introduced an additional step into the setup, requiring the user to install the module first.

So in practice, we don't use the `.pyx` files directly (and they aren't required at runtime!). When the build process is over, the bindings are part of the Tyk binary and can be loaded and accessed from Go code.

The bindings [declare an initialization function](tyk/gateway.h) that should be called after the Python interpreter is invoked, this will load the actual module and make it possible to import it using `import mymodule`. This is how [`gateway.pyx`](tyk/gateway.pyx) and its functions become available.

### Middleware and wrappers

There are [quick wrappers](tyk/) for the HTTP request and session objects, written in Python, the idea is to provide an idiomatic way of writing middleware:

```python
from tyk.decorators import *

@Pre
def AppendHeader(request, session, spec):
    request.add_header("custom_header", "custom_value")
    return request, session
```

The decorators provide a simple way of indicating when it's the right moment to execute your handlers, a handler that is decorated with `Pre` will be called before any authentication occurs, `Post` will occur after authentication and will have access to the `session` object.
You may find more information about Tyk middleware [here](https://tyk.io/docs/tyk-api-gateway-v1-9/javascript-plugins/middleware-scripting/).