#
# This file is open source software, licensed to you under the terms
# of the Apache License, Version 2.0 (the "License").  See the NOTICE file
# distributed with this work for additional information regarding copyright
# ownership.  You may not use this file except in compliance with the License.
#
# You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

#
# Copyright (C) 2018 Scylladb, Ltd.
#

# Logical target for all unit tests.
add_custom_target (unit_tests)

set (Seastar_UNIT_TEST_SMP
  2
  CACHE
  STRING
  "Run unit tests with this many cores.")

#
# Define a new unit test with the given name.
#
# seastar_add_test (name
#   [KIND {SEASTAR,BOOST,CUSTOM}]
#   [SOURCES source1 source2 ... sourcen]
#   [WORKING_DIRECTORY dir]
#   [LIBRARIES library1 library2 ... libraryn]
#   [RUN_ARGS arg1 arg2 ... argn])
#
# There are three kinds of test we support (the KIND parameter):
#
# - SEASTAR: Unit tests which use macros like `SEASTAR_TEST_CASE`
# - BOOST: Unit tests which use macros like `BOOST_AUTO_TEST_CASE`
# - CUSTOM: Custom tests which need to be specified
#
# SEASTAR and BOOST tests will have their output saved for interpretation by the Jenkins continuous integration service
# if this is configured for the build.
#
# KIND can be omitted, in which case it is assumed to be SEASTAR.
#
# If SOURCES is provided, then the test files are first compiled into an executable which has the same name as the test
# but with a suffix ("_test").
#
# WORKING_DIRECTORY can be optionally provided to choose where the test is executed.
#
# If LIBRARIES is provided along with SOURCES, then the executable is additionally linked with these libraries.
#
# RUN_ARGS are optional additional arguments to pass to the executable. For SEASTAR tests, these come after `--`. For
# CUSTOM tests with no SOURCES, this parameter can be used to specify the executable name as well as its arguments since
# no executable is compiled.
#
function (seastar_add_test name)
  set (test_kinds
    SEASTAR
    BOOST
    CUSTOM)

  cmake_parse_arguments (parsed_args
    ""
    "WORKING_DIRECTORY;KIND"
    "RUN_ARGS;SOURCES;LIBRARIES"
    ${ARGN})

  if (NOT parsed_args_KIND)
    set (parsed_args_KIND SEASTAR)
  elseif (NOT (parsed_args_KIND IN_LIST test_kinds))
    message (FATAL_ERROR "Invalid test kind. KIND must be one of ${test_kinds}")
  endif ()

  if (parsed_args_SOURCES)
    # These may be unused.
    seastar_jenkins_arguments (${name} jenkins_args)

    #
    # Each kind of test must populate the `args` and `libraries` lists.
    #

    set (libraries "${parsed_args_LIBRARIES}")

    if (parsed_args_KIND STREQUAL "SEASTAR")
      set (args "")

      list (APPEND libraries
        seastar_testing
        seastar_private)

      if (NOT (Seastar_JENKINS STREQUAL ""))
        list (APPEND args ${jenkins_args})
      endif ()

      list (APPEND args -- -c ${Seastar_UNIT_TEST_SMP})
      list (APPEND args ${parsed_args_RUN_ARGS})
    elseif (parsed_args_KIND STREQUAL "BOOST")
      set (args "")

      list (APPEND libraries
        Boost::unit_test_framework
        seastar_private)

      if (NOT (Seastar_JENKINS STREQUAL ""))
        list (APPEND args ${jenkins_args})
      endif ()

      list (APPEND args ${parsed_args_RUN_ARGS})
    else () # CUSTOM
      set (args ${parsed_args_RUN_ARGS})
    endif ()

    set (executable_target test_unit_${name})
    add_executable (${executable_target} ${parsed_args_SOURCES})

    target_link_libraries (${executable_target}
      PRIVATE ${libraries})

    target_compile_definitions (${executable_target}
      PRIVATE SEASTAR_TESTING_MAIN)

    target_include_directories (${executable_target}
      PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${Seastar_SOURCE_DIR}/src)

    set_target_properties (${executable_target}
      PROPERTIES
        OUTPUT_NAME ${name}_test)

    add_dependencies (unit_tests ${executable_target})
    set (forwarded_args COMMAND ${executable_target} ${args})
  else ()
    if (NOT (parsed_args_KIND STREQUAL "CUSTOM"))
      message (FATAL_ERROR "SOURCES are required for ${parsed_args_KIND} tests")
    endif ()

    set (forwarded_args COMMAND ${parsed_args_RUN_ARGS})
  endif ()

  #
  # We expect `forwarded_args` to be populated correctly at this point.
  #

  set (target test_unit_${name}_run)

  if (parsed_args_WORKING_DIRECTORY)
    list (APPEND forwarded_args WORKING_DIRECTORY ${parsed_args_WORKING_DIRECTORY})
  endif ()

  add_custom_target (${target}
    ${forwarded_args}
    USES_TERMINAL)

  add_test (
    NAME Seastar.unit.${name}
    COMMAND ${CMAKE_COMMAND} --build ${Seastar_BINARY_DIR} --target ${target})

  set_tests_properties (Seastar.unit.${name}
    PROPERTIES
      TIMEOUT "${Seastar_TEST_TIMEOUT}")
endfunction ()

#
# Define a new custom unit test whose entry point is a Seastar application.
#
# seastar_add_app_test (name
#   [SOURCES source1 source2 ... sourcen]
#   [LIBRARIES library1 library2 ... libraryn]
#   [RUN_ARGS arg1 arg2 ... argn])
#
# These kinds of tests are structured like Seastar applications.
#
# These tests always link against `seastar_private` and are always invoked with
# `-c ${Seastar_UNIT_TEST_SMP}`.
#
function (seastar_add_app_test name)
  cmake_parse_arguments (parsed_args
    ""
    ""
    "RUN_ARGS;SOURCES;LIBRARIES"
    ${ARGN})

  seastar_add_test (${name}
    KIND CUSTOM
    SOURCES ${parsed_args_SOURCES}
    LIBRARIES
      seastar_private
      ${parsed_args_LIBRARIES}
    RUN_ARGS
      -c ${Seastar_UNIT_TEST_SMP}
      ${parsed_args_RUN_ARGS})
endfunction ()

function (prepend_each var prefix)
  set (result "")

  foreach (x ${ARGN})
    list (APPEND result ${prefix}/${x})
  endforeach ()

  set (${var} ${result} PARENT_SCOPE)
endfunction ()

add_custom_target (test_unit
  COMMAND ctest --verbose -R Seastar.unit
  USES_TERMINAL)

seastar_add_test (abort_source
  SOURCES abort_source_test.cc)

seastar_add_test (alloc
  SOURCES alloc_test.cc)

if (NOT Seastar_EXECUTE_ONLY_FAST_TESTS)
  set (allocator_test_args "")
else ()
  if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set (allocator_test_args --iterations 5)
  else ()
    set (allocator_test_args --time 0.1)
  endif ()
endif ()

seastar_add_test (allocator
  SOURCES allocator_test.cc
  RUN_ARGS ${allocator_test_args})

seastar_add_app_test (alien
  SOURCES alien_test.cc)

seastar_add_test (checked_ptr
  SOURCES checked_ptr_test.cc)

seastar_add_test (chunked_fifo
  SOURCES chunked_fifo_test.cc)

seastar_add_test (circular_buffer
  SOURCES circular_buffer_test.cc)

seastar_add_test (circular_buffer_fixed_capacity
  SOURCES circular_buffer_fixed_capacity_test.cc)

seastar_add_test (connect
  SOURCES connect_test.cc)

if (Seastar_EXPERIMENTAL_COROUTINES_TS)
  seastar_add_test (coroutines
    SOURCES coroutines_test.cc)
endif ()

seastar_add_test (defer
  SOURCES defer_test.cc)

seastar_add_test (deleter
  SOURCES deleter_test.cc)

seastar_add_app_test (directory
  SOURCES directory_test.cc)

seastar_add_app_test (distributed
  SOURCES distributed_test.cc)

seastar_add_test (dns
  SOURCES dns_test.cc)

seastar_add_test (execution_stage
  SOURCES execution_stage_test.cc)

seastar_add_test (expiring_fifo
  SOURCES expiring_fifo_test.cc)

seastar_add_test (fair_queue
  SOURCES fair_queue_test.cc)

seastar_add_test (file_io
  SOURCES file_io_test.cc)

seastar_add_test (foreign_ptr
  SOURCES foreign_ptr_test.cc)

seastar_add_test (fstream
  SOURCES
    fstream_test.cc
    mock_file.hh)

seastar_add_test (futures
  SOURCES futures_test.cc)

seastar_add_test (sharded
  SOURCES sharded_test.cc)

seastar_add_test (httpd
  SOURCES
    httpd_test.cc
    loopback_socket.hh)

seastar_add_test (ipv6
  SOURCES ipv6_test.cc)

seastar_add_test (json_formatter
  SOURCES json_formatter_test.cc)

seastar_add_test (lowres_clock
  SOURCES lowres_clock_test.cc)

seastar_add_test (metrics
  SOURCES metrics_test.cc)

seastar_add_test (net_config
  KIND BOOST
  SOURCES net_config_test.cc)

seastar_add_test (noncopyable_function
  KIND BOOST
  SOURCES noncopyable_function_test.cc)

seastar_add_test (output_stream
  SOURCES output_stream_test.cc)

seastar_add_test (packet
  KIND BOOST
  SOURCES packet_test.cc)

seastar_add_test (program_options
  KIND BOOST
  SOURCES program_options_test.cc)

seastar_add_test (queue
  SOURCES queue_test.cc)

seastar_add_test (rpc
  SOURCES
    loopback_socket.hh
    rpc_test.cc)

seastar_add_test (semaphore
  SOURCES semaphore_test.cc)

seastar_add_test (shared_ptr
  KIND BOOST
  SOURCES shared_ptr_test.cc)

seastar_add_test (signal
  SOURCES signal_test.cc)

seastar_add_test (simple_stream
  KIND BOOST
  SOURCES simple_stream_test.cc)

# TODO: Disabled for now. See GH-520.
# seastar_add_test (slab
#   SOURCES slab_test.cc
#   NO_SEASTAR_TESTING_LIBRARY)

seastar_add_app_test (smp
  SOURCES smp_test.cc)

seastar_add_app_test (socket
  SOURCES socket_test.cc)

seastar_add_test (sstring
  KIND BOOST
  SOURCES sstring_test.cc)

seastar_add_test (stall_detector
  SOURCES stall_detector_test.cc)

seastar_add_test (thread
  SOURCES thread_test.cc)

seastar_add_app_test (thread_context_switch
  SOURCES thread_context_switch_test.cc)

seastar_add_app_test (timer
  SOURCES timer_test.cc)

seastar_add_test (uname
  KIND BOOST
  SOURCES uname_test.cc)

set (tls_certificate_files
  catest.key
  catest.pem
  tls-ca-bundle.pem
  test.crl
  test.crt
  test.csr
  test.key)

prepend_each (
  in_tls_certificate_files
  ${CMAKE_CURRENT_SOURCE_DIR}/
  ${tls_certificate_files})

prepend_each (
  out_tls_certificate_files
  ${CMAKE_CURRENT_BINARY_DIR}/
  ${tls_certificate_files})

add_custom_command (
  DEPENDS ${in_tls_certificate_files}
  OUTPUT ${out_tls_certificate_files}
  COMMAND ${CMAKE_COMMAND} -E copy ${in_tls_certificate_files} ${CMAKE_CURRENT_BINARY_DIR})

# TODO: Disabled for now. See GH-514.
# seastar_add_test (tls
#   DEPENDS ${out_tls_certificate_files}
#   SOURCES tls_test.cc
#   WORKING_DIRECTORY ${Seastar_BINARY_DIR})

seastar_add_test (tuple_utils
  KIND BOOST
  SOURCES tuple_utils_test.cc)

seastar_add_test (unwind
  KIND BOOST
  SOURCES unwind_test.cc)

seastar_add_test (weak_ptr
  KIND BOOST
  SOURCES weak_ptr_test.cc)
