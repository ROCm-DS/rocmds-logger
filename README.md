<!-- MIT License
  --
  -- Modifications Copyright (c) 2025 Advanced Micro Devices, Inc.
  --
  -- Permission is hereby granted, free of charge, to any person obtaining a copy
  -- of this software and associated documentation files (the "Software"), to deal
  -- in the Software without restriction, including without limitation the rights
  -- to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
  -- copies of the Software, and to permit persons to whom the Software is
  -- furnished to do so, subject to the following conditions:
  --
  -- The above copyright notice and this permission notice shall be included in all
  -- copies or substantial portions of the Software.
  --
  -- THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
  -- IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
  -- FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
  -- AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
  -- LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
  -- OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
  -- SOFTWARE.
-->

> [!CAUTION]
> This release is an *early-access* software technology preview. Running production workloads is *not* recommended.

> [!NOTE]
> This ROCm&trade; port is derived from the NVIDIA RAPIDS&reg; rapids-logger project (**commit: 8968ab3337f31c845d4e3bf6c55ae89242ded22b**). It aims to
> follow the latter's directory structure, file naming and API naming as closely as possible to minimize porting
> friction for users that are interested in using both projects.

# About

The `rapids-logger` project defines an easy way to produce a project-specific logger using the excellent [spdlog](https://github.com/gabime/spdlog) package.
This project has two primary goals:
1. Ensure that projects wishing to provide their own logger may do so easily without needing to reimplement their own custom wrappers around spdlog.
2. Ensure that custom logger implementations based on spdlog do not leak any spdlog (or fmt) symbols, allowing the safe coexistence of different projects in the same environment even if they use different versions of spdlog.

`rapids-logger` is designed to be used via CMake.
Its CMakeLists.txt defines a function `rapids_make_logger` that can be used to produce a project-specific logger class in a provided namespace.
The resulting logger exposes spdlog-like functionality via the [PImpl idiom](https://en.cppreference.com/w/cpp/language/pimpl) to avoid exposing spdlog symbols publicly.
It uses CMake and template C++ files to generate a public header file to describe the user interface and an inline header that should be placed in a single TU by consumers to compile the implementation.
To simplify usage, each invocation of the function produces two CMake targets, one representing the public header and one representing a trivial source file including the inline header.
Projects using `rapids-logger` should make the first target part of their public link interface while the latter should be linked to privately so that it is compiled into the project's library without public exposure.

Logging levels are controlled both at compile-time and at runtime.
To mirror spdlog, each generated logger ships with a set of logging macros `<project-name>_LOG_<log-level>`.
These macros are compiled based on the value of the compile-time variable `<project-name>_LOG_ACTIVE_LEVEL`.
For example, a project called "RAPIDS" will be able to write code like this:
```
RAPIDS_LOG_DEBUG("Some message to be shown when the debug level is enabled");
```
and control whether that message is shown by compiling the code with `RAPIDS_LOG_ACTIVE_LEVEL=RAPIDS_LOG_LEVEL_DEBUG`.
Additionally, the default runtime logging level can be controlled at compile time through the `LOGGER_DEFAULT_LEVEL` argument of `rapids_make_logger`.
This default runtime value allows for compiling with `INFO` level messages available, but only showing `WARN` or higher at runtime by default.
Users can then opt in to more verbose logging at runtime using `default_logger().set_level(...)`.

Each project is endowed with its own definition of levels, so different projects in the same environment may be safely configured independently of each other and of spdlog.
Each project is also given a `default_logger` function that produces a global logger that may be used anywhere, but projects may also freely instantiate additional loggers as needed.


# Usage Guide

This guide explains how to use the rocmds-logger project in your CMake-based C++ projects.

## Overview

rocmds-logger is a **CMake-only** project that provides the `rapids_make_logger()` function to generate project-specific logger implementations based on spdlog.
It uses the PImpl idiom to avoid exposing spdlog symbols publicly, allowing safe coexistence of different projects using different spdlog versions.
For easy plug-and-play with existing RAPIDS projects, the package is called `RAPIDS_LOGGER`.

## Installation and Consumption

### Package Information

* **Package Name**: `RAPIDS_LOGGER`
* **CMake Function**: `rapids_make_logger()`
* **Version**: 1.0.0

### Finding the Package

To use rocmds-logger in your rapids CMake project, first find the package:

```cmake
rapids_cpm_init()

# Include and use RAPIDS_LOGGER via CPM
include(${rapids-cmake-dir}/cpm/rocmds_logger.cmake)
rapids_cpm_rapids_logger()
```

### Basic Usage

Once the package is found, you can use the `rapids_make_logger()` function to generate a logger for your project:

```cmake
# Generate a logger for your project namespace
rapids_make_logger(myproject
    EXPORT_SET myproject-exports
    LOGGER_DEFAULT_LEVEL INFO
    LOGGER_HEADER_DIR include/myproject
)
```

This creates two CMake targets:

* `myproject_logger` - Interface target providing the logger header
* `myproject_logger_impl` - Implementation target that must be linked privately

### Linking to Your Targets

```cmake
# For a library target
add_library(mylib src/mylib.cpp)

# Link the logger interface (public)
target_link_libraries(mylib PUBLIC myproject::myproject_logger)

# Link the logger implementation (private)
target_link_libraries(mylib PRIVATE myproject::myproject_logger_impl)
```

### RAPIDS CPM Integration

For projects using RAPIDS CPM (CMake Package Manager), you can integrate rocmds\_logger as follows (see `examples/rapids_cpm/CMakeLists.txt`):

```cmake
cmake_minimum_required(VERSION 3.26.4)

# Include RAPIDS CMake utilities
include(/my_rapids_proj_folder/rapids_config.cmake)
include(rapids-cpm)
include(rapids-export)
include(rapids-hip)

# Initialize HIP architectures for the project
rapids_hip_init_architectures(MY_PROJ)
list(APPEND lang_list "HIP")

project(
  MY_PROJ
  VERSION "1.0.0"
  LANGUAGES ${lang_list} CXX
)

# Initialize CPM
rapids_cpm_init()

# Include and use RAPIDS_LOGGER via CPM
include(${rapids-cmake-dir}/cpm/rocmds_logger.cmake)
rapids_cpm_rapids_logger()

# Generate logger for this project
rapids_make_logger(my_proj
    LOGGER_HEADER_DIR include/my_proj/core
    EXPORT_SET my_proj-exports
    LOGGER_DEFAULT_LEVEL INFO
    LOGGER_MACRO_PREFIX MY_PROJ
)

# Create executable
add_executable(rapids_cpm_example main.cpp)

# Link logger targets
target_link_libraries(rapids_cpm_example
        PUBLIC my_proj::my_proj_logger
        PRIVATE my_proj::my_proj_logger_impl
)
```

## Generated Logger Usage

### C++ Code Example

Once you've set up the CMake configuration, you can use the generated logger in your C++ code (see `examples/rapids_cpm/main.cpp`):

```cpp
#include <my_proj/core/logger.hpp>

int main() {
    // Get the default logger
    auto& logger = my_proj::default_logger();
    
    // Log at different levels using project-specific macros
    MY_PROJ_LOG_INFO("RAPIDS CPM example application started");
    MY_PROJ_LOG_WARN("This is a warning message from rapids-cpm example");
    MY_PROJ_LOG_ERROR("An error occurred with code: %d", 42);
    
    // Set runtime log level
    logger.set_level(my_proj::level_enum::debug);
    MY_PROJ_LOG_DEBUG("Debug message now visible in rapids-cpm example");
    MY_PROJ_LOG_INFO("RAPIDS CPM example application finished");
    
    return 0;
}
```

### Compile-time Log Level Control

Control which log levels are compiled into your binary:

```cmake
# Only compile INFO level and above
target_compile_definitions(mylib PRIVATE 
    MY_PROJECT_LOG_ACTIVE_LEVEL=MY_PROJECT_LOG_LEVEL_INFO
)
```

### Runtime Log Level Control

The default runtime log level can be set during logger generation:

```cmake
rapids_make_logger(myproject
    LOGGER_DEFAULT_LEVEL DEBUG  # Available: TRACE, DEBUG, INFO, WARN, ERROR, CRITICAL, OFF
    LOGGER_HEADER_DIR include/myproject/core
    EXPORT_SET myproject-exports
)
```

Or changed at runtime:

```cpp
myproject::default_logger().set_level(myproject::level_enum::trace);
```

## Function Reference

### rapids\_make\_logger()

```cmake
rapids_make_logger(<logger_namespace>
    [EXPORT_SET <export-set-name>]
    [LOGGER_TARGET <logger-target-name>]
    [LOGGER_HEADER_DIR <header-dir>]
    [LOGGER_MACRO_PREFIX <macro-prefix>]
    [LOGGER_DEFAULT_LEVEL <logger-default-level>]
    [CMAKE_ALIAS_NAMESPACE <alias-namespace>]
)
```

**Parameters:**

* `logger_namespace` - The namespace for the generated logger (required)
* `EXPORT_SET` - CMake export set name for installation (optional but recommended)
* `LOGGER_TARGET` - Name of generated targets (default: `<namespace>_logger`)
* `LOGGER_HEADER_DIR` - Header installation directory (default: `include/<namespace>`, recommended to specify)
* `LOGGER_MACRO_PREFIX` - Prefix for log macros (default: uppercase namespace)
* `LOGGER_DEFAULT_LEVEL` - Default runtime log level (default: INFO)
* `CMAKE_ALIAS_NAMESPACE` - Namespace for CMake alias targets (default: same as logger\_namespace)

**Generated Targets:**

* `<namespace>::<target>` - Interface target with logger header
* `<namespace>::<target>_impl` - Implementation target to link privately

## API Reference

> The C++ surface mirrors `rapids_logger` (which wraps **spdlog**) but **format strings are printf-style and are formatted with `snprintf`/`vsnprintf`**, *not* spdlog’s `{}`/fmt-style formatter.

### Headers and Namespace

* Generated header (example):
  `#include <myproject/logger.hpp>` (path depends on `LOGGER_HEADER_DIR`)
* Primary namespace: `<namespace>` (e.g., `myproject` or `my_proj`)

### Logger Access

* `logger& <namespace>::default_logger();`
  Returns a reference to the singleton logger for your project.

* `void logger::set_level(<namespace>::level_enum lvl);`
  Changes runtime verbosity.

* `enum class <namespace>::level_enum { trace, debug, info, warn, error, critical, off };`

### Logging Macros

Macros are generated using `LOGGER_MACRO_PREFIX`. For a prefix `MY_PROJECT`, the following macros are available:

```cpp
MY_PROJECT_LOG_TRACE(const char* fmt, ...);
MY_PROJECT_LOG_DEBUG(const char* fmt, ...);
MY_PROJECT_LOG_INFO (const char* fmt, ...);
MY_PROJECT_LOG_WARN (const char* fmt, ...);
MY_PROJECT_LOG_ERROR(const char* fmt, ...);
MY_PROJECT_LOG_CRITICAL(const char* fmt, ...);
```

* **Formatting**: `fmt` is a **printf-style** format string; arguments are formatted using `snprintf`/`vsnprintf` before being emitted.

  * **Do not** use `{}` placeholders; use `%d`, `%s`, `%zu`, etc.
  * Example:

    ```cpp
    int code = 42;
    const std::string file = "dataset.parquet";
    MY_PROJECT_LOG_ERROR("failed to load %s (code=%d)", file.c_str(), code);
    ```

* **Cost when disabled**:

  * Compile-time: define `MY_PROJECT_LOG_ACTIVE_LEVEL` to compile out higher-verbosity calls:

    ```cpp
    // Compile out TRACE and DEBUG
    // Allowed values typically mirror: MY_PROJECT_LOG_LEVEL_TRACE/DEBUG/INFO/WARN/ERROR/CRITICAL/OFF
    ```
  * Runtime: messages below the current `level_enum` are dropped; implementations usually guard formatting with level checks to minimize overhead.

### printf/snprintf formatting cheat sheet

> **Important:** Log message formatting is **printf-style** using `snprintf`/`vsnprintf` (C locale), **not** `{}`/fmt-style. Always pass arguments that match the format specifiers.

#### Quick reference

| Category                   | Specifier(s)                                       | Notes / Examples                                                                              |
| -------------------------- | -------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **C string**               | `%s`                                               | Pass `c_str()` for `std::string`. Example: `MY_PROJECT_LOG_INFO("file=%s", path.c_str());`    |
| **Character**              | `%c`                                               | Single byte character.                                                                        |
| **Signed int**             | `%d`, `%i`                                         | `int`. Example: `MY_PROJECT_LOG_INFO("code=%d", code);`                                       |
| **Unsigned int**           | `%u`                                               | `unsigned int`.                                                                               |
| **Hex / octal**            | `%x`, `%X`, `%o`, `%#x`                            | `%#x` adds `0x` prefix.                                                                       |
| **long / long long**       | `%ld` / `%lld` (signed), `%lu` / `%llu` (unsigned) | Match the exact type.                                                                         |
| **Fixed-width ints**       | `PRId64`, `PRIu64`, etc.                           | Include `<cinttypes>` (or `<inttypes.h>`). Example: `MY_PROJECT_LOG_INFO("id=%" PRIu64, id);` |
| **`size_t` / `ptrdiff_t`** | `%zu` / `%td`                                      | Example: `MY_PROJECT_LOG_INFO("bytes=%zu", nbytes);`                                          |
| **Pointer**                | `%p`                                               | Cast to `void*` if needed: `MY_PROJECT_LOG_DEBUG("ptr=%p", (void*)p);`                        |
| **Float / double**         | `%f`, `%e`, `%g`                                   | Precision via `%.3f` etc.                                                                     |
| **Boolean**                | `%d` or `%s`                                       | Either cast to `int` or map to `"true"/"false"`.                                              |
| **Percent sign**           | `%%`                                               | Example: `MY_PROJECT_LOG_INFO("progress=%d%%", pct);`                                         |

#### Safer string handling

* `std::string`: use `.c_str()` with `%s`.
* `std::string_view` / non-null-terminated ranges: use precision with `%.*s`:

  ```cpp
  std::string_view name = get_name();
  MY_PROJECT_LOG_INFO("name=%.*s", (int)name.size(), name.data());
  ```

#### Width, precision, and alignment (handy patterns)

```cpp
// pad width 8 with spaces
MY_PROJECT_LOG_INFO("|%8d|", 42);          // -> "|      42|"

// zero-pad width 8
MY_PROJECT_LOG_INFO("|%08d|", 42);         // -> "|00000042|"

// left-align width 8
MY_PROJECT_LOG_INFO("|%-8s|", "ok");       // -> "|ok      |"

// floating precision
MY_PROJECT_LOG_INFO("loss=%.6f", loss);    // 6 digits after decimal

// hex with 0x prefix
MY_PROJECT_LOG_INFO("addr=%#x", addr);
```

#### 64-bit and platform-portable integers

Use the `<cinttypes>` macros to be portable across platforms/ABIs:

```cpp
#include <cinttypes>

uint64_t id = 9007199254740991ULL;
int64_t  off = -123;

MY_PROJECT_LOG_INFO("id=%" PRIu64 " off=%" PRId64, id, off);
```

#### Do / Don’t

* **Do** keep the format string **literal** or controlled:

  ```cpp
  // Safe when user_input may contain % characters
  MY_PROJECT_LOG_INFO("%s", user_input.c_str());
  ```
* **Don’t** pass untrusted data as the format string:

  ```cpp
  // Not safe: user_input is treated as a format string
  // MY_PROJECT_LOG_INFO(user_input.c_str());
  ```

#### Common pitfalls & notes

* **Match types exactly.** Passing a `long long` to `%d` (expects `int`) is undefined behavior.
* **`size_t` vs `%zu`.** `%zu` is standard; if you’re on very old toolchains lacking `%zu`, prefer casting to a known width or use the appropriate PRI macros. Modern compilers support `%zu`.
* **Truncation.** Very long messages may be truncated by the internal buffer (implementation-defined). Prefer concise messages or split across lines.
* **Locale.** `snprintf` uses the C locale (e.g., decimal point is `.`); thousands separators aren’t inserted automatically.


### Compile-Time Controls

* **Active level** (remove code above a threshold):

  ```cmake
  # Keep INFO+; remove DEBUG/TRACE at compile time
  target_compile_definitions(your_target PRIVATE
    MY_PROJECT_LOG_ACTIVE_LEVEL=MY_PROJECT_LOG_LEVEL_INFO)
  ```

* **Header-only interface vs implementation TU**:
  Link `<namespace>::<target>` **publicly** and `<namespace>::<target>_impl` **privately** to avoid leaking spdlog/fmt symbols.

### Runtime Controls

* **Change level**:

  ```cpp
  <namespace>::default_logger().set_level(<namespace>::level_enum::debug);
  ```

* **Environment / pattern**:
  Message **formatting of arguments** uses `snprintf`; any log pattern (timestamps, level tags) is handled by the underlying implementation and **does not** alter how your arguments are formatted.

### Error Handling & Truncation

* Internal formatting uses `vsnprintf` into a bounded buffer. If a message is truncated, it will be emitted in truncated form (exact behavior depends on implementation). Keep messages concise or split them across lines for very large payloads.

### Minimal End-to-End Example

```cpp
#include <myproject/logger.hpp>
#include <inttypes.h>

int main() {
    auto& log = myproject::default_logger();
    log.set_level(myproject::level_enum::info);

    const char* phase = "bootstrap";
    size_t loaded = 1024;
    uint64_t id = 1234567890123456789ULL;

    MY_PROJECT_LOG_INFO("phase=%s loaded=%zu id=%" PRIu64, phase, loaded, id);
    MY_PROJECT_LOG_DEBUG("this won't show at INFO");
}
```

## Advanced Usage

### Custom Header Location

```cmake
rapids_make_logger(myproject
    LOGGER_HEADER_DIR "include/custom/path"
    LOGGER_TARGET "custom_logger"
)
```

### Multiple Loggers

You can generate multiple loggers for different components:

```cmake
# Core library logger
rapids_make_logger(myproject_core
    EXPORT_SET myproject-exports
    LOGGER_DEFAULT_LEVEL INFO
)

# Plugin system logger  
rapids_make_logger(myproject_plugins
    EXPORT_SET myproject-exports
    LOGGER_DEFAULT_LEVEL WARN
)
```

### Integration with Package Exports

```cmake
# Export targets for installation
install(EXPORT myproject-exports
    FILE myproject-targets.cmake
    NAMESPACE myproject::
    DESTINATION lib/cmake/myproject
)
```

## Important Notes

### Header-Only vs Implementation

* **Logger Interface**: Link publicly to share the logger interface with consumers
* **Logger Implementation**: Link privately to avoid exposing spdlog symbols

### Symbol Isolation

The generated logger uses the PImpl idiom to ensure:

* No spdlog symbols are exposed in your public API
* Multiple projects can use different spdlog versions safely
* Clean separation between interface and implementation

### C++ Standard Requirement

rocmds\_logger requires **C++17** or later. Make sure your project sets:

```cmake
target_compile_features(your_target PUBLIC cxx_std_17)
```

