cmake_minimum_required(VERSION 2.8)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMake;${CMAKE_MODULE_PATH}")
include(Utilities)
project(KinesisVideoWebRTCClient LANGUAGES C)

# User Flags
option(ADD_MUCLIBC "Add -muclibc c flag" OFF)
option(BUILD_DEPENDENCIES "Whether or not to build depending libraries from source" ON)
option(USE_OPENSSL "Use openssl as crypto library" ON)
option(USE_MBEDTLS "Use mbedtls as crypto library" OFF)
option(BUILD_STATIC_LIBS "Build all libraries statically. (This includes third-party libraries.)" OFF)
option(BUILD_OPENSSL_PLATFORM "If buildng OpenSSL what is the target platform" OFF)
option(BUILD_LIBSRTP_HOST_PLATFORM "If buildng LibSRTP what is the current platform" OFF)
option(BUILD_LIBSRTP_DESTINATION_PLATFORM "If buildng LibSRTP what is the destination platform" OFF)

# Developer Flags
option(BUILD_TEST "Build the testing tree." OFF)
option(CODE_COVERAGE "Enable coverage reporting" OFF)
option(COMPILER_WARNINGS "Enable all compiler warnings." OFF)
option(ADDRESS_SANITIZER "Build with AddressSanitizer." OFF)
option(MEMORY_SANITIZER "Build with MemorySanitizer." OFF)
option(THREAD_SANITIZER "Build with ThreadSanitizer." OFF)
option(UNDEFINED_BEHAVIOR_SANITIZER "Build with UndefinedBehaviorSanitizer." OFF)

set(CMAKE_MACOSX_RPATH TRUE)
get_filename_component(ROOT "${CMAKE_CURRENT_SOURCE_DIR}" ABSOLUTE)

# static settings
if(BUILD_STATIC_LIBS OR WIN32)
  set(LINKAGE STATIC)
else()
  set(LINKAGE SHARED)
endif()

set(KINESIS_VIDEO_WEBRTC_CLIENT_SRC "${CMAKE_CURRENT_SOURCE_DIR}")


message(STATUS "Kinesis Video WebRTC Client path is ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}")
message(STATUS "dependencies install path is ${OPEN_SRC_INSTALL_PREFIX}")

# pass ca cert location to sdk
add_definitions(-DKVS_CA_CERT_PATH="${CMAKE_SOURCE_DIR}/certs/cert.pem")
add_definitions(-DCMAKE_DETECTED_CACERT_PATH)

if(USE_OPENSSL)
  add_definitions(-DKVS_USE_OPENSSL)
elseif(USE_MBEDTLS)
  add_definitions(-DKVS_USE_MBEDTLS)
endif()

if(BUILD_DEPENDENCIES)
  set(OPEN_SRC_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/open-source)
  if(NOT EXISTS ${OPEN_SRC_INSTALL_PREFIX})
    file(MAKE_DIRECTORY ${OPEN_SRC_INSTALL_PREFIX})
  endif()

  set(ENV{PKG_CONFIG_PATH} "$ENV{PKG_CONFIG_PATH}:${OPEN_SRC_INSTALL_PREFIX}/lib/pkgconfig")
  set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${OPEN_SRC_INSTALL_PREFIX})

  message(STATUS "Begin building dependencies.")

  set(SSL_OPTIONS USE_OPENSSL USE_MBEDTLS)
  count_true(ENABLED_SSL_OPTION_COUNT ${SSL_OPTIONS})

  if(ENABLED_SSL_OPTION_COUNT GREATER "1")
    message(FATAL_ERROR "Only one of ${SSL_OPTIONS} can be enabled")
  endif()

  if (USE_OPENSSL)
    set(BUILD_ARGS -DBUILD_STATIC=${BUILD_STATIC}
                   -DBUILD_OPENSSL_PLATFORM=${BUILD_OPENSSL_PLATFORM}
                   -DOPENSSL_EXTRA=${OPENSSL_EXTRA})
    build_dependency(openssl ${BUILD_ARGS})
    set(OPENSSL_ROOT_DIR ${OPEN_SRC_INSTALL_PREFIX})
  elseif(USE_MBEDTLS)
    set(BUILD_ARGS -DBUILD_STATIC=${BUILD_STATIC})
    build_dependency(mbedtls ${BUILD_ARGS})
  endif()


  set(BUILD_ARGS -DBUILD_STATIC_LIBS=${BUILD_STATIC_LIBS}
                 -DOPENSSL_DIR=${OPEN_SRC_INSTALL_PREFIX}
                 -DUSE_OPENSSL=${USE_OPENSSL}
                 -DUSE_MBEDTLS=${USE_MBEDTLS})
  build_dependency(websockets ${BUILD_ARGS})

  set(BUILD_ARGS
      -DBUILD_STATIC_LIBS=${BUILD_STATIC_LIBS}
      -DOPENSSL_DIR=${OPEN_SRC_INSTALL_PREFIX}
      -DBUILD_LIBSRTP_HOST_PLATFORM=${BUILD_LIBSRTP_HOST_PLATFORM}
      -DBUILD_LIBSRTP_DESTINATION_PLATFORM=${BUILD_LIBSRTP_DESTINATION_PLATFORM}
      -DUSE_OPENSSL=${USE_OPENSSL}
      -DUSE_MBEDTLS=${USE_MBEDTLS}
  )
  build_dependency(srtp ${BUILD_ARGS})

  build_dependency(usrsctp)
  if(BUILD_TEST)
   build_dependency(gtest)
  endif()

  # building kvsCommonLws also builds kvspic
  set(BUILD_ARGS
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DUSE_OPENSSL=${USE_OPENSSL}
      -DUSE_MBEDTLS=${USE_MBEDTLS})
  build_dependency(kvsCommonLws ${BUILD_ARGS})

  message(STATUS "Finished building dependencies.")
endif()

############# find dependent libraries ############

find_package(Threads)
find_package(PkgConfig REQUIRED)
if (USE_OPENSSL)
  find_package(OpenSSL REQUIRED)
  set(OPEN_SRC_INCLUDE_DIRS ${OPEN_SRC_INCLUDE_DIRS} ${OPENSSL_INCLUDE_DIR})
else()
  find_package(MbedTLS REQUIRED)
  set(OPEN_SRC_INCLUDE_DIRS ${OPEN_SRC_INCLUDE_DIRS} ${MBEDTLS_INCLUDE_DIRS})
endif()

if (OPEN_SRC_INSTALL_PREFIX)
  find_library(SRTP_LIBRARIES srtp2 REQUIRED PATHS ${OPEN_SRC_INSTALL_PREFIX})
else()
  find_library(SRTP_LIBRARIES srtp2 REQUIRED )
endif()

if (WIN32)
  SET(LIBWEBSOCKETS_LIBRARIES "websockets.lib")
else()
  pkg_check_modules(LIBWEBSOCKETS REQUIRED libwebsockets)
endif()

# usrsctp dont support pkgconfig yet
find_library(
  Usrsctp
  NAMES ${USRSCTP_LIBNAME} usrsctp REQUIRED
  PATHS ${OPEN_SRC_INSTALL_PREFIX}/lib)

set(OPEN_SRC_INCLUDE_DIRS ${OPEN_SRC_INCLUDE_DIRS} ${LIBSRTP_INCLUDE_DIRS} 
                          ${CURL_INCLUDE_DIRS} ${LIBWEBSOCKETS_INCLUDE_DIRS})

link_directories(${LIBSRTP_LIBRARY_DIRS})
link_directories(${LIBWEBSOCKETS_LIBRARY_DIRS})
link_directories(${OPEN_SRC_INSTALL_PREFIX}/lib)

pkg_check_modules(GST gstreamer-1.0)
if(GST_FOUND)

  if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(ENV{PKG_CONFIG_PATH} "$ENV{PKG_CONFIG_PATH}:/usr/local/opt/libffi/lib/pkgconfig")
    find_library(Intl "intl" REQUIRED PATHS "/usr/local/opt/gettext/lib")
    link_directories("/usr/local/opt/gettext/lib")
  endif()

  pkg_check_modules(GLIB2 REQUIRED glib-2.0)
  pkg_check_modules(GST_APP REQUIRED gstreamer-app-1.0)
  pkg_check_modules(GOBJ2 REQUIRED gobject-2.0)
  message("gstreamer found. Will build gstreamer samples")

  set(OPEN_SRC_INCLUDE_DIRS
      ${OPEN_SRC_INCLUDE_DIRS} ${GLIB2_INCLUDE_DIRS} ${GST_INCLUDE_DIRS}
      ${GST_APP_INCLUDE_DIRS} ${GOBJ2_INCLUDE_DIRS})

  set(GST_SAMPLE_LIBRARIES ${GLIB2_LIBRARIES} ${GST_LIBRARIES}
                           ${GST_APP_LIBRARIES} ${GOBJ2_LIBRARIES} ${Intl})

  link_directories(${GLIB2_LIBRARY_DIRS})
  link_directories(${GST_LIBRARY_DIRS})
  link_directories(${GST_APP_LIBRARY_DIRS})
  link_directories(${GOBJ2_LIBRARY_DIRS})
else()
  message("gstreamer not found. Will not build gstreamer samples")
endif()

############# find dependent libraries end ############

if("${CMAKE_C_COMPILER_ID}" MATCHES "GNU|Clang")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")

  if(ADD_MUCLIBC)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -muclibc")
  endif()

  if(CODE_COVERAGE)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g -fprofile-arcs -ftest-coverage")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
  endif()

  if(ADDRESS_SANITIZER)
    enableSanitizer("address")
  endif()
  if(MEMORY_SANITIZER)
    enableSanitizer("memory")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-memory-track-origins")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize-memory-track-origins")
  endif()
  if(THREAD_SANITIZER)
    enableSanitizer("thread")
  endif()
  if(UNDEFINED_BEHAVIOR_SANITIZER)
    enableSanitizer("undefined")
  endif()
endif()

# Uncomment below line for debug heap
# add_definitions(-DHEAP_DEBUG)

# Uncomment below to add thread id in log
# add_definitions(-DENABLE_LOG_THREAD_ID)

# Uncomment below line for very verbose logging
# add_definitions(-DLOG_STREAMING)

file(
  GLOB
  WEBRTC_CLIENT_SOURCE_FILES
  "src/source/Crypto/*.c"
  "src/source/Ice/*.c"
  "src/source/PeerConnection/*.c"
  "src/source/Rtcp/*.c"
  "src/source/Rtp/*.c"
  "src/source/Rtp/Codecs/*.c"
  "src/source/Sdp/*.c"
  "src/source/Srtp/*.c"
  "src/source/Stun/*.c"
  "src/source/Sctp/*.c"
  "src/source/Metrics/*.c")

if (USE_OPENSSL)
  list(FILTER WEBRTC_CLIENT_SOURCE_FILES EXCLUDE REGEX ".*_mbedtls\\.c")
elseif (USE_MBEDTLS)
  list(FILTER WEBRTC_CLIENT_SOURCE_FILES EXCLUDE REGEX ".*_openssl\\.c")
endif()

file(GLOB WEBRTC_SIGNALING_CLIENT_SOURCE_FILES "src/source/Signaling/*.c")

file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/samples/opusSampleFrames" DESTINATION .)
file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/samples/h264SampleFrames" DESTINATION .)

include_directories(${OPEN_SRC_INCLUDE_DIRS})
include_directories(${OPEN_SRC_INSTALL_PREFIX}/include)
include_directories(${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/src/include)
include_directories(${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/src/ice)

add_library(kvsWebrtcClient ${LINKAGE} ${WEBRTC_CLIENT_SOURCE_FILES})

target_link_libraries(
  kvsWebrtcClient
  PRIVATE kvspicUtils
          kvspicState
          ${CMAKE_THREAD_LIBS_INIT}
          ${OPENSSL_SSL_LIBRARY}
          ${OPENSSL_CRYPTO_LIBRARY}
          ${SRTP_LIBRARIES}
          ${Usrsctp}
          ${MBEDTLS_LIBRARIES}
          ${EXTRA_DEPS})

add_library(kvsWebrtcSignalingClient ${LINKAGE} ${WEBRTC_SIGNALING_CLIENT_SOURCE_FILES})

target_link_libraries(
  kvsWebrtcSignalingClient
  PRIVATE kvspicUtils
          kvspicState
          ${CMAKE_THREAD_LIBS_INIT}
          ${LIBWEBSOCKETS_LIBRARIES}
          ${EXTRA_DEPS}
          ${OPENSSL_SSL_LIBRARY}
          ${OPENSSL_CRYPTO_LIBRARY}
          ${MBEDTLS_LIBRARIES}
  PUBLIC kvsCommonLws)

if (WIN32)
  target_link_libraries(kvsWebrtcClient PRIVATE "Ws2_32" "iphlpapi")
endif()

add_executable(
  kvsWebrtcClientMaster
  ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/samples/Common.c
  ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/samples/kvsWebRTCClientMaster.c)
target_link_libraries(kvsWebrtcClientMaster kvsWebrtcClient kvsWebrtcSignalingClient kvspicUtils)

add_executable(
  kvsWebrtcClientViewer
  ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/samples/Common.c
  ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/samples/kvsWebRTCClientViewer.c)
target_link_libraries(kvsWebrtcClientViewer kvsWebrtcClient kvsWebrtcSignalingClient kvspicUtils)

add_executable(
        discoverNatBehavior
        ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/samples/discoverNatBehavior.c)
target_link_libraries(discoverNatBehavior kvsWebrtcClient)

if(COMPILER_WARNINGS)
  target_compile_options(kvsWebrtcClient PUBLIC -Wall -Werror -pedantic -Wextra -Wno-unknown-warning-option)
  target_compile_options(kvsWebrtcSignalingClient PUBLIC -Wall -Werror -pedantic -Wextra -Wno-unknown-warning-option)
endif()

if(GST_FOUND)
  add_executable(
    kvsWebrtcClientMasterGstSample
    ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/samples/Common.c
    ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/samples/kvsWebRTCClientMasterGstreamerSample.c
  )
  target_link_libraries(kvsWebrtcClientMasterGstSample kvsWebrtcClient kvsWebrtcSignalingClient ${GST_SAMPLE_LIBRARIES} kvspicUtils)

  install(TARGETS kvsWebrtcClientMasterGstSample
    RUNTIME DESTINATION bin
  )
endif()

if(BUILD_TEST)
  add_subdirectory(tst)
endif()

install(TARGETS kvsWebrtcClient kvsWebrtcSignalingClient kvsWebrtcClientMaster kvsWebrtcClientViewer discoverNatBehavior
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  )

install(DIRECTORY ${KINESIS_VIDEO_WEBRTC_CLIENT_SRC}/src/include/
  DESTINATION include
  )
