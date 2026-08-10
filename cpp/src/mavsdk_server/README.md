# MAVSDK with GRPC

### Build with mavsdk_server

From the `cpp/` directory:

```bash
cmake -S . -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_MAVSDK_SERVER=ON
cmake --build build --target mavsdk_server_bin -j8
```

### Run the mavsdk_server

`mavsdk_server_bin` is the CMake target name. On non-MSVC platforms, the executable is named
`mavsdk_server` and can be run with:

```bash
./build/src/mavsdk_server/src/mavsdk_server
```

MSVC retains the `mavsdk_server_bin.exe` output name.
