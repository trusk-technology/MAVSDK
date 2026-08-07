## Trusk changes

> This is a fork with local changes to support a **custom ArduPilot flight-mode number for offboard control**.

By default, starting offboard control forces ArduPilot into `GUIDED` mode. In this fork you can pass a custom
mode number; `mavsdk_server` then sends a `MAV_CMD_DO_SET_MODE` command (with the mode in `param2`) before
entering offboard.

### What changed

- `proto/protos/offboard/offboard.proto`: `StartRequest` gained `uint32 mode = 1; /* 0 keeps the default
  behaviour (GUIDED on ArduPilot). */`
- `offboard_impl.cpp/.hpp`: `start()`/`start_async()` now take a `uint32_t mode`. When `mode != 0` and the
  autopilot is ArduPilot they send `MAV_CMD_DO_SET_MODE` (`param1` from arming state, `param2` = `mode`);
  `mode == 0` (and non-ArduPilot autopilots) keep the original `set_flight_mode(FlightMode::Offboard)` path.
- Regenerated proto/C++ bindings (`offboard.hpp/.cpp`, `offboard_service_impl.hpp`, protobuf stubs) reflect
  the new `mode` argument.

### How to build both projects

1. **C++ (mavsdk_server)** – build as usual, then configure the Python repo to reuse the local binary:
   ```sh
   # from reporoot, build mavsdk_server (as in the original build instructions):
   cmake -B build/default -DCMAKE_BUILD_TYPE=Release \
         -DBUILD_MAVSDK_SERVER=ON -DBUILD_MAVSDK_LIB=ON -DBUILD_MAVSDK_CORE=ON
   cmake --build build/default -j$(nproc)
   ```
2. **Python** – the `download_server` script (`other/tools/download_server.py`) copies the newest built
   `mavsdk_server` from the folder in `MAVSDK_CPP_PROJECT_ROOT` (default `../trusk-mavsdk-cpp`) into
   `mavsdk/bin/`, so the local mode changes are used instead of the released binary:
   ```sh
   export MAVSDK_CPP_PROJECT_ROOT=../trusk-mavsdk-cpp   # optional; this is the default
   hatch run install-plugin
   hatch run download-server
   hatch run generate
   ```
   > Note: the locally-built `mavsdk_server` relies on the shared libs in the C++ build dir
   > (`libmavsdk_server.so.*`). Keep the build tree around, or set `RPATH`/`LD_LIBRARY_PATH` accordingly.

<img alt="MAVSDK" src="docs/assets/site/sdk_logo_full.png" width="400">

[![Linux](https://github.com/mavlink/MAVSDK/actions/workflows/linux.yml/badge.svg?branch=main)](https://github.com/mavlink/MAVSDK/actions/workflows/linux.yml)
[![macOS](https://github.com/mavlink/MAVSDK/actions/workflows/macos.yml/badge.svg?branch=main)](https://github.com/mavlink/MAVSDK/actions/workflows/macos.yml)
[![Windows](https://github.com/mavlink/MAVSDK/actions/workflows/windows.yml/badge.svg?branch=main)](https://github.com/mavlink/MAVSDK/actions/workflows/windows.yml)
[![Docs](https://github.com/mavlink/MAVSDK/actions/workflows/docs_deploy.yml/badge.svg?branch=main)](https://github.com/mavlink/MAVSDK/actions/workflows/docs_deploy.yml)

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
