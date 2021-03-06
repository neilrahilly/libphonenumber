# Copyright (C) 2011 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Author: Philippe Liard

cmake_minimum_required (VERSION 2.8)

project (libphonenumber)

# Helper functions dealing with finding libraries and programs this library
# depends on.

function (print_error DESCRIPTION FILE)
  message (FATAL_ERROR
    "Can't find ${DESCRIPTION}: can't locate ${FILE}. Please read the README.")
endfunction ()

# Find a library. If it has not been found, stop CMake with a fatal error
# message.
function (find_required_library NAME HEADER LIBRARY DESCRIPTION)
  # Check the header.
  find_path (${NAME}_INCLUDE_DIR ${HEADER})
  set (INCLUDE_DIR ${${NAME}_INCLUDE_DIR})

  if (${INCLUDE_DIR} STREQUAL "${INCLUDE_DIR}-NOTFOUND")
    print_error (${DESCRIPTION} ${HEADER})
  endif ()
  include_directories (${INCLUDE_DIR})
  # Check the binary.
  find_library (${NAME}_LIB ${LIBRARY})
  set (LIB ${NAME}_LIB)

  if (${LIB} STREQUAL "${LIB}-NOTFOUND")
    print_error (${DESCRIPTION} ${LIBRARY})
  endif ()
endfunction (find_required_library)

# Check the library version (if pkg-config available).
find_package (PkgConfig)
function (check_library_version NAME LIBRARY VERSION)
  if (PKG_CONFIG_EXECUTABLE)
    pkg_check_modules (NAME REQUIRED ${LIBRARY}>=${VERSION})
  endif (PKG_CONFIG_EXECUTABLE)
endfunction ()

# Find a program. If it has not been found, stop CMake with a fatal error
# message.
function (find_required_program NAME FILENAME DESCRIPTION)
  find_program (${NAME}_BIN NAMES ${FILENAME})

  if (${NAME}_BIN STREQUAL "${${NAME}_BIN}-NOTFOUND")
    print_error (${DESCRIPTION} ${FILENAME})
  endif ()
endfunction (find_required_program)

# Options that can be passed to CMake using 'cmake -DKEY=VALUE'.
option ("USE_LITE_METADATA" "Use lite metadata" "OFF")
option ("USE_RE2" "Use RE2 instead of ICU" "OFF")

# Find all the required libraries and programs.
find_required_library (GTEST gtest/gtest.h gtest "Google Test framework")

if (${USE_RE2} STREQUAL "ON")
  find_required_library (RE2 re2/re2.h re2 "Google RE2")
endif ()

find_required_library (PROTOBUF google/protobuf/message_lite.h protobuf
                       "Google Protocol Buffers")
check_library_version (PC_PROTOBUF protobuf 2.4)

find_required_library (ICU_UC unicode/uchar.h icuuc "ICU")
check_library_version (PC_ICU_UC icuuc 4.4)
set (ICU_INCLUDE_DIR ${ICU_UC_INCLUDE_DIR})
set (ICU_LIB ${ICU_UC_LIB})
# If ICU regexp engine is used, use icui18n as well.
if (${USE_RE2} STREQUAL "OFF")
  find_required_library (ICU_I18N unicode/regex.h icui18n "ICU")
  check_library_version (PC_ICU_I18N icui18n 4.4)
  list (APPEND ICU_INCLUDE_DIR ${ICU_I18N_INCLUDE_DIR})
  list (APPEND ICU_LIB ${ICU_I18N_LIB})
endif ()

find_required_program (PROTOC protoc
                       "Google Protocol Buffers compiler (protoc)")

find_required_program (JAVA java
                       "Java Runtime Environment")

if (APPLE)
  FIND_LIBRARY (COREFOUNDATION_LIB CoreFoundation)
  FIND_LIBRARY (FOUNDATION_LIB Foundation)
endif ()

INCLUDE (CheckIncludeFileCXX)
CHECK_INCLUDE_FILE_CXX ("tr1/unordered_map" HAVE_CXX_TR1_UNORDERED_MAP)

if (HAVE_CXX_TR1_UNORDERED_MAP)
  add_definitions ("-DUSE_TR1_UNORDERED_MAP")
else ()
  CHECK_INCLUDE_FILE_CXX (hash_map HAVE_CXX_HASH_MAP)
  if (HAVE_CXX_HASH_MAP)
    add_definitions ("-DUSE_HASH_MAP")
  else ()
    print_error ("C++ map class" "tr1/unordered_map or hash_map")
  endif ()
endif ()

# Add protoc (Protocol Buffers compiler) target.
set (RESOURCES_DIR "${CMAKE_SOURCE_DIR}/../resources")

set (
  PROTOBUF_SOURCES "${RESOURCES_DIR}/phonemetadata.proto"
                   "${RESOURCES_DIR}/phonenumber.proto"
)

set (
  PROTOBUF_OUTPUT "${CMAKE_SOURCE_DIR}/src/phonemetadata.pb.cc"
                  "${CMAKE_SOURCE_DIR}/src/phonemetadata.pb.h"
                  "${CMAKE_SOURCE_DIR}/src/phonenumber.pb.cc"
                  "${CMAKE_SOURCE_DIR}/src/phonenumber.pb.h"
)

add_custom_command (
  COMMAND ${PROTOC_BIN} --cpp_out=${CMAKE_SOURCE_DIR}/src
    --proto_path=${RESOURCES_DIR} ${PROTOBUF_SOURCES}

  OUTPUT ${PROTOBUF_OUTPUT}
  DEPENDS ${PROTOBUF_SOURCES}
)

add_custom_target (
  generate-sources

  DEPENDS ${PROTOBUF_OUTPUT}
  COMMENT "Generating Protocol Buffers code"
)

# Platform independent sources.
set (
  SOURCES
  "src/base/at_exit.cc"
  "src/base/lazy_instance.cc"
  "src/base/string_piece.cc"
  "src/base/synchronization/lock.cc"
  "src/base/threading/thread_restrictions.cc"
  "src/default_logger.cc"
  "src/logger_adapter.cc"
  "src/metadata.h"          # Generated by build tools.
  "src/phonemetadata.pb.cc" # Generated by Protocol Buffers.
  "src/phonenumber.cc"
  "src/phonenumber.pb.cc"   # Generated by Protocol Buffers.
  "src/phonenumberutil.cc"
  "src/regexp_cache.cc"
  "src/stringutil.cc"
  "src/utf/rune.c"
  "src/utf/unicodetext.cc"
  "src/utf/unilib.cc"
)

# Add regexp engine sources. ICU is used by default.
if (${USE_RE2} STREQUAL "ON")
  list (APPEND SOURCES "src/regexp_adapter_re2.cc")
else ()
  list (APPEND SOURCES "src/regexp_adapter_icu.cc")
endif ()

# Library sources excluding the metadata files, since special metadata is used
# for unit-testing.
set (TESTING_LIBRARY_SOURCES ${SOURCES})

# Add metadata code generation targets.

# This function is invoked to create metadata, test metadata and lite metadata
# code generation targets.
function (add_metadata_gen_target TARGET_NAME
                                  XML_FILE
                                  METADATA_TYPE)
  set (METADATA_SOURCE_DIR "${CMAKE_SOURCE_DIR}/src")
  set (GEN_OUTPUT "${METADATA_SOURCE_DIR}/${METADATA_TYPE}.cc"
                  "${METADATA_SOURCE_DIR}/metadata.h")
  set (JAR_PATH "${CMAKE_SOURCE_DIR}/../tools/java/cpp-build/target")
  set (JAR_PATH "${JAR_PATH}/cpp-build-1.0-SNAPSHOT-jar-with-dependencies.jar")

  add_custom_command (
    COMMAND ${JAVA_BIN} -jar
      ${JAR_PATH} BuildMetadataCppFromXml ${XML_FILE} ${CMAKE_SOURCE_DIR}/src
      ${METADATA_TYPE}

    OUTPUT ${GEN_OUTPUT}
    DEPENDS ${XML_FILE}
  )
  add_custom_target (
    ${TARGET_NAME}
    DEPENDS ${GEN_OUTPUT}
    COMMENT "Generating Metadata code"
  )
endfunction (add_metadata_gen_target)

if (${USE_LITE_METADATA} STREQUAL "ON")
  # Add the lite metadata generation target.
  set (METADATA_TARGET "generate-lite-metadata")
  add_metadata_gen_target (
    ${METADATA_TARGET}
    "${RESOURCES_DIR}/PhoneNumberMetaData.xml"
    "lite_metadata"
  )
  list (APPEND SOURCES "src/lite_metadata.cc")
else ()
  # Add the metadata generation target.
  set (METADATA_TARGET "generate-metadata")
  add_metadata_gen_target (
    ${METADATA_TARGET}
    "${RESOURCES_DIR}/PhoneNumberMetaData.xml"
    "metadata"
  )
  list (APPEND SOURCES "src/metadata.cc")
endif ()

# Add the test metadata generation target.
set (TEST_METADATA_TARGET "generate-test-metadata")
add_metadata_gen_target (
  ${TEST_METADATA_TARGET}
  "${RESOURCES_DIR}/PhoneNumberMetaDataForTesting.xml"
  "test_metadata"
)
list (APPEND TESTING_LIBRARY_SOURCES "src/test_metadata.cc")

# Platform-specific sources.
set (PLATFORM_SPECIFIC_SOURCES "")

if (UNIX)
  if (CMAKE_COMPILER_IS_GNUCXX)
    add_definitions ("-Wall -Wextra -Werror")

    # The next flags are needed by base/ source files to compile low level code
    # needed by Singleton.
    add_definitions ("-DCOMPILER_GCC -DOS_POSIX")

    if (APPLE)
      add_definitions ("-DOS_MACOSX")
    else ()
      add_definitions ("-DOS_LINUX")
    endif ()

    if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86.*" OR
        ${CMAKE_SYSTEM_PROCESSOR} MATCHES "i386")
      add_definitions ("-DARCH_CPU_X86_FAMILY")
      # Add GCC specific sources.
      set (GCC_SOURCES "src/base/atomicops_internals_x86_gcc.cc")
      list (APPEND PLATFORM_SPECIFIC_SOURCES ${GCC_SOURCES})
    elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES ".*arm.*")
      add_definitions ("-DARCH_CPU_ARM_FAMILY")
    endif ()
  endif ()

  # Add POSIX specific sources.
  set (
    POSIX_SOURCES
    "src/base/synchronization/lock_impl_posix.cc"
    "src/base/threading/platform_thread_posix.cc"
    "src/base/threading/thread_local_posix.cc"
  )
  list (APPEND PLATFORM_SPECIFIC_SOURCES ${POSIX_SOURCES})

  if (APPLE)
    set (APPLE_SOURCES "src/base/threading/platform_thread_mac.mm")
    list (APPEND PLATFORM_SPECIFIC_SOURCES ${APPLE_SOURCES})
  endif ()
else (WIN32)
  # TODO: add Windows support (define COMPILER_MSVC, OS_WIN).
  set (
    WIN32_SOURCES
    "src/base/synchronization/lock_impl_win.cc"
    "src/base/threading/platform_thread_win.cc"
    "src/base/threading/thread_local_win.cc"
  )
  list (APPEND PLATFORM_SPECIFIC_SOURCES ${WIN32_SOURCES})
  # TODO: Windows specific flags.
endif ()

# Add platform-specific sources to both versions of phonenumber library (the one
# used by the client and the one used for testing).
list (APPEND SOURCES ${PLATFORM_SPECIFIC_SOURCES})
list (APPEND TESTING_LIBRARY_SOURCES ${PLATFORM_SPECIFIC_SOURCES})

include_directories ("src")
include_directories (".")

add_library (phonenumber STATIC ${SOURCES})
add_dependencies (phonenumber generate-sources ${METADATA_TARGET})

set (LIBRARY_DEPS ${PROTOBUF_LIB} ${ICU_LIB})

if (${USE_RE2} STREQUAL "ON")
  list (APPEND LIBRARY_DEPS ${RE2_LIB})
endif ()

if (APPLE)
  list (APPEND LIBRARY_DEPS ${COREFOUNDATION_LIB} ${FOUNDATION_LIB})
endif ()

target_link_libraries (phonenumber ${LIBRARY_DEPS})

# Build a specific library for testing purposes.
add_library (phonenumber_testing STATIC ${TESTING_LIBRARY_SOURCES})
target_link_libraries (phonenumber_testing ${LIBRARY_DEPS})
add_dependencies (phonenumber_testing generate-sources ${TEST_METADATA_TARGET})

set (TEST_SOURCES
  "src/phonenumberutil_test.cc"
  "src/regexp_adapter_test.cc"
  "src/regexp_cache_test.cc"
  "src/run_tests.cc"
  "src/stringutil_test.cc"
  "src/utf/unicodetext_test.cc"
)

# Build the testing binary.
add_executable (libphonenumber_test ${TEST_SOURCES})
target_link_libraries (
  libphonenumber_test
  phonenumber_testing ${GTEST_LIB} pthread
)
