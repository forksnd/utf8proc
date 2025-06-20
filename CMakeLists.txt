cmake_minimum_required (VERSION 3.10)

include (utils.cmake)

disallow_intree_builds()

# API version - be sure to update utf8proc.h and Makefile, too!
project (utf8proc VERSION 2.11.0 LANGUAGES C)

# This is the ABI version number, which may differ from the
# API version number (defined in utf8proc.h and above).
# Be sure to also update these in Makefile and MANIFEST!
set(SO_MAJOR 3)
set(SO_MINOR 2)
set(SO_PATCH 0)

option(UTF8PROC_INSTALL "Enable installation of utf8proc" On)
option(UTF8PROC_ENABLE_TESTING "Enable testing of utf8proc" Off)
option(LIB_FUZZING_ENGINE "Fuzzing engine to link against" Off)

add_library (utf8proc
  utf8proc.c
  utf8proc.h
)

# expose header path, for when this is part of a larger cmake project
target_include_directories(utf8proc PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}> $<INSTALL_INTERFACE:include>)

if (BUILD_SHARED_LIBS)
  # Building shared library
else()
  # Building static library
  target_compile_definitions(utf8proc PUBLIC "UTF8PROC_STATIC")
  if (MSVC)
    set_target_properties(utf8proc PROPERTIES OUTPUT_NAME "utf8proc_static")
  endif()
endif()

target_compile_definitions(utf8proc PRIVATE "UTF8PROC_EXPORTS")

if (NOT MSVC)
  set_target_properties(
    utf8proc PROPERTIES
    COMPILE_FLAGS "-O2 -std=c99 -pedantic -Wall"
  )
endif ()

set_target_properties (utf8proc PROPERTIES
  POSITION_INDEPENDENT_CODE ON
  VERSION "${SO_MAJOR}.${SO_MINOR}.${SO_PATCH}"
  SOVERSION ${SO_MAJOR}
)

if (UTF8PROC_INSTALL)
  include(GNUInstallDirs)
  install(FILES utf8proc.h DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  install(TARGETS utf8proc
    EXPORT utf8proc-targets
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
  )
  configure_file(libutf8proc.pc.cmakein libutf8proc.pc @ONLY)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/libutf8proc.pc" DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
  # Install CMake targets file.
  install(EXPORT utf8proc-targets FILE utf8proc-targets.cmake DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/utf8proc" NAMESPACE utf8proc::)
  include(CMakePackageConfigHelpers)
  configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/utf8proc-config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/utf8proc-config.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/utf8proc"
    NO_SET_AND_CHECK_MACRO
  )
  write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/utf8proc-config-version.cmake"
    COMPATIBILITY SameMajorVersion
  )
  install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/utf8proc-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/utf8proc-config-version.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/utf8proc"
  )
endif()

if(UTF8PROC_ENABLE_TESTING)
  enable_testing()
  file(MAKE_DIRECTORY data)
  set(UNICODE_VERSION 17.0.0)
  file(DOWNLOAD https://www.unicode.org/Public/${UNICODE_VERSION}/ucd/NormalizationTest.txt ${CMAKE_BINARY_DIR}/data/NormalizationTest.txt SHOW_PROGRESS)
  file(DOWNLOAD https://www.unicode.org/Public/${UNICODE_VERSION}/ucd/auxiliary/GraphemeBreakTest.txt ${CMAKE_BINARY_DIR}/data/GraphemeBreakTest.txt SHOW_PROGRESS)
  add_executable(case test/tests.h test/tests.c utf8proc.h test/case.c)
  target_link_libraries(case utf8proc)
  add_executable(custom test/tests.h test/tests.c utf8proc.h test/custom.c)
  target_link_libraries(custom utf8proc)
  add_executable(iterate test/tests.h test/tests.c utf8proc.h test/iterate.c)
  target_link_libraries(iterate utf8proc)
  add_executable(misc test/tests.h test/tests.c utf8proc.h test/misc.c)
  target_link_libraries(misc utf8proc)
  add_executable(printproperty test/tests.h test/tests.c utf8proc.h test/printproperty.c)
  target_link_libraries(printproperty utf8proc)
  add_executable(valid test/tests.h test/tests.c utf8proc.h test/valid.c)
  target_link_libraries(valid utf8proc)
  add_executable(maxdecomposition test/tests.h test/tests.c utf8proc.h test/maxdecomposition.c)
  target_link_libraries(maxdecomposition utf8proc)
  add_test(utf8proc.testcase case)
  add_test(utf8proc.testcustom custom)
  add_test(utf8proc.testiterate iterate)
  add_test(utf8proc.testmisc misc)
  add_test(utf8proc.testprintproperty printproperty)
  add_test(utf8proc.testvalid valid)
  add_test(utf8proc.testmaxdecomposition maxdecomposition)

  if (NOT WIN32)
    # no wcwidth function on Windows
    add_executable(charwidth test/tests.h test/tests.c utf8proc.h test/charwidth.c)
    target_link_libraries(charwidth utf8proc)
    add_test(utf8proc.testcharwidth charwidth)
  endif()
  add_executable(graphemetest test/tests.h test/tests.c utf8proc.h test/graphemetest.c)
  target_link_libraries(graphemetest utf8proc)
  add_executable(normtest test/tests.h test/tests.c utf8proc.h test/normtest.c)
  target_link_libraries(normtest utf8proc)
  add_test(utf8proc.testgraphemetest graphemetest data/GraphemeBreakTest.txt)
  add_test(utf8proc.testnormtest normtest data/NormalizationTest.txt)

  if(LIB_FUZZING_ENGINE)
    add_executable(fuzzer utf8proc.h test/fuzzer.c)
    target_link_libraries(fuzzer ${LIB_FUZZING_ENGINE} utf8proc)
  else()
    add_executable(fuzzer utf8proc.h test/fuzz_main.c test/fuzzer.c)
    target_link_libraries(fuzzer utf8proc)
  endif()
endif()
