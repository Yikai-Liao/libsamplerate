cmake_minimum_required(VERSION 3.5)

project(libsamplerate VERSION 0.2.2 LANGUAGES C)

# Configuration

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  set(IS_ROOT_PROJECT ON)
else()
  set(IS_ROOT_PROJECT OFF)
endif()

option(LIBSAMPLERATE_EXAMPLES "Enable to generate examples" ${IS_ROOT_PROJECT})
option(LIBSAMPLERATE_INSTALL "Enable to add install directives" ${IS_ROOT_PROJECT})

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED TRUE)

include(TestBigEndian)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckLibraryExists)
include(CheckSymbolExists)
include(GNUInstallDirs)

include(CTest)

if(DEFINED LIBSAMPLERATE_TESTS)
  message(DEPRECATION "LIBSAMPLERATE_TESTS option deprecated, use BUILD_TESTING option instead.")
  set(BUILD_TESTING ${LIBSAMPLERATE_TESTS})
endif()

add_definitions(-DHAVE_CONFIG_H)
include_directories(${PROJECT_BINARY_DIR})
if(MSVC)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

if(NOT (WIN32 OR APPLE OR CYGWIN OR HAIKU OR BEOS))
  find_library(MATH_LIBRARY m)
  if(MATH_LIBRARY)
    set(LIBM_REQUIRED 1)
    if(LIBM_REQUIRED)
      list(APPEND CMAKE_REQUIRED_LIBRARIES m)
      if(LIBM_REQUIRED)
        link_libraries(${MATH_LIBRARY})
      endif()
    endif()
  endif()
endif()

option(LIBSAMPLERATE_SSE2_LRINT "Implement lrintf using SSE2 on x86 CPUs if possible" OFF)
if(LIBSAMPLERATE_SSE2_LRINT)
  add_definitions(-DENABLE_SSE2_LRINT)
endif()

if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID MATCHES "Clang")
  option(LIBSAMPLERATE_ENABLE_SANITIZERS "Enable ASAN and UBSAN" OFF)

  if(LIBSAMPLERATE_ENABLE_SANITIZERS)
    # Use ASAN and UBSAN, make it fail on any error, improve stack traces
    set(sanitizer_flags -fsanitize=address,undefined -fno-sanitize-recover=all -fno-omit-frame-pointer)

    add_compile_options(${sanitizer_flags})
    string(REPLACE ";" " " sanitizer_flags "${sanitizer_flags}")
    string(APPEND CMAKE_EXE_LINKER_FLAGS " ${sanitizer_flags}")
    string(APPEND CMAKE_MODULE_LINKER_FLAGS " ${sanitizer_flags}")
    string(APPEND CMAKE_SHARED_LINKER_FLAGS " ${sanitizer_flags}")
  endif()
endif()

test_big_endian(CPU_IS_BIG_ENDIAN)
if(CPU_IS_BIG_ENDIAN)
  set(CPU_IS_LITTLE_ENDIAN 0)
else()
  set(CPU_IS_LITTLE_ENDIAN 1)
endif()

check_include_file(stdbool.h HAVE_STDBOOL_H)
check_include_file(unistd.h HAVE_UNISTD_H)
check_include_file(immintrin.h HAVE_IMMINTRIN_H)

# For examples and tests

find_package(PkgConfig)

if(LIBSAMPLERATE_EXAMPLES OR BUILD_TESTING)
  if((NOT VCPKG_TOOLCHAIN) AND PKG_CONFIG_FOUND AND (NOT CMAKE_VERSION VERSION_LESS 3.6))
    pkg_check_modules(SndFile sndfile IMPORTED_TARGET)
    if(SndFile_FOUND)
      set(SNDFILE_TARGET PkgConfig::SndFile)
    endif()
  else()
    find_package(SndFile)
    if(SndFile_FOUND)
      set(SNDFILE_TARGET SndFile::sndfile)
    endif()
  endif()

  set(HAVE_SNDFILE ${SndFile_FOUND})
endif()

# SampleRate library

add_subdirectory(src)

# Tests

# BUILD_TESTING is declared by CTest module and is ON by default
if(BUILD_TESTING)
  add_subdirectory(tests)
endif()

# Examples

if(LIBSAMPLERATE_EXAMPLES)
  add_subdirectory(examples)
endif()

configure_file(config.h.cmake config.h)

if(LIBSAMPLERATE_INSTALL)
  add_subdirectory(docs)
endif()

# Packaging support

include(CPack)
