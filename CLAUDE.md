# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

xrtransport is a transparent remote OpenXR runtime that serializes OpenXR API calls on a client, transmits them to a server, executes them on the server's OpenXR runtime, and returns results. The primary use case is running Meta Quest applications in Android emulators on PC with local HMD hardware.

The protocol is stateful and tightly coupled—clients and servers must be built from the same commit. The serialization format is dynamic with minimal metadata, relying on context for deserialization.

## Build Commands

### Code Generation
**CRITICAL**: Run code generation after OpenXR spec changes, custom struct modifications, or new extensions:
```bash
./regenerate.sh
```
This requires Python 3 with the Mako package (`pip install -r requirements.txt`).

### Building with CMake Presets
**Default build (debug with Vulkan2 module)**:
```bash
cmake --preset default-debug
cmake --build --preset default-debug-build
```

**Manual build**:
```bash
mkdir build && cd build
cmake ..
cmake --build .
```

**Android builds** (Waydroid target - client only, no server/tests):
```bash
cmake --preset waydroid-x86_64-debug
cmake --build --preset waydroid-x86_64-debug-build
```
Available Android ABIs: `x86_64`, `x86`, `arm64-v8a`, `armeabi-v7a`

### CMake Build Options
- `XRTRANSPORT_BUILD_CLIENT` (default ON) - Build client runtime
- `XRTRANSPORT_BUILD_SERVER` (default ON) - Build server
- `XRTRANSPORT_BUILD_TESTS` (default ON) - Build test suite
- `XRTRANSPORT_BUILD_MODULE_VULKAN2` - Build Vulkan2 module (enabled in default-debug preset)

### Build Artifacts
When using the default-debug preset:
- Server: `build/default/debug/src/server/xrtransport_server_main`
- Client runtime: `build/default/debug/src/client/libxrtransport_client.so`
- Client manifest: `build/default/debug/src/client/xrtransport_client_manifest.json`

When using manual build (`mkdir build && cmake ..`):
- Server: `build/src/server/xrtransport_server_main`
- Client runtime: `build/src/client/libxrtransport_client.so`
- Client manifest: `build/src/client/xrtransport_client_manifest.json`

### Running Tests
Using default-debug preset:
```bash
# Serialization fuzzer
./build/default/debug/test/serialization/serialization_tests

# Transport unit tests
./build/default/debug/test/transport/transport_tests

# Transport integration tests (requires starting server first)
./build/default/debug/test/transport/transport_server &
./build/default/debug/test/transport/transport_integration_tests
```

Using manual build:
```bash
# Serialization fuzzer
./build/test/serialization/serialization_tests

# Transport unit tests
./build/test/transport/transport_tests

# Transport integration tests (requires starting server first)
./build/test/transport/transport_server &
./build/test/transport/transport_integration_tests
```

**Windows note**: Copy `xrtransport_transport.dll` from `build/src/transport/<config>/` to test executable directories.

## Architecture

### Three-Layer System

1. **Client (OpenXR API Layer)**: Intercepts OpenXR calls, serializes arguments, sends to server via RPC
   - Location: `src/client/`
   - Entry point: `src/client/entry.cpp` (implements OpenXR API layer interface)
   - RPC implementation: `src/client/rpc.cpp` (generated from templates)
   - Function table: `src/client/function_table.cpp` (generated)

2. **Server**: Receives serialized calls, deserializes, executes on native runtime, returns results
   - Location: `src/server/`
   - Main: `src/server/main.cpp`
   - Function dispatch: `src/server/function_dispatch.cpp` (generated from templates)

3. **Transport Layer**: Handles client-server communication (TCP by default)
   - Location: `src/common/transport/`
   - C API: `include/xrtransport/transport/transport_c_api.h`
   - C++ wrapper: `include/xrtransport/transport/transport.h`

### Serialization System
- Location: `src/common/serialization/`
- Generated from OpenXR spec via Mako templates in `code_generation/templates/structs/`
- Custom handling: `src/common/serialization/custom_serializer.cpp` and `custom_deserializer.cpp`
- Protocol: Defined in `protocol.txt` - includes handshake, function calls, returns, and sync messages

### Module System

**Purpose**: Extend xrtransport with graphics API support (Vulkan, OpenGL ES, etc.)

**Client Modules** (API layers):
- Loaded dynamically by client runtime
- Must implement: `module_get_info()` in `include/xrtransport/client/module_interface.h`
- Can layer functions and advertise extensions
- Access transport via `xrtransportGetTransport` (obtained through `xrGetInstanceProcAddr`)

**Server Modules**:
- Loaded from `modules/` directory next to server executable
- Must implement interface in `include/xrtransport/server/module_interface.h`:
  - `on_init()` - register handlers, load XR functions
  - `get_required_extensions()` - specify needed XR extensions
  - `on_instance()` - post-instance setup
  - `on_shutdown()` - cleanup
- Can send/receive custom messages via transport

**Current modules**:
- `vulkan2`: Vulkan graphics support (in development)
  - Client: `src/modules/vulkan2/client/`
  - Server: `src/modules/vulkan2/server/`

Both client and server modules must link with `xrtransport_transport` library for stream access.

### Code Generation System

Location: `code_generation/`

**Key files**:
- `__main__.py` - Main entry point, orchestrates generation
- `spec_parser.py` - Parses OpenXR spec XML
- `bindings.py` - Determines which function parameters are modifiable (for return path)
- `function_ids.py` - Manages function ID assignment/updates
- `struct_fuzzer.py` - Generates random OpenXR structs for tests

**Generated files** (marked AUTO-GENERATED):
- Client RPC stubs: `src/client/rpc.{h,cpp}`
- Server dispatch: `src/server/function_dispatch.{h,cpp}`
- Serializers: `include/xrtransport/serialization/serializer.h`, `src/common/serialization/serializer.cpp`
- Deserializers: `include/xrtransport/serialization/deserializer.h`, `src/common/serialization/deserializer.cpp`
- Extension support: `include/xrtransport/extensions/enabled_extensions.h`, `extension_functions.h`
- Test fuzzer: `test/serialization/fuzzer.cpp`

**Function ID management**: `function_ids.json` (located in project root) maps OpenXR functions to protocol IDs. Use `--regenerate-function-ids` flag only for complete regeneration; default behavior incrementally updates IDs when spec changes.

### Important Implementation Details

- **Protocol statefulness**: Client/server must serialize/deserialize using identical code. Struct shape is unpredictable due to variable-length arrays and pNext chains.
- **Modifiable bindings**: Only non-const parameters are sent back from server to client after function execution. Logic in `code_generation/bindings.py`.
- **Synchronization**: Client sends `XRTP_MSG_SYNCHRONIZATION_REQUEST` with client time; server responds with server time for clock sync.
- **Time handling**: OpenXR uses `XrTime` (nanoseconds). Client/server clock sync in `src/client/synchronization.cpp`.

## File Organization

```
xrtransport/
├── code_generation/        # Code generation system
│   ├── templates/          # Mako templates for generated code
│   │   ├── client/         # Client RPC generation
│   │   ├── server/         # Server dispatch generation
│   │   ├── structs/        # Serializer/deserializer generation
│   │   └── test/           # Test fuzzer generation
│   └── *.py                # Generation logic
├── external/               # Git submodules (OpenXR-SDK, spdlog)
├── include/xrtransport/    # Public headers
│   ├── client/             # Client module interface
│   ├── server/             # Server module interface
│   ├── transport/          # Transport layer API
│   └── serialization/      # Serialization API (generated)
├── src/
│   ├── client/             # OpenXR client runtime
│   ├── server/             # Server executable
│   ├── common/             # Shared code
│   │   ├── serialization/  # Serializer/deserializer implementations
│   │   ├── transport/      # Transport implementation
│   │   └── config/         # Configuration handling
│   └── modules/            # Client/server modules
│       └── vulkan2/        # Vulkan 2 module (in progress)
├── test/
│   ├── serialization/      # Serialization fuzzer tests
│   └── transport/          # Transport layer tests
└── doc/                    # Documentation
```

## Development Notes

- **C++ Standard**: C++17 (set in root CMakeLists.txt)
- **Logging**: Uses spdlog (in `external/`)
- **OpenXR dependency**: OpenXR-SDK submodule in `external/` (server only)
- **Client manifest**: Auto-generated via `create_openxr_manifest.cmake` function
- **Position Independent Code**: Enabled globally for static→shared library linking

When modifying serialization behavior or adding custom struct handling, edit the custom serializer/deserializer files, not the generated ones. When adding new OpenXR extensions or functions, run `./regenerate.sh` to update generated code.
