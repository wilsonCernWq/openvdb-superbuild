## Based on OpenVKL Super Build

## Copyright 2019-2021 Intel Corporation
## SPDX-License-Identifier: Apache-2.0

## Global settings ##

cmake_minimum_required(VERSION 3.1)

set(CMAKE_MODULE_PATH
  ${CMAKE_MODULE_PATH}
  ${CMAKE_CURRENT_SOURCE_DIR}
)

if(NOT CMAKE_INSTALL_PREFIX)
  set(CMAKE_INSTALL_PREFIX
      "${CMAKE_BINARY_DIR}/install"
      CACHE STRING "Final install location." FORCE)
endif()

if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

## ======================================================================== ##
## Project setup.
## ======================================================================== ##

project(OpenVDB_superbuild)

include(ExternalProject)
include(GNUInstallDirs)
include(ProcessorCount)

## Setup top-level variables ##

set(installDir ${CMAKE_INSTALL_PREFIX})

get_filename_component(INSTALL_DIR_ABSOLUTE
  ${installDir} ABSOLUTE BASE_DIR ${CMAKE_CURRENT_BINARY_DIR})

ProcessorCount(PROCESSOR_COUNT)

if(NOT PROCESSOR_COUNT EQUAL 0)
  set(BUILD_JOBS ${PROCESSOR_COUNT} CACHE STRING "Number of build jobs '-j <n>'")
else()
  set(BUILD_JOBS 4 CACHE STRING "Number of build jobs '-j <n>'")
endif()

if (${CMAKE_VERSION} VERSION_GREATER 3.11.4)
  set(PARALLEL_JOBS_OPTS -j ${BUILD_JOBS})
endif()

set(DEFAULT_BUILD_COMMAND ${CMAKE_COMMAND} --build . --config Release ${PARALLEL_JOBS_OPTS})

# We must replace separators so that ExternalProject_Add can work with the
# paths we generate.
macro(add_to_prefix_path PATH)
  list(APPEND CMAKE_PREFIX_PATH ${PATH})
  string(REPLACE ";" "|" CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH}")
endmacro()

## ======================================================================== ##
## Superbuild options.
## ======================================================================== ##

option(INSTALL_IN_SEPARATE_DIRECTORIES
  "Install libraries into their own directories under CMAKE_INSTALL_PREFIX"
  OFF
)

## ======================================================================== ##
## Build dependencies.
##
## Use the BUILD_* options to enable or disable building in superbuild. If 
## These options are OFF, then required dependencies must be available in the 
## environment.
##
## For most of the dependencies, you may override the *_URL and *_HASH
## variables to select different versions.
## ======================================================================== ##

option(BUILD_TBB "Build Intel Threading Building Blocks or search in environment?" OFF)
if (BUILD_TBB)
  set(TBB_VERSION "2021.1.1" CACHE STRING "TBB version to download")
  if (TBB_VERSION VERSION_LESS 2021)
    set(TBB_BASE_URL "https://github.com/oneapi-src/oneTBB/releases/download/v${TBB_VERSION}/tbb-${TBB_VERSION}")
  else()
    set(TBB_BASE_URL "https://github.com/oneapi-src/oneTBB/releases/download/v${TBB_VERSION}/oneapi-tbb-${TBB_VERSION}")
  endif()
  if (APPLE)
    set(_TBB_URL "${TBB_BASE_URL}-mac.tgz")
    set(_TBB_HASH "c17ce0773401acfe17f4f2b6c6fb798694daa5763c0d13e2f20ba3bc8f27d339")
    set(TBB_LIB_SUBDIR "")
  elseif(WIN32)
    set(_TBB_URL "${TBB_BASE_URL}-win.zip")
    set(_TBB_HASH "544e61a490ceafd6756f000926850685ccaf59dbdf01ec5f2bef4855a3b50974")
    set(TBB_LIB_SUBDIR "intel64/vc14")
  else()
    set(_TBB_URL "${TBB_BASE_URL}-lin.tgz")
    set(_TBB_HASH "f89b7757ad97ca9f972072de86684b4486dd70ee8f2363e02ffbd275d34d34ac")
    set(TBB_LIB_SUBDIR "intel64/gcc4.8")
  endif()
  set(TBB_URL "${_TBB_URL}" CACHE STRING "URL of the TBB archive.")
  set(TBB_HASH "${_TBB_HASH}" CACHE STRING "SHA256 hash of the TBB archive.")
  include(dep_tbb)
endif()

# We always build or download OpenVDB and all its dependencies if OpenVDB is
# enabled, and to not attempt to find them in the environment.
set(BUILD_OPENVDB_DEFAULT ON)
mark_as_advanced(BUILD_OPENVDB_DEFAULT)
if (WIN32)
  set(BUILD_OPENVDB_DEFAULT OFF)
endif()

option(BUILD_OPENVDB "Enable OpenVDB to support loading .vdb files?" ${BUILD_OPENVDB_DEFAULT})
option(BUILD_BLOSC "Support loading compressed .vdb files using c-BLOSC? (Only used when BUILD_OPENVDB=ON)" ON)
if (BUILD_OPENVDB)
  if (WIN32)
    message(FATAL_ERROR "Open VKL superbuild currently does not support building on Windows.")
    set(BUILD_OPENVDB OFF)
  elseif (CMAKE_VERSION VERSION_LESS 3.12)
    message(WARNING "Disabling OpenVDB support because the cmake version is too low (3.12 is required)")
    set(BUILD_OPENVDB OFF)
  elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 6.3)
    message(WARNING "Disabling OpenVDB support because the gcc compiler version is too low (6.3 is required)")
    set(BUILD_OPENVDB OFF)
  elseif (CMAKE_CXX_COMPILER_ID STREQUAL "Intel" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.1)
    message(WARNING "Disabling OpenVDB support because the icc compiler version is too low (19.1 is required)")
    set(BUILD_OPENVDB OFF)
  elseif (BUILD_TBB AND (NOT TBB_VERSION VERSION_LESS 2021))
    message(WARNING "Disabling OpenVDB support because TBB 2021 is not supported")
    set(BUILD_OPENVDB OFF)
  else()
    set(ILMBASE_VERSION "2.4.1")
    set(ILMBASE_URL "https://github.com/AcademySoftwareFoundation/openexr/archive/v${ILMBASE_VERSION}.zip"
      CACHE STRING "URL of the IlmBase archive.")
    set(ILMBASE_HASH "fd828641897f4ceb9277b61e3add25c3306adcb4670355ac7081d78cf7ee8f5f"
      CACHE STRING "SHA256 hash of the IlmBase archive.")
    include(dep_ilmbase)

    set(ZLIB_VERSION "1.2.12")
    set(ZLIB_URL "https://www.zlib.net/zlib-${ZLIB_VERSION}.tar.gz"
      CACHE STRING "URL of the zlib archive.")
    set(ZLIB_HASH "91844808532e5ce316b3c010929493c0244f3d37593afd6de04f71821d5136d9" 
      CACHE STRING "SHA256 hash of the zlib archive.")
    include(dep_zlib)

    set(BOOST_VERSION "1.72.0")
    string(REPLACE "." "_" BOOST_FILE_BASE "${BOOST_VERSION}")
    set(BOOST_BASE_URL "https://boostorg.jfrog.io/artifactory/main/release/${BOOST_VERSION}/source/boost")
    set(_BOOST_URL "${BOOST_BASE_URL}_${BOOST_FILE_BASE}.tar.gz")
    set(_BOOST_HASH "c66e88d5786f2ca4dbebb14e06b566fb642a1a6947ad8cc9091f9f445134143f")
    set(BOOST_URL "${_BOOST_URL}" CACHE STRING "URL of the boost archive.")
    set(BOOST_HASH "${_BOOST_HASH}" CACHE STRING "SHA256 hash of the boost archive.")
    include(dep_boost)

    if (BUILD_BLOSC)
      set(BLOSC_VERSION "1.5.0")
      set(BLOSC_URL "https://github.com/Blosc/c-blosc/archive/v${BLOSC_VERSION}.zip"
        CACHE STRING "URL of the c-BLOSC archive.")
      set(BLOSC_HASH "fddd00ae717fc840c49509c367444f37bc50daea7d7d431b6c05e9570bc443a9" 
        CACHE STRING "SHA256 hash of the c-BLOSC archive.")
      include(dep_blosc)
    endif()

    set(OPENVDB_VERSION "8.2.0")
    set(OPENVDB_BASE_URL "https://github.com/AcademySoftwareFoundation/openvdb/archive")
    set(OPENVDB_URL "${OPENVDB_BASE_URL}/v${OPENVDB_VERSION}.zip"
      CACHE STRING "URL of the OpenVDB archive.")
    set(OPENVDB_HASH "e0095d8a5ba9a2b1d80ca15186ced2d1c3dfb03411b54fdff8bde0be7c1f1fa0"
      CACHE STRING "SHA256 hash of the OpenVDB archive.")
    include(dep_openvdb)
  endif()
endif()
