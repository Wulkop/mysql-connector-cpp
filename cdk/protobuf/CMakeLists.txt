# Copyright (c) 2015, 2018, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License, version 2.0, as
# published by the Free Software Foundation.
#
# This program is also distributed with certain software (including
# but not limited to OpenSSL) that is licensed under separate terms,
# as designated in a particular file or component or in included license
# documentation.  The authors of MySQL hereby grant you an
# additional permission to link the program and your derivative works
# with the separately licensed software that they have included with
# MySQL.
#
# Without limiting anything contained in the foregoing, this file,
# which is part of MySQL Connector/C++, is also subject to the
# Universal FOSS Exception, version 1.0, a copy of which can be found at
# http://oss.oracle.com/licenses/universal-foss-exception.
#
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
# See the GNU General Public License, version 2.0, for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software Foundation, Inc.,
# 51 Franklin St, Fifth Floor, Boston, MA 02110-1301  USA

# The files in protobuf-2.6.1/ are unmodified versions of google source files.
# To save some space and time, we have remove directories which are not
# needed by MySQL:
#   protobuf-2.6.1/gtest/
#   protobuf-2.6.1/java/
#   protobuf-2.6.1/python/
#   protobuf-2.6.1/m4/
#   protobuf-2.6.1/vsprojects/

cmake_minimum_required(VERSION 2.8)
PROJECT(Protobuf)

#
# Create empty config.h as Protobuf code includes it.
# TODO: Put some meaningful configuration there.
#

file(WRITE ${PROJECT_BINARY_DIR}/config.h "\n")

#
# Do not export any symbols
#
# Note: The LIBPROTOBUF_EXPORTS macro should *not* be defined
#

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-fvisibility=hidden)
endif()


#
# Solaris compile and link settings
#

if(CMAKE_SYSTEM_NAME MATCHES "SunOS")

  add_definitions(-DSOLARIS_64BIT_ENABLED)

  if(CMAKE_SYSTEM_PROCESSOR MATCHES "sparc")
    add_definitions(-DGOOGLE_PROTOBUF_ARCH_SPARC)
  endif()

  add_compile_options(
    -m64 -std=c++11
    -xbuiltin=%all
    -xlibmil
    -xatomic=studio
  )

  foreach(type EXE SHARED MODULE)

    set(CMAKE_${type}_LINKER_FLAGS
      "-m64 -std=c++11 -lstdc++ -xatomic=studio -lCrunG3 -lgcc_s -lc"
    )

  endforeach()

endif()


#
# Configure static runtime library on Windows if requested
#

OPTION(STATIC_MSVCRT "Use static MSVC runtime library" OFF)

IF(WIN32 AND STATIC_MSVCRT)

message("Using static runtime library")

foreach(LANG C CXX)
  set(CMAKE_${LANG}_FLAGS "${CMAKE_${LANG}_FLAGS} /MT")
  foreach(TYPE RELEASE RELWITHDEBINFO MINSIZEREL)
    set(CMAKE_${LANG}_FLAGS_${TYPE} "${CMAKE_${LANG}_FLAGS_${TYPE}} /MT")
  endforeach()
  set(CMAKE_${LANG}_FLAGS_DEBUG "${CMAKE_${LANG}_FLAGS_DEBUG} /MTd")
endforeach(LANG)

ENDIF()


#
# Dependency on threads library.
#

INCLUDE(FindThreads)
IF(CMAKE_USE_PTHREADS_INIT)
  message("Using pthreads for protobuf code")
  ADD_DEFINITIONS(-DHAVE_PTHREAD)
ENDIF()


#
# Skip compression support for now.
#

set(HAVE_ZLIB 0)
ADD_DEFINITIONS(-DHAVE_ZLIB=${HAVE_ZLIB})


#
# Turn off some warning flags when compiling protobuf.
#

IF(UNIX)

  list(APPEND warnings_list "-Wno-sign-compare")
  list(APPEND warnings_list "-Wno-ignored-qualifiers")
  list(APPEND warnings_list "-Wno-return-type")
  list(APPEND warnings_list "-Wno-unused-function")
  list(APPEND warnings_list "-Wno-unused-local-typedefs")

  if(APPLE)

    # Our version of protobuf uses many constructs (mainly atomic ops)
    # which were depracated in OSX 10.12.
    # TODO: Upgrade bundled protobuf

    list(APPEND warnings_list -Wno-deprecated-declarations)

  endif()

  if(CMAKE_COMPILER_IS_GNUCXX)

    list(APPEND warnings_list "-Wno-unused-local-typedefs")
    list(APPEND warnings_list "-Wno-maybe-uninitialized")
    list(APPEND warnings_list "-Wno-unused-but-set-parameter")

  elseif(CMAKE_SYSTEM_NAME MATCHES "SunOS")

    # Option -erroff disables only warnings, actually. This option requires
    # a warning tag, which is printed if -errtags option is enabled.

    add_compile_options(
      -errtags=yes
      -erroff=anonnotype
    )

  endif()

ELSE(WIN32)

  list(APPEND warnings_list "/wd4018")
  list(APPEND warnings_list "/wd4996")
  list(APPEND warnings_list "/wd4244")
  list(APPEND warnings_list "/wd4715")
  list(APPEND warnings_list "/wd4065")
  list(APPEND warnings_list "/wd4800")
  list(APPEND warnings_list "/wd4355")
  list(APPEND warnings_list "/wd4267")

ENDIF()

add_compile_options(${warnings_list})


#
# Define Protobuf targets.
#


SET(PROTO_SRC_DIR
  "${PROJECT_SOURCE_DIR}/protobuf/src")

set(build_stamp "${PROJECT_BINARY_DIR}/build.stamp")


file(WRITE "${PROJECT_BINARY_DIR}/exports.cmake"
  "set(PROTOBUF_INCLUDE_DIR \"${PROTO_SRC_DIR}\")\n"
  "set(PROTOBUF_BUILD_STAMP \"${build_stamp}\")\n")

macro(pb_export)
  export(TARGETS ${ARGV} NAMESPACE pb_
         APPEND FILE "${PROJECT_BINARY_DIR}/exports.cmake")
endmacro()

INCLUDE_DIRECTORIES(
  ${PROJECT_BINARY_DIR}
  ${PROTO_SRC_DIR}
  ${ZLIB_INCLUDE_DIR}
)


SET(LIBPROTOBUF_LITE_SOURCES
  ${PROTO_SRC_DIR}/google/protobuf/arena.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/structurally_valid.cc
  
  ${PROTO_SRC_DIR}/google/protobuf/stubs/common.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/once.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/hash.h
  ${PROTO_SRC_DIR}/google/protobuf/stubs/map_util.h
  ${PROTO_SRC_DIR}/google/protobuf/stubs/shared_ptr.h
  ${PROTO_SRC_DIR}/google/protobuf/stubs/stringprintf.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/stringprintf.h
  ${PROTO_SRC_DIR}/google/protobuf/extension_set.cc
  ${PROTO_SRC_DIR}/google/protobuf/generated_message_util.cc
  ${PROTO_SRC_DIR}/google/protobuf/message_lite.cc
  ${PROTO_SRC_DIR}/google/protobuf/repeated_field.cc
  ${PROTO_SRC_DIR}/google/protobuf/wire_format_lite.cc
  ${PROTO_SRC_DIR}/google/protobuf/io/coded_stream.cc
  ${PROTO_SRC_DIR}/google/protobuf/io/coded_stream_inl.h
  ${PROTO_SRC_DIR}/google/protobuf/io/zero_copy_stream.cc
  ${PROTO_SRC_DIR}/google/protobuf/io/zero_copy_stream_impl_lite.cc
)

IF(MSVC)

  list(APPEND LIBPROTOBUF_LITE_SOURCES
       ${PROTO_SRC_DIR}/google/protobuf/stubs/atomicops_internals_x86_msvc.cc)

ELSEIF(NOT APPLE)

  # Platforms such as FreeBSD require this file even though we build with clang
  # Only on OSX the file is not used (and generates empty object file compiler
  # warnings)

  list(APPEND LIBPROTOBUF_LITE_SOURCES
       ${PROTO_SRC_DIR}/google/protobuf/stubs/atomicops_internals_x86_gcc.cc)

ENDIF()

ADD_LIBRARY(protobuf-lite STATIC ${LIBPROTOBUF_LITE_SOURCES})
SET_PROPERTY(TARGET protobuf-lite PROPERTY EXCLUDE_FROM_ALL TRUE)
pb_export(protobuf-lite)

SET(LIBPROTOBUF_SOURCES
  ${PROTO_SRC_DIR}/google/protobuf/arena.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/io_win32.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/stringpiece.cc
  ${PROTO_SRC_DIR}/google/protobuf/map_field.cc
  ${PROTO_SRC_DIR}/google/protobuf/any.cc
  ${PROTO_SRC_DIR}/google/protobuf/any.pb.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/status.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/common.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/int128.cc
  ${PROTO_SRC_DIR}/google/protobuf/implicit_weak_message.cc
  
  ${PROTO_SRC_DIR}/google/protobuf/stubs/strutil.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/strutil.h
  ${PROTO_SRC_DIR}/google/protobuf/stubs/substitute.cc
  ${PROTO_SRC_DIR}/google/protobuf/stubs/substitute.h
  ${PROTO_SRC_DIR}/google/protobuf/stubs/structurally_valid.cc
  ${PROTO_SRC_DIR}/google/protobuf/descriptor.cc
  ${PROTO_SRC_DIR}/google/protobuf/descriptor.pb.cc
  ${PROTO_SRC_DIR}/google/protobuf/descriptor_database.cc
  ${PROTO_SRC_DIR}/google/protobuf/dynamic_message.cc
  ${PROTO_SRC_DIR}/google/protobuf/extension_set_heavy.cc
  ${PROTO_SRC_DIR}/google/protobuf/generated_message_reflection.cc
  ${PROTO_SRC_DIR}/google/protobuf/message.cc
  ${PROTO_SRC_DIR}/google/protobuf/reflection_ops.cc
  ${PROTO_SRC_DIR}/google/protobuf/service.cc
  ${PROTO_SRC_DIR}/google/protobuf/text_format.cc
  ${PROTO_SRC_DIR}/google/protobuf/unknown_field_set.cc
  ${PROTO_SRC_DIR}/google/protobuf/wire_format.cc
  ${PROTO_SRC_DIR}/google/protobuf/io/printer.cc
  ${PROTO_SRC_DIR}/google/protobuf/io/strtod.cc
  ${PROTO_SRC_DIR}/google/protobuf/io/tokenizer.cc
  ${PROTO_SRC_DIR}/google/protobuf/io/zero_copy_stream_impl.cc
  ${PROTO_SRC_DIR}/google/protobuf/compiler/importer.cc
  ${PROTO_SRC_DIR}/google/protobuf/compiler/parser.cc
)

if(HAVE_ZLIB)
  LIST(APPEND LIBPROTOBUF_SOURCES
       ${PROTO_SRC_DIR}/google/protobuf/io/gzip_stream.cc)
endif()

ADD_LIBRARY(protobuf STATIC ${LIBPROTOBUF_SOURCES})
TARGET_LINK_LIBRARIES(protobuf protobuf-lite
  ${CMAKE_THREAD_LIBS_INIT} ${ZLIB_LIBRARY} ${LIBRT})

pb_export(protobuf)


set(LIBPROTOC_SOURCES

	${PROTO_SRC_DIR}/google/protobuf/compiler/code_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/command_line_interface.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_enum.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_enum_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_extension.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_file.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_helpers.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_map_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_message.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_message_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_padding_optimizer.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_primitive_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_service.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/cpp/cpp_string_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_doc_comment.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_enum.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_enum_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_field_base.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_generator.cc 
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_helpers.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_map_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_message.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_message_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_primitive_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_reflection_class.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_repeated_enum_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_repeated_message_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_repeated_primitive_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_source_generator_base.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/csharp/csharp_wrapper_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/importer.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_context.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_doc_comment.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_enum.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_enum_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_enum_field_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_enum_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_extension.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_extension_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_file.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_generator_factory.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_helpers.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_lazy_message_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_lazy_message_field_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_map_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_map_field_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_message.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_message_builder.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_message_builder_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_message_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_message_field_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_message_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_name_resolver.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_primitive_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_primitive_field_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_service.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_shared_code_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_string_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/java/java_string_field_lite.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_enum.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_enum_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_extension.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_file.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_helpers.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_map_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_message.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_message_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/javanano/javanano_primitive_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/js/embed.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/js/js_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/js/well_known_types_embed.cc

	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_enum.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_enum_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_extension.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_file.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_helpers.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_map_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_message.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_message_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_oneof.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/objectivec/objectivec_primitive_field.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/parser.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/php/php_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/plugin.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/plugin.pb.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/python/python_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/ruby/ruby_generator.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/subprocess.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/test_plugin.cc
	${PROTO_SRC_DIR}/google/protobuf/compiler/zip_writer.cc

)

ADD_LIBRARY(protoclib STATIC ${LIBPROTOC_SOURCES})
TARGET_LINK_LIBRARIES(protoclib protobuf)

ADD_EXECUTABLE(protoc ${PROTO_SRC_DIR}/google/protobuf/compiler/main.cc)
TARGET_LINK_LIBRARIES(protoc protoclib)

ADD_CUSTOM_COMMAND(TARGET protoc POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E touch ${build_stamp}
)

set(js_well_known_types_sources
  ${PROTO_SRC_DIR}/google/protobuf/compiler/js/well_known_types/any.js
  ${PROTO_SRC_DIR}/google/protobuf/compiler/js/well_known_types/struct.js
  ${PROTO_SRC_DIR}/google/protobuf/compiler/js/well_known_types/timestamp.js
)
add_executable(js_embed ${PROTO_SRC_DIR}/google/protobuf/compiler/js/embed.cc)
add_custom_command(
  OUTPUT ${PROTO_SRC_DIR}/google/protobuf/compiler/js/well_known_types_embed.cc
  DEPENDS js_embed ${js_well_known_types_sources}
  COMMAND js_embed ${js_well_known_types_sources} > ${PROTO_SRC_DIR}/google/protobuf/compiler/js/well_known_types_embed.cc
)

pb_export(protoc)
