# Extending the Ice interface

The server component supports RPC via [ZeroC Ice](https://zeroc.com/products/ice). This document describes how the Ice interface of the server can be
extended.

Note: If not stated otherwise all referenced files live in `src/murmur/`.

The files involved in extending the Ice interface are
| **File** | **Description** |
| -------- | --------------- |
| `Murmur.ice` | Contains the formal definition of the interface |
| `Murmur.h` | Contains the C++ interface definition (abstract base classes). This file is automatically generated based on `Murmur.ice` when invoking cmake (via `slice2cpp`).  It lives in `<build directory>/src/murmur`. This file is needed to generate `MurmurIceWrapper.cpp` |
| `Murmur.cpp` | Contains some boilerplate and Ice-internal implementation code. This file is generated and lives alongside `Murmur.h` |
| `MurmurI.h` | Contains the definition of the actually implemented API classes (`ServerI` and `MetaI`). These extend the abstract base classes from `Murmur.h` |
| `MurmurIceWrapper.cpp` | Contains wrapper implementation of the `*I` API classes. This file is auto-generated by the `scripts/generateIceWrapper.py` script |
| `MurmurIce.h` | Contains the definition of a statically used helper class |
| `MurmurIce.cpp` | Contains the implementation of that helper class **and** _static_ functions used to actually implement the server-side functionality of the Ice API functions |
| `RPC.cpp` | Contains the implementations of the `Server` (the Mumble server, _not_ the Ice API type) class's member functions that are required to make certain functionality accessible to the static functions in `MurmurIce.cpp` |


## Overview

The steps are
1. Let cmake invoke `slice2cpp` to generate `Murmur.h` and `Murmur.cpp`
2. Invoke `generateIceWrapper.py` to generate `MurmurIceWrapper.cpp`
3. Add new function declarations to `MurmurU.h`
4. Write impl function in `MurmurIce.cpp`
5. Potentially write new public API functions (declare in `Server.h` and define in `RPC.cpp`

## Detailed instructions

Before proceeding any further you should run cmake once in order for these files to get generated as they are needed for the next step. Assuming your
build directory lives directly under the repository's root, you can do this by invoking
```bash
cmake ..
```

The next step is the generation of the `MurmurIceWrapper.cpp` file by executing the following command (assuming the call is performed from the
repository's root and the build directory is called `build`):
```bash
$ python3 scripts/generateIceWrapper.py --ice-file src/murmur/Murmur.ice --generated-ice-header build/src/murmur/Murmur.h --out-file src/murmur/MurmurIceWrapper.cpp 
Using ICE-file at                   "src/murmur/Murmur.ice"
Using ICE-generated header file at  "build/src/murmur/Murmur.h"
```
The paths that are given in the example may have to be adapted in order for it to work on your machine.

The `MurmurIceWrapper.cpp` file generates the `*_async` versions of the Ice callbacks that handle the async nature of these callbacks and also
contain the boilerplate for e.g. verification of the caller and things like this. Most importantly though these functions call the `impl_*` functions
defined in `MurmurIce.cpp`. For instance a function called `updateCertificates` inside the `Server` class will call `impl_Server_updateCertificate`
which has to be defined as a `static` function inside `MurmurIce.cpp`.

The declaration of the async functions generated this way are contained inside `MurmurI.h`. You have to manually add the function's declaration into
there. The easiest way to do this is to let the script generate the implementation as described above and then copy the function signature from there
into the `Murmur.h` file (make the declaration `virtual` though).

The impl function's signature is always
```cpp
static void impl_<className>_<functionName>(const ::Murmur::AMD_<className>_<functionName>Ptr cb [, int server_id] [, <function arguments>]) {
    // Implementation goes here

    cb->ice_response([<function return value>]);
}
```
- `<className>`: Name of the class the function is declared in (e.g. `Server` or `Meta`)
- `<functionName>`: Name of the function as declared in the `Murmur.ice` file
- `[, int server_id]`: Only needed when extending the `Server` API class (the brackets are not part of what needs to be written in code)
- `[, <function arguments>]`: To be replaced by the list of arguments the function takes
- `[<function return value>]`: To be replaced with the value this function returns or to be removed if the function does not return anything.


If you have used non-default types that are declared in `Murmur.ice` (e.g. `IdList`), you can reference them here as `::Murmur::<typeName>` (e.g.
`::Murmur::IdList`).

Error reporting works via the `cb->ice_exception` function and if everything went well, the function must end by calling `cb->ice_response`
(potentially passing a value to that function that shall be returned to the caller of the function).

In general it is a good idea to have a look at the existing implementation inside `MurmurIce.cpp` and take inspiration from those.

Note that the implementations make heavy use of macros (e.g. `NEED_SERVER`, `NEED_CHANNEL`, etc.). These will initialize the corresponding variables
(`server`, `channel`, etc.) based in the parameters fed into the function (In order to obtain the channel, user, etc. you always have to initialize
the `server` variable first). For this to work it is essential that you are using the same parameter names as the existing implementations (e.g.
`server_id` for the server's ID). (For historical reasons the macro to obtain the `user` variable is called `NEED_PLAYER`)

If the function requires action on the server's side (beyond its public API), you have to declare a new public function in the `Server` class (this
time the Mumble server though; not the Ice server class) defined in `Server.h` (the definitions belong to the group of other RPC functions in there -
section marked by a comment). The implementation of this new function should then be written in `RPC.cpp`.

An example of when this is needed is for instance if you have to access the list of connected clients (e.g. because you want to send them a message).
While in the current state of the code it would be possible to access this list from the outside (public visibility), you should prefer creating a
public API function in the Mumble `Server` class that has the implementation in `RPC.cpp`.