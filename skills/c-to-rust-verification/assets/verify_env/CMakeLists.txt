cmake_minimum_required(VERSION 3.19)
project(harvest_verify_env C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 99)

include(FetchContent)

# GoogleTest pinned to a release tag. Declared before FuzzTest so the whole
# build reuses this one copy. Offline: pre-populate the source once and point
# CMake at it with -DFETCHCONTENT_SOURCE_DIR_GOOGLETEST=/path/to/googletest.
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.17.0.tar.gz
  DOWNLOAD_EXTRACT_TIMESTAMP ON
)
set(BUILD_GMOCK OFF CACHE BOOL "" FORCE)
set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
FetchContent_Declare(
  fuzztest
  GIT_REPOSITORY https://github.com/google/fuzztest.git
  GIT_TAG        2026-06-29
)
FetchContent_MakeAvailable(googletest fuzztest)

enable_testing()
include(GoogleTest)

# Enable fuzzing instrumentation AFTER the frameworks are configured, so
# gtest/fuzztest/abseil are not themselves instrumented — only what follows.
fuzztest_setup_fuzzing_flags()

# C reference under test, compiled INTO the test binary so it carries coverage
# instrumentation under fuzzing mode. The translated Rust is a black box loaded
# via dlopen at run time (see rust_lib.h) — never linked here, so the two sides'
# same-named exports do not collide.
#
# ADJUST the glob and include path below to the actual layout of the C source
# directory (some projects keep sources at the top level or in lib/, not src/).
file(GLOB C_UNDER_TEST_SOURCES CONFIGURE_DEPENDS
     "${CMAKE_CURRENT_SOURCE_DIR}/../c_src/src/*.c")
add_library(c_under_test STATIC ${C_UNDER_TEST_SOURCES})
target_include_directories(c_under_test PUBLIC
     "${CMAKE_CURRENT_SOURCE_DIR}/../c_src/include")
# Project-specific C build flags. FILL THIS IN from the C project's own build
# files (CMakeLists.txt target_compile_definitions / add_definitions, Makefile
# CFLAGS): namespace macros, behavior switches, etc. A misbuilt C reference
# makes every comparison a false mismatch — see the skill's oracle check.
# Example: target_compile_definitions(c_under_test PRIVATE XXH_NAMESPACE=LZ4_)
target_compile_definitions(c_under_test PRIVATE)
target_compile_options(c_under_test PRIVATE -w)

add_executable(verification_tests verification_tests.cc)
target_include_directories(verification_tests PRIVATE
     "${CMAKE_CURRENT_SOURCE_DIR}/../c_src/include")
target_link_libraries(verification_tests PRIVATE c_under_test ${CMAKE_DL_LIBS})
link_fuzztest(verification_tests)
gtest_discover_tests(verification_tests)
