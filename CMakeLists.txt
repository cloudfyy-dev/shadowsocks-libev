cmake_minimum_required(VERSION 3.28.3)

set(PROJECT_NAME shadowsocks-libev)
set(RELEASE_DATE 2024-05-03)
set(PROJECT_VERSION "3.3.6")
set(PROJECT_DESC "a lightweight secured socks5 proxy")
set(PROJECT_URL "https://shadowsocks.org")
set(PROJECT_ISSUES_URL "https://github.com/shadowsocks/shadowsocks-libev")
project(${PROJECT_NAME} VERSION ${PROJECT_VERSION})

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=gnu99")

#set(PROJECT_BINARY_DIR ${PROJECT_SOURCE_DIR}/out)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/cmake)
set(RUNTIME_SHARED_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/shared/bin)

set(CMAKE_MACOSX_RPATH TRUE)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif ()

# get linux distrubution infomation
function(get_linux_lsb_release_information)
    find_program(LSB_RELEASE_EXEC lsb_release)
    if(NOT LSB_RELEASE_EXEC)
        message(FATAL_ERROR "Could not detect lsb_release executable, can not gather required information")
    endif()
    execute_process(COMMAND "${LSB_RELEASE_EXEC}" --short --id OUTPUT_VARIABLE LSB_RELEASE_ID_SHORT OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND "${LSB_RELEASE_EXEC}" --short --release OUTPUT_VARIABLE LSB_RELEASE_VERSION_SHORT OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND "${LSB_RELEASE_EXEC}" --short --codename OUTPUT_VARIABLE LSB_RELEASE_CODENAME_SHORT OUTPUT_STRIP_TRAILING_WHITESPACE)

    set(LSB_RELEASE_ID_SHORT "${LSB_RELEASE_ID_SHORT}" PARENT_SCOPE)
    set(LSB_RELEASE_VERSION_SHORT "${LSB_RELEASE_VERSION_SHORT}" PARENT_SCOPE)
    set(LSB_RELEASE_CODENAME_SHORT "${LSB_RELEASE_CODENAME_SHORT}" PARENT_SCOPE)
endfunction()

# get libc-ares-dev package version
function(get_libc_ares_dev_pkg_version)
    find_program(DPKG_RELEASE_EXEC dpkg)
    if(NOT DPKG_RELEASE_EXEC)
        message(FATAL_ERROR "Could not detect dpkg executable...")
    endif()
    execute_process(COMMAND "${DPKG_RELEASE_EXEC}" -s libc-ares-dev OUTPUT_VARIABLE OUTPUT_DPKG_RELEASE_EXEC OUTPUT_STRIP_TRAILING_WHITESPACE)
   
    string(REGEX MATCH "1.27.0-1.0ubuntu1" HAS_1_27_0 "${OUTPUT_DPKG_RELEASE_EXEC}" )
   
    set(DPKG_RELEASE_EXEC_VERSION ${HAS_1_27_0} PARENT_SCOPE)
   
endfunction()

# Detect linux
if (UNIX AND NOT APPLE)
    set(LINUX TRUE)
    get_linux_lsb_release_information()
    message(STATUS "Linux ${LSB_RELEASE_ID_SHORT} ${LSB_RELEASE_VERSION_SHORT} ${LSB_RELEASE_CODENAME_SHORT}")
    
    if(LSB_RELEASE_ID_SHORT STREQUAL "Ubuntu")
        get_libc_ares_dev_pkg_version()
        message(STATUS "libc-ares-dev package version: ${DPKG_RELEASE_EXEC_VERSION}")
              
    else()
       
    endif()
endif ()

message(STATUS "Running cmake version ${CMAKE_VERSION}")

option(WITH_STATIC "build with static libraries." ON)
option(WITH_EMBEDDED_SRC "build with embedded libcork, libipset, and libbloom source." ON)

# Will set GIT_EXECUTABLE and GIT_FOUND
# find_package(Git)

# When choose to not use embedded libcork, libipset and libbloom, use libs shipped by system
if (NOT WITH_EMBEDDED_SRC)
    set(USE_SYSTEM_SHARED_LIB TRUE)
endif ()

# Run platform tests
include(${PROJECT_SOURCE_DIR}/cmake/configure.cmake)
configure_file(${PROJECT_SOURCE_DIR}/cmake/config.h.cmake ${PROJECT_BINARY_DIR}/src/config.h)
add_definitions(-I${PROJECT_BINARY_DIR}/src)
add_definitions(-DHAVE_CONFIG_H)

# pkg-config
configure_file(
        ${PROJECT_SOURCE_DIR}/cmake/shadowsocks-libev.pc.cmake
        ${PROJECT_BINARY_DIR}/pkgconfig/shadowsocks-libev.pc
        @ONLY
)
install(FILES
        ${PROJECT_BINARY_DIR}/pkgconfig/shadowsocks-libev.pc
        DESTINATION lib/pkgconfig
        )

if (WITH_EMBEDDED_SRC)
# We need libcork,libipset headers
include_directories(libcork/include)
include_directories(libipset/include)
include_directories(libbloom/murmur2)
include_directories(libbloom)

set(LIBCORK_SOURCE
        libcork/src/libcork/cli/commands.c
        libcork/src/libcork/core/allocator.c
        libcork/src/libcork/core/error.c
        libcork/src/libcork/core/gc.c
        libcork/src/libcork/core/hash.c
        libcork/src/libcork/core/ip-address.c
        libcork/src/libcork/core/mempool.c
        libcork/src/libcork/core/timestamp.c
        libcork/src/libcork/core/u128.c
        libcork/src/libcork/core/version.c
        libcork/src/libcork/ds/array.c
        libcork/src/libcork/ds/bitset.c
        libcork/src/libcork/ds/buffer.c
        libcork/src/libcork/ds/dllist.c
        libcork/src/libcork/ds/file-stream.c
        libcork/src/libcork/ds/hash-table.c
        libcork/src/libcork/ds/managed-buffer.c
        libcork/src/libcork/ds/ring-buffer.c
        libcork/src/libcork/ds/slice.c
        libcork/src/libcork/posix/directory-walker.c
        libcork/src/libcork/posix/env.c
        libcork/src/libcork/posix/exec.c
        libcork/src/libcork/posix/files.c
        libcork/src/libcork/posix/process.c
        libcork/src/libcork/pthreads/thread.c
        )
if (NOT MINGW)
set(LIBCORK_SOURCE ${LIBCORK_SOURCE} libcork/src/libcork/posix/subprocess.c)
else ()
set(LIBCORK_SOURCE ${LIBCORK_SOURCE} libcork/src/libcork/posix/mingw.c)
endif ()

if (WITH_STATIC)
add_library(cork STATIC ${LIBCORK_SOURCE})
target_compile_definitions(cork PUBLIC -DCORK_API=CORK_LOCAL)
endif ()

add_library(cork-shared SHARED ${LIBCORK_SOURCE})
target_compile_definitions(cork-shared PUBLIC -DCORK_API=CORK_EXPORT)
set_target_properties(cork-shared PROPERTIES OUTPUT_NAME cork)
if (MINGW)
target_link_libraries(cork-shared ws2_32)
endif ()

set(LIBIPSET_SOURCE
        libipset/src/libipset/general.c
        libipset/src/libipset/bdd/assignments.c
        libipset/src/libipset/bdd/basics.c
        libipset/src/libipset/bdd/bdd-iterator.c
        libipset/src/libipset/bdd/expanded.c
        libipset/src/libipset/bdd/reachable.c
        libipset/src/libipset/bdd/read.c
        libipset/src/libipset/bdd/write.c
        libipset/src/libipset/map/allocation.c
        libipset/src/libipset/map/inspection.c
        libipset/src/libipset/map/ipv4_map.c
        libipset/src/libipset/map/ipv6_map.c
        libipset/src/libipset/map/storage.c
        libipset/src/libipset/set/allocation.c
        libipset/src/libipset/set/inspection.c
        libipset/src/libipset/set/ipv4_set.c
        libipset/src/libipset/set/ipv6_set.c
        libipset/src/libipset/set/iterator.c
        libipset/src/libipset/set/storage.c
        )

if (WITH_STATIC)
add_library(ipset STATIC ${LIBIPSET_SOURCE})
endif ()

add_library(ipset-shared SHARED ${LIBIPSET_SOURCE})
set_target_properties(ipset-shared PROPERTIES OUTPUT_NAME ipset)

set(LIBBLOOM_SOURCE
        libbloom/bloom.c
        libbloom/murmur2/MurmurHash2.c
        )

if (WITH_STATIC)
add_library(bloom STATIC ${LIBBLOOM_SOURCE})
target_link_libraries(ipset cork bloom)
endif ()

add_library(bloom-shared SHARED ${LIBBLOOM_SOURCE})
target_link_libraries(ipset-shared cork-shared bloom-shared)
set_target_properties(bloom-shared PROPERTIES OUTPUT_NAME bloom)
endif ()

add_subdirectory(src)
add_subdirectory(doc)

