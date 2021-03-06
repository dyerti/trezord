#
# CMake build file for trezord
#

cmake_minimum_required(VERSION 2.8)

project(trezord)

file (STRINGS "VERSION" VERSION)

option(BUILD_TESTS "Build tests?" off)

include_directories(src)

set (SRCS
  src/main.cpp
  src/http_api.hpp
  src/http_server.hpp
  src/http_client.hpp
  src/core.hpp
  src/wire.hpp
  src/utils.hpp
  src/protobuf/state.hpp
  src/protobuf/json_codec.hpp
  src/protobuf/wire_codec.hpp)

if (WIN32)
  set (SRCS src/trezord.rc ${SRCS})
endif(WIN32)

if (UNIX AND NOT APPLE)
  set (SRCS src/glibc_compat.c ${SRCS})
endif(UNIX AND NOT APPLE)

# use c++11, add version macro
set(CMAKE_CXX_FLAGS "-std=c++11 ${CMAKE_CXX_FLAGS} -DVERSION='\"${VERSION}\"'")

# use vendored cmake modules
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/modules")

# add dynamic libs
find_package(OpenSSL REQUIRED)
find_package(libmicrohttpd REQUIRED)
find_package(GnuTLS REQUIRED)
find_package(GCrypt REQUIRED)
find_package(CURL REQUIRED)

# add static libs
set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
set(BUILD_SHARED_LIBS off)
set(Boost_USE_STATIC_LIBS on)
find_package(Boost 1.53.0 REQUIRED regex thread system unit_test_framework program_options)
find_package(Protobuf 2.5.0 REQUIRED)
find_package(jsoncpp REQUIRED)

# generate protobuf source and header files
# resulting source and headers are generated in binary dir
# include binary dir to allow includes of headers
include_directories(${CMAKE_CURRENT_BINARY_DIR})
# generate files and add them to the build
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS vendor/trezor-common/protob/config.proto)
set(SRCS ${SRCS} ${PROTO_SRCS} ${PROTO_HDRS})

add_executable(trezord ${SRCS})

# add vendored libs
add_subdirectory(vendor/hidapi)
add_subdirectory(vendor/trezor-crypto)

include_directories(
  ${Boost_INCLUDE_DIRS}
  ${LIBMICROHTTPD_INCLUDE_DIRS}
  ${PROTOBUF_INCLUDE_DIRS}
  ${JSONCPP_INCLUDE_DIRS}
  ${CURL_INCLUDE_DIRS}
  vendor/hidapi/hidapi
  vendor/trezor-crypto
  vendor/easyloggingpp)

if (WIN32)
  set(OS_LIBRARIES wsock32 ws2_32 z)
  add_definitions(-D_WIN32_WINNT=0x6000)
else(WIN32)
  if (APPLE)
    set(OS_LIBRARIES pthread)
  else(APPLE)
    set(OS_LIBRARIES pthread)
  endif(APPLE)
endif(WIN32)

target_link_libraries(trezord
  ${Boost_LIBRARIES}
  ${OPENSSL_LIBRARIES}
  ${GNUTLS_LIBRARIES}
  ${GCRYPT_LIBRARIES}
  ${LIBMICROHTTPD_LIBRARIES}
  ${PROTOBUF_LIBRARIES}
  ${JSONCPP_LIBRARIES}
  ${CURL_LIBRARIES}
  ${OS_LIBRARIES}
  hidapi
  TrezorCrypto)

# set_target_properties(trezord PROPERTIES LINK_SEARCH_START_STATIC 1)
# set_target_properties(trezord PROPERTIES LINK_SEARCH_END_STATIC 1)

if(BUILD_TESTS)

  include_directories(test)

  add_executable(test-protobuf_codecs test/protobuf_codecs.cpp)

  target_link_libraries(test-protobuf_codecs
    ${Boost_LIBRARIES}
    ${PROTOBUF_LIBRARIES}
    ${JSONCPP_LIBRARIES})

  enable_testing()
  add_test(ProtobufCodecs test-protobuf_codecs)

endif(BUILD_TESTS)
