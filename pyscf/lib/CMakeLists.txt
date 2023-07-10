# Copyright 2014-2018 The PySCF Developers. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required (VERSION 3.14)

project (pyscf)

# Required to import External Project in a cross-compilation
# friendly way.
include(FetchContent)

# List of options
option(BUILD_LIBCINT "Using libcint for analytical gaussian integral" ON)
option(WITH_F12 "Compling F12 integrals" ON)
option(USE_QCINT "Using the qcint library (optimized for x86-64) for gaussian integral evaluation" OFF)
option(ENABLE_LIBXC "Using libxc for XC functional library" ON)
option(ENABLE_XCFUN "Using xcfun for XC functional library" ON)
option(BUILD_LIBXC "Download and build libxc library" ON)
option(BUILD_XCFUN "Download and build xcfun library" ON)
option(DISABLE_DFT "Disable building of dft library" OFF)

# Avoid warning about DOWNLOAD_EXTRACT_TIMESTAMP in CMake 3.24:
if (CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
  cmake_policy(SET CMP0135 NEW)
endif()

if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RELWITHDEBINFO)
endif()

# installation directory
if (NOT Python_INSTALL_DIR)
  if (SKBUILD)
    # If built with scikit-build-core, let it handle the installation
    set(Python_INSTALL_DIR "./pyscf/")
  else ()
    # Otherwise try to install in current python executable's setup
    set(Python_INSTALL_DIR ${Python3_SITEARCH}/pyscf/)
  endif ()
endif ()

# set(CMAKE_VERBOSE_MAKEFILE OFF)

if (CMAKE_COMPILER_IS_GNUCC) # Does it skip the link flag on old OsX?
  # TODO: If updating to minimum requirement cmake>=3.7, use
  # CMAKE_SHARED_LINKER_FLAGS_INIT to combine LDFLAGS options.
  # https://cmake.org/cmake/help/v3.7/variable/CMAKE_SHARED_LINKER_FLAGS_INIT.html
  # see also issue #661
  if(UNIX AND NOT APPLE AND NOT DEFINED ENV{LDFLAGS})
    set(CMAKE_SHARED_LINKER_FLAGS "-Wl,--no-as-needed")
  endif()
endif()
set(CMAKE_C_FLAGS "-Wall ${CMAKE_C_FLAGS}")

option(BUILD_MARCH_NATIVE "gcc flag -march=native" off)
if (BUILD_MARCH_NATIVE)
  include(CheckCCompilerFlag)
  CHECK_C_COMPILER_FLAG("-march=native" COMPILER_SUPPORTS_MARCH_NATIVE)
  if(COMPILER_SUPPORTS_MARCH_NATIVE)
    if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Intel")
      message("Add CFLAGS -march=native -unroll-aggressive")
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -unroll-aggressive -ipo")
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-math-errno")
    else()
      message("Add CFLAGS -march=native -ftree-vectorize")
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=native -funroll-loops -ftree-vectorize")
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-math-errno")
    endif()
  endif()
else()
  if ("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
    include(CheckCCompilerFlag)
    CHECK_C_COMPILER_FLAG("-msse3" COMPILER_SUPPORTS_SSE3)
    if(COMPILER_SUPPORTS_SSE3)
      # Avoids error "‘SIMDD’ undeclared here (not in a function)" in the qcint two-electron integral interface
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse3")
    endif()
  endif()
endif()


set(CMAKE_INCLUDE_CURRENT_DIR ON)

# Architecture specified cmake flags.  See also the templates in
# pyscf/lib/cmake_arch_config
if(EXISTS "${PROJECT_SOURCE_DIR}/cmake.arch.inc")
  include("${PROJECT_SOURCE_DIR}/cmake.arch.inc")
endif()

if (NOT BLAS_LIBRARIES)
  #enable_language(Fortran)
  find_package(BLAS)
  check_function_exists(ffsll HAVE_FFS)
endif()

if (NOT BLAS_LIBRARIES)
  message(FATAL_ERROR "A required library with BLAS API not found.")
else()
  message(STATUS "BLAS libraries: ${BLAS_LIBRARIES}")
endif()
# if unable to find mkl library, just create BLAS_LIBRARIES here, e.g.
# set(BLAS_LIBRARIES "-L/path/to/mkl/lib -lmkl_intel_lp64 -lmkl_sequential -lmkl_core -lpthread -lmkl_avx -lm")
# or
# set(BLAS_LIBRARIES "                  /path/to/mkl/lib/intel64/libmkl_intel_lp64.so")
# set(BLAS_LIBRARIES "${BLAS_LIBRARIES};/path/to/mkl/lib/intel64/libmkl_sequential.so")
# set(BLAS_LIBRARIES "${BLAS_LIBRARIES};/path/to/mkl/lib/intel64/libmkl_core.so")
# set(BLAS_LIBRARIES "${BLAS_LIBRARIES};/path/to/mkl/lib/intel64/libmkl_avx.so")
# set(BLAS_LIBRARIES "${BLAS_LIBRARIES};/path/to/mkl/lib/intel64/libmkl_def.so")
# set(BLAS_LIBRARIES "-Wl,-rpath=${MKLROOT}/lib/intel64/ ${BLAS_LIBRARIES}")
#
# These settings can be written in the cmake.arch.inc file. This config file
# will automatically load all settings specified in cmake.arch.inc
#

option(ENABLE_OPENMP "Compiling C extensions with openmp" ON)
set(OPENMP_C_PROPERTIES "")
if(ENABLE_OPENMP)
  find_package(OpenMP)
  if(OPENMP_FOUND)
    set(HAVE_OPENMP 1)
    set(OPENMP_C_PROPERTIES OpenMP::OpenMP_C)
  endif()
endif()


# See also https://gitlab.kitware.com/cmake/community/wikis/doc/cmake/RPATH-handling
# if (WIN32)
#   #?
# elseif (APPLE)
#   set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
#   set(CMAKE_INSTALL_RPATH "@loader_path;@loader_path/deps/lib;@loader_path/deps/lib64")
#   set(CMAKE_BUILD_RPATH "@loader_path;@loader_path/deps/lib;@loader_path/deps/lib64")
# else ()
#   set(CMAKE_SKIP_BUILD_RPATH  True)
#   set(CMAKE_BUILD_WITH_INSTALL_RPATH True)
#   set(CMAKE_INSTALL_RPATH "\$ORIGIN:\$ORIGIN/deps/lib:\$ORIGIN/deps/lib64")
# endif ()

##
# Fetch Content Block:
# The external dependencies must be fetched and their CMakeList.txt files
# declared before polluting the include namespace with more `include_directories`
# commands. 

# Set the download directory
set(FETCHCONTENT_BASE_DIR "${PROJECT_BINARY_DIR}/deps")

message("building or not CINT? " ${BUILD_LIBCINT})
if(BUILD_LIBCINT)
  set(LIBCINT_GIT https://github.com/sunqm/libcint.git) # libcint is a portable, cross-platform implementation
  if (USE_QCINT)
    set(LIBCINT_GIT https://github.com/sunqm/qcint.git) # qcint is an optimized implementation for x86-64 architecture
    if(NOT BUILD_MARCH_NATIVE)
      message(WARNING "The BUILD_MARCH_NATIVE option is not specified! qcint may not compile unless you explicitly pass compiler flags that turn on vectorization!")    
    endif()
  endif()

  FetchContent_Declare(libcint
    GIT_REPOSITORY ${LIBCINT_GIT}
    GIT_TAG v5.4.0
  )
  set(WITH_F12 ${WITH_F12})
  set(WITH_RANGE_COULOMB 1)
  set(WITH_FORTRAN 0)
  set(WITH_CINT2_INTERFACE 0)
  set(MIN_EXPCUTOFF 20)
  set(KEEP_GOING 1)
  set(BUILD_SHARED_LIBS 1)

  FetchContent_MakeAvailable(libcint)
endif() # BUILD_LIBCINT

if(ENABLE_LIBXC AND BUILD_LIBXC)
  FetchContent_Declare(libxc
    # GIT_REPOSITORY https://gitlab.com/libxc/libxc.git
    # GIT_TAG master
    URL https://gitlab.com/libxc/libxc/-/archive/6.1.0/libxc-6.1.0.tar.gz
  )
  set(BUILD_SHARED_LIBS 1)
  set(ENABLE_FORTRAN 0)
  set(DISABLE_KXC 0)
  set(DISABLE_LXC 0)
  set(ENABLE_XHOST ${BUILD_MARCH_NATIVE})

  FetchContent_MakeAvailable(libxc)
endif() # ENABLE_LIBXC

if(ENABLE_XCFUN AND BUILD_XCFUN)
  FetchContent_Declare(libxcfun
    #GIT_REPOSITORY https://github.com/sunqm/xcfun.git
    GIT_REPOSITORY https://github.com/fishjojo/xcfun.git
    # copy of v2.1.1, downgrade to cmake 3.5
    GIT_TAG cmake-3.5
  )
  set(BUILD_SHARED_LIBS 1)
  set(ENABLE_TESTALL 0)
  set(DISABLE_KXC 0)
  set(DISABLE_LXC 0)

  FetchContent_MakeAvailable(libxcfun)
endif()
##

include_directories(${PROJECT_BINARY_DIR}/deps/libxc-build/include)
include_directories(${PROJECT_BINARY_DIR}/deps/libxc-src/include)

include_directories(${PROJECT_SOURCE_DIR})
include_directories(${CMAKE_INSTALL_PREFIX}/include)

configure_file(
  "${PROJECT_SOURCE_DIR}/config.h.in"
  "${PROJECT_SOURCE_DIR}/config.h")
# to find config.h
include_directories("${PROJECT_BINARY_DIR}")

add_subdirectory(np_helper)
add_subdirectory(gto)
add_subdirectory(vhf)
add_subdirectory(ao2mo)
add_subdirectory(mcscf)
add_subdirectory(cc)
add_subdirectory(ri)
#add_subdirectory(localizer)
add_subdirectory(pbc)
add_subdirectory(agf2)

if(NOT DISABLE_DFT)
  add_subdirectory(dft)
endif()

# Overwrite CMAKE_C_CREATE_SHARED_LIBRARY in Modules/CMakeCInformation.cmake
# to remove the SONAME flag in the so file. The soname information causes
# dynamic link error when importing libcint library.
set(C_LINK_TEMPLATE "<CMAKE_C_COMPILER> <CMAKE_SHARED_LIBRARY_C_FLAGS> <LANGUAGE_COMPILE_FLAGS> <LINK_FLAGS> <CMAKE_SHARED_LIBRARY_CREATE_C_FLAGS> -o <TARGET> <OBJECTS> <LINK_LIBRARIES>")
set(CXX_LINK_TEMPLATE "<CMAKE_CXX_COMPILER> <CMAKE_SHARED_LIBRARY_CXX_FLAGS> <LANGUAGE_COMPILE_FLAGS> <LINK_FLAGS> <CMAKE_SHARED_LIBRARY_CREATE_CXX_FLAGS> -o <TARGET> <OBJECTS> <LINK_LIBRARIES>")

if(BUILD_LIBCINT)
  add_dependencies(cgto cint)
  add_dependencies(ao2mo cint)

  get_target_property(LIBCINT_INCLUDES cint INCLUDE_DIRECTORIES)
  foreach(SUB_LIB ao2mo cc dft cgto ri cvhf pbc)
    # this assumes that cgto is built as an imported dependency
    target_include_directories(${SUB_LIB} PRIVATE ${LIBCINT_INCLUDES})
    get_target_property(LIB_INC ${SUB_LIB} INCLUDE_DIRECTORIES)
  endforeach()

  # special dependency of pbc
  get_target_property(LIBCINT_SRC cint SOURCE_DIR)
  target_include_directories(pbc PRIVATE ${LIBCINT_INCLUDES} ${LIBCINT_SRC}/include)

endif() # BUILD_LIBCINT

if(NOT DISABLE_DFT)
  if(ENABLE_LIBXC AND BUILD_LIBXC)
    # this assumes that xc is built as an imported dependency
    get_target_property(LIBXC_INCLUDES xc INCLUDE_DIRECTORIES)
    target_include_directories(xc_itrf PRIVATE ${LIBXC_INCLUDES})

    add_dependencies(xc_itrf xc)
    add_dependencies(dft xc)
  endif() # ENABLE_LIBXC

  if(ENABLE_XCFUN AND BUILD_XCFUN)
    # this assumes that xc is built as an imported dependency
    get_target_property(LIBXCFUN_INCLUDES xcfun INCLUDE_DIRECTORIES)
    target_include_directories(xcfun_itrf PRIVATE ${LIBXCFUN_INCLUDES})

    add_dependencies(xcfun_itrf xcfun)
    add_dependencies(dft xcfun)
  endif() # ENABLE_XCFUN
endif() # DISABLE_DFT

if(EXISTS "${PROJECT_SOURCE_DIR}/cmake.user.inc")
  include("${PROJECT_SOURCE_DIR}/cmake.user.inc")
endif()

install(TARGETS 
  np_helper
  cvhf
  ao2mo
  mcscf
  cc
  ri
  pbc
  agf2
  cgto
  fci
 LIBRARY DESTINATION ./pyscf/lib)
if(BUILD_LIBCINT)
  install(TARGETS cint LIBRARY DESTINATION ./pyscf/lib)
endif()
if(ENABLE_LIBXC AND BUILD_LIBXC)
  install(TARGETS xc LIBRARY DESTINATION ./pyscf/lib)
endif()
if(ENABLE_XCFUN AND BUILD_XCFUN)
  install(TARGETS xcfun LIBRARY DESTINATION ./pyscf/lib)
endif()

if(NOT DISABLE_DFT)
  install(TARGETS dft LIBRARY DESTINATION ./pyscf/lib)
  if(ENABLE_LIBXC AND BUILD_LIBXC)
    install(TARGETS xc_itrf LIBRARY DESTINATION ./pyscf/lib)
  endif()
  if(ENABLE_XCFUN AND BUILD_XCFUN)
    install(TARGETS xcfun_itrf LIBRARY DESTINATION ./pyscf/lib)
  endif()
endif()
