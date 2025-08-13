# MIT License
#
# Copyright (c) 2025 Advanced Micro Devices, Inc.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.26.4)

# Include RAPIDS CMake utilities
include(rapids_config.cmake)
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
