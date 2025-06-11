# =============================================================================
# Copyright (c) 2024, NVIDIA CORPORATION.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except
# in compliance with the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software distributed under the License
# is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
# or implied. See the License for the specific language governing permissions and limitations under
# the License.
# =============================================================================

# Modifications Copyright (c) 2025 Advanced Micro Devices, Inc. Permission is hereby granted,
# free of charge, to any person obtaining a copy of this software and associated documentation files
# (the "Software"), to deal in the Software without restriction, including without limitation the
# rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the
# Software, and to permit persons to whom the Software is furnished to do so, subject to the
# following conditions: The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software. THE SOFTWARE IS PROVIDED "AS IS", WITHOUT
# WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
# MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN
# ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR
# THE USE OR OTHER DEALINGS IN THE SOFTWARE.

# cmake-lint: disable=R0915

include_guard(GLOBAL)

cmake_minimum_required(VERSION 3.26.4 FATAL_ERROR)

include(rapids_config.cmake)

include(rapids-cmake)
include(rapids-cpm)

project(
  RAPIDS_LOGGER
  # TODO: Determine appropriate versioning.
  VERSION 0.0.1
  LANGUAGES CXX)

rapids_cmake_build_type(Release)

rapids_cpm_init()

include("rapids_make_logger.cmake")

include(GNUInstallDirs)

install(FILES
        "${CMAKE_CURRENT_LIST_DIR}/rapids_make_logger.cmake"
        "${CMAKE_CURRENT_LIST_DIR}/logger.hpp.in"
        "${CMAKE_CURRENT_LIST_DIR}/logger.cpp.in"
        "${CMAKE_CURRENT_LIST_DIR}/logger_impl.hpp.in"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)

include(CMakePackageConfigHelpers)


string(TOLOWER "${PROJECT_NAME}" _proj_lc)
set(_ver_file "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake")

write_basic_package_version_file(
        "${_ver_file}"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY AnyNewerVersion
)

# Install both variants without running write_basic twice:
install(FILES
        "${_ver_file}"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
        RENAME "${PROJECT_NAME}-config-version.cmake"
)
install(FILES
        "${_ver_file}"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
        RENAME "${_proj_lc}-config-version.cmake"
)

configure_package_config_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}-config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
        INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)

install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
        RENAME "${PROJECT_NAME}Config.cmake"
)
install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
        RENAME "${_proj_lc}-config.cmake"
)

include(InstallRequiredSystemLibraries)
set(CPACK_GENERATOR "DEB;TGZ;RPM")
set(CPACK_PACKAGE_NAME "${PROJECT_NAME}")
set(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")
set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Grant Pinkert gpinkert@amd.com")
include(CPack)
