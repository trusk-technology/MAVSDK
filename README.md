## Trusk changes

> This is a fork with local changes to support a **custom ArduPilot flight-mode number for offboard control**.

By default, starting offboard control forces ArduPilot into `GUIDED` mode. In this fork you can pass a custom
mode number; `mavsdk_server` then sends a `MAV_CMD_DO_SET_MODE` command (with the mode in `param2`) before
entering offboard.

### What changed

- `proto/protos/offboard/offboard.proto`: `StartRequest` gained `uint32 mode = 1; /* 0 keeps the default behaviour (GUIDED on ArduPilot). */`
- `offboard_impl.cpp/.hpp`: `start()`/`start_async()` now take a `uint32_t mode`. When `mode != 0` and the
autopilot is ArduPilot they send `MAV_CMD_DO_SET_MODE` (`param1` from arming state, `param2` = `mode`);
`mode == 0` (and non-ArduPilot autopilots) keep the original `set_flight_mode(FlightMode::Offboard)` path.
- Regenerated proto/C++ bindings (`offboard.hpp/.cpp`, `offboard_service_impl.hpp`, protobuf stubs) reflect
the new `mode` argument.



### How to build both projects

The commands below assume that `trusk-mavsdk-cpp` and `trusk-mavsdk` are sibling directories.
CMake 3.22.1 or newer, a C++ compiler, Git, Python, and
[Hatch](https://hatch.pypa.io/latest/install/) are required.

1. **Build** `mavsdk_server` **from the C++ repository root:**
  ```sh
   git submodule update --init --recursive

   cmake -S cpp -B cpp/build/default \
       -DCMAKE_BUILD_TYPE=Release \
       -DBUILD_MAVSDK_SERVER=ON
   cmake --build cpp/build/default \
       --target mavsdk_server_bin \
       -j"$(nproc)"
  ```
   `mavsdk_server_bin` is the CMake target name. On Unix-like systems, the resulting executable is named
   `mavsdk_server` and is written to:
2. **Build the Python package with that exact server:**
  ```sh
   cd ../trusk-mavsdk
   git submodule update --init --recursive

   export MAVSDK_CPP_PROJECT_ROOT="$(realpath ../trusk-mavsdk-cpp/cpp/build/default)"
   # On x86-64 Linux, this also makes an accidental release fallback use the host architecture.
   export MAVSDK_SERVER_ARCH=x86_64

   hatch run build
  ```
   `hatch run build` regenerates the Python bindings, copies `mavsdk_server` from the configured directory,
   and builds the wheel and source distribution. Confirm that it prints `Found local mavsdk_server`; if no
   local executable is found, the downloader falls back to an upstream release that does not contain this
   fork's local C++ changes. To install the checkout for editable development after generating and copying
   the server, run `hatch run install-local`.

`MAVSDK_CPP_PROJECT_ROOT` is searched recursively and the newest matching executable is selected. Pointing it
at the intended build directory, rather than the entire C++ checkout, prevents a stale build for another
architecture from being selected.

> **Shared-library note:** the default C++build links against the corresponding++ `libmavsdk_server` ++and++
> `libmavsdk` ++libraries in the C++ build tree. Copying only the executable does not make it self-contained.
> Keep `cpp/build/default` available, or configure the runtime library path as needed, and use `ldd` to check
> that no dependency is reported as `not found`.



### NVIDIA Jetson Orin Nano (ARM64/aarch64)

Build natively on the Jetson with its normal compiler. Use a fresh build directory so an x86-64 CMake cache or
binary cannot be reused accidentally:

```sh
# In trusk-mavsdk-cpp on the Jetson
sudo apt-get update
sudo apt-get install -y build-essential cmake git python3 python3-pip
cmake --version  # must be 3.22.1 or newer
uname -m         # expected: aarch64

git submodule update --init --recursive

cmake -S cpp -B cpp/build/jetson-aarch64 \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_MAVSDK_SERVER=ON \
    -DBUILD_TESTING=OFF
cmake --build cpp/build/jetson-aarch64 \
    --target mavsdk_server_bin \
    -j"$(nproc)"

SERVER="$PWD/cpp/build/jetson-aarch64/src/mavsdk_server/src/mavsdk_server"
test -x "$SERVER"
file "$SERVER"
ldd "$SERVER"
```

`file` must identify a 64-bit ARM/AArch64 ELF executable, and `ldd` must contain no `not found` entries. Do not
set `CMAKE_SYSTEM_PROCESSOR` for this native build; the Jetson compiler already selects AArch64. This repository
does not provide a JetPack/glibc-sysroot-specific cross-compilation recipe, so a native build is the recommended
way to target the Jetson.

Then build the Python package on the Jetson:

```sh
cd ../trusk-mavsdk
git submodule update --init --recursive
python3 -m pip install hatch

export MAVSDK_CPP_PROJECT_ROOT="$(realpath ../trusk-mavsdk-cpp/cpp/build/jetson-aarch64)"
export MAVSDK_SERVER_ARCH=aarch64

hatch run build
hatch run install-local  # optional editable development installation

file mavsdk/bin/mavsdk_server
ldd mavsdk/bin/mavsdk_server
```

`MAVSDK_SERVER_ARCH=aarch64` controls only the architecture of a release-download fallback; it does not compile
or select a local C++ binary. Confirm that the downloader found the local Jetson build and that the copied binary
is still AArch64. The default shared build and a wheel containing it are tied to the matching build tree and are
not automatically portable to other ARM64 systems or JetPack/glibc versions.



[Linux](https://github.com/mavlink/MAVSDK/actions/workflows/linux.yml)
[macOS](https://github.com/mavlink/MAVSDK/actions/workflows/macos.yml)
[Windows](https://github.com/mavlink/MAVSDK/actions/workflows/windows.yml)
[Docs](https://github.com/mavlink/MAVSDK/actions/workflows/docs_deploy.yml)

## Description

[MAVSDK](https://mavsdk.mavlink.io/main/en/) is a set of libraries providing a high-level API to [MAVLink](https://mavlink.io/en/).
It aims to be:

- Easy to use with a simple API supporting both synchronous (blocking) API calls and asynchronous API calls using callbacks.
- Fast and lightweight.
- Cross-platform (Linux, macOS, Windows, iOS, Android).
- Extensible (using the `MavlinkDirect` plugin, or the soon-to-be-deprecated `MavlinkPassthrough` plugin).
- Fully compliant with the MAVLink standard/definitions.

In order to support multiple programming languages, MAVSDK implements a gRPC server in C++ which allows clients in different programming languages to connect to. The API is defined by the proto IDL ([proto files](https://github.com/mavlink/MAVSDK-Proto/tree/master/protos)).
This architecture allows the clients to be implemented in idiomatic patterns, so using the tooling and syntax expected by end users. For example, the Python library can be installed from PyPi using `pip`.

The MAVSDK C++ part consists of:

- The [core library](https://github.com/mavlink/MAVSDK/tree/main/cpp/src/mavsdk/core) implementing the basic MAVLink communication.
- The [plugin libraries](https://github.com/mavlink/MAVSDK/tree/main/cpp/src/mavsdk/plugins) implementing the MAVLink communication specific to a feature.
- The [mavsdk_server](https://github.com/mavlink/MAVSDK/tree/main/cpp/src/mavsdk_server) implementing the gRPC server for the language clients.



## Repos

- [MAVSDK](https://github.com/mavlink/MAVSDK) - this repo containing the source code for the C++ core.
- [MAVSDK-Proto](https://github.com/mavlink/MAVSDK-Proto) - Common interface definitions for API specified as proto files used by gRPC between language clients and mavsdk_server.
- [MAVSDK-Python](https://github.com/mavlink/MAVSDK-Python) - MAVSDK client for Python (first released on Pypi 2019).
- [MAVSDK-Swift](https://github.com/mavlink/MAVSDK-Swift) - MAVSDK client for Swift (used in production, first released 2018).
- [MAVSDK-Java](https://github.com/mavlink/MAVSDK-Java) - MAVSDK client for Java (first released on MavenCentral in 2019).
- [MAVSDK-Go](https://github.com/mavlink/MAVSDK-Go) - MAVSDK client for Go (work in progress).
- [MAVSDK-JavaScript](https://github.com/mavlink/MAVSDK-JavaScript) - MAVSDK client in JavaScript (proof of concept, 2019).
- [MAVSDK-Rust](https://github.com/mavlink/MAVSDK-Rust) - MAVSDK client for Rust (proof of concept, 2019).
- [MAVSDK-CSharp](https://github.com/mavlink/MAVSDK-CSharp) - MAVSDK client for CSharp (proof of concept, 2019).
- [Docs](https://github.com/mavlink/MAVSDK/tree/main/docs) - MAVSDK [docs](https://mavsdk.mavlink.io/main/en/) source.



## Docs

Instructions for how to use the C++ library can be found in the [MAVSDK docs](https://mavsdk.mavlink.io/main/en/) (links to other programming languages can be found from the documentation sidebar).

Quick Links:

- [Getting started](https://mavsdk.mavlink.io/main/en/cpp/#getting-started)
- [C++ API Overview](https://mavsdk.mavlink.io/main/en/cpp/#api-overview)
- [API Reference](https://mavsdk.mavlink.io/main/en/cpp/api_reference/)
- [Installing the Library](https://mavsdk.mavlink.io/main/en/cpp/guide/installation.html)
- [Building the Library](https://mavsdk.mavlink.io/main/en/cpp/guide/build.html)
- [Examples](https://mavsdk.mavlink.io/main/en/cpp/examples/)
- [FAQ](https://mavsdk.mavlink.io/main/en/faq.html)



## License

This project is licensed under the permissive BSD 3-clause, see [LICENSE.md](LICENSE.md).

## Maintenance

This project is maintained by volunteers:

- [Julian Oes](https://github.com/julianoes) ([sponsoring](https://github.com/sponsors/julianoes), [consulting](https://julianoes.com)).
- [Jonas Vautherin](https://github.com/JonasVautherin)

Maintenance is not sponsored by any company, however, hosting of the [docs](https://mavsdk.mavlink.io/main/en/) and the [forum](https://discuss.px4.io/c/mavsdk/) is provided by the [Dronecode Foundation](https://dronecode.org).

## Support and issues

If you just have a question, consider asking in the [forum](https://discuss.px4.io/c/mavsdk/).

If you have run into an issue, discovered a bug, or want to request a feature, create an [issue](https://github.com/mavlink/MAVSDK/issues). If it is important or urgent to you, consider sponsoring any of the maintainers to move the issue up on their todo list.

If you need private support, consider paid consulting:

- [Julian Oes consulting](https://julianoes.com)

(Create a pull request if you wish to be listed here.)