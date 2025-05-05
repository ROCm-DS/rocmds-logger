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
