# You may redistribute this program and/or modify it under the terms of
# the GNU General Public License as published by the Free Software Foundation,
# either version 3 of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
project(cjdns C)
cmake_minimum_required(VERSION 2.8)
message(${CMAKE_VERSION})

if(CMAKE_BINARY_DIR STREQUAL ${CMAKE_SOURCE_DIR} AND NOT OPENWRT)
    message( FATAL_ERROR "type: git clean -df && mkdir build && cd build && cmake .. && make" )
endif()

if (OPENWRT)
    # OpenWRT messes around with the code quite a bit and makes the codestyle validator angry.
    set(NO_CODESTYLE TRUE)

    # OpenWRT's NACL is not built position independent.
    set(WITH_PIE FALSE)

    # Use the bundled NaCl because it's faster.
    #set(HAVE_NACL TRUE)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}
    ${CMAKE_SOURCE_DIR}/cmake/modules)

# validation
add_definitions(-Wall -Wextra -std=c99 -pedantic)
if ("$ENV{NO_WERROR}" STREQUAL "")
    add_definitions(-Werror)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL SunOS)
    set(ILLUMOS TRUE)
    set(SYSTEM Illumos)
elseif(CMAKE_SYSTEM_NAME STREQUAL Linux)
    set(LINUX TRUE)
    set(SYSTEM Linux)
elseif(CMAKE_SYSTEM_NAME STREQUAL Darwin)
    set(APPLE TRUE)
    set(SYSTEM Darwin)
elseif(CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
    set(FREEBSD TRUE)
    set(BSD TRUE)
    add_definitions(-D BSD=1)
    set(SYSTEM FreeBSD)
elseif(CMAKE_SYSTEM_NAME STREQUAL OpenBSD)
    set(OPENBSD TRUE)
    set(BSD TRUE)
    add_definitions(-D BSD=1)
    set(SYSTEM OpenBSD)
elseif(CMAKE_SYSTEM_NAME STREQUAL Windows)
    set(Windows TRUE)
    set(SYSTEM Windows)
endif()
add_definitions(-D ${SYSTEM}=1)

if(Windows)
    # ISO C does not support the ‘I64’ ms_printf length modifier
    # but PRId64 is #defined as "I64d"
    add_definitions(-Wno-format)

    # SRWLOCK is vista or newer and needed by libuv
    add_definitions(-D_WIN32_WINNT=0x0600)

    # BufferAllocator.c:1:0: error: -fPIC ignored for target (all code is position independent)
    set(WITH_PIE OFF CACHE INTERNAL "" FORCE)
endif()
if(CMAKE_CROSSCOMPILING)
    if (NOT "$ENV{REMOTE_TEST_IP_PORT}" STREQUAL "")
        set(REMOTE_TEST_IP_PORT "$ENV{REMOTE_TEST_IP_PORT}")
    endif()
endif()

if(EXISTS ${CMAKE_SOURCE_DIR}/interface/ETHInterface_${SYSTEM}.c)
  add_definitions(-D HAS_ETH_INTERFACE=1)
  set(HAS_ETH_INTERFACE TRUE)
endif()

if(CMAKE_C_COMPILER MATCHES "(.*)-gcc")
    string(REGEX REPLACE "-gcc$" "" toolchain ${CMAKE_C_COMPILER})
    string(REGEX REPLACE ".*/" "" toolchain ${toolchain})
    message("detected toolchain ${toolchain}")
    set(TOOLCHAIN ${toolchain} CACHE STRING "Toolchain for compiling (prefix for -gcc, ... binaries)")
else()
    set(TOOLCHAIN "" CACHE STRING "Toolchain for compiling (prefix for -gcc, ... binaries)")
endif()

if(TOOLCHAIN AND NOT CMAKE_C_COMPILER)
    set(CMAKE_C_COMPILER ${TOOLCHAIN}-gcc)
endif()

if(TOOLCHAIN AND NOT CMAKE_CXX_COMPILER)
    set(CMAKE_C_COMPILER ${TOOLCHAIN}-g++)
endif()

if(Windows AND TOOLCHAIN AND NOT CMAKE_CXX_COMPILER)
    set(CMAKE_C_COMPILER ${TOOLCHAIN}-windres)
endif()

message("c compiler: ${CMAKE_C_COMPILER}")
message("c++ compiler: ${CMAKE_CXX_COMPILER}")
if(Windows)
    message("rc compiler: ${CMAKE_RC_COMPILER}")
endif()

# This breaks logging since loggers are passed to functions
# and only used if the log level is high.
add_definitions(-Wno-unused-parameter)

include(CheckCCompilerFlag)

# There are a number of places in Admin where read() and write() are used
# and they fail silently if anything goes wrong so the result is unused.
check_c_compiler_flag(-Wno-unused-result HAS_NO_UNUSED_RESULT)
if(HAS_NO_UNUSED_RESULT)
    add_definitions(-Wno-unused-result)
endif()

# GCC only sends a warning and not an error if it can't compile with stack canaries.
set(CMAKE_REQUIRED_FLAGS "-Werror")
check_c_compiler_flag(-fstack-protector-all HAS_F_STACK_PROTECTOR)
if(NOT HAS_F_STACK_PROTECTOR)
    message("WARNING: Stack Smashing Protector is not available on this target.")
    message("SSP is an important security measure to minimize the risk of a vulnerability.")
    message("")
else()
    add_definitions(
        # Broken GCC patch makes -fstack-protector-all not work
        # workaround is to give -fno-stack-protector first.
        # see: https://bugs.launchpad.net/ubuntu/+source/gcc-4.5/+bug/691722
        -fno-stack-protector
        -fstack-protector-all
        -Wstack-protector
    )
endif()

OPTION(NO_CODESYTLE "No codestyle checks")

# check codestyle
if(NOT NO_CODESTYLE)
    set(checker ${CMAKE_SOURCE_DIR}/cmake/modules/CheckCodestyle.cmake)
    add_custom_command(OUTPUT check.codestyle
        COMMAND ${CMAKE_COMMAND}
        -DCSD=${CMAKE_SOURCE_DIR} -DSKIP=CMakeFiles -P ${checker}
    )
    add_custom_target(check_output ALL DEPENDS check.codestyle)
endif()

check_c_compiler_flag(-fcatch-undefined-behavior HAS_CATCH_UNDEFINED)
if(HAS_CATCH_UNDEFINED)
    add_definitions(-fcatch-undefined-behavior)
endif()

# __builtin_constant_p()
include(CheckCSourceCompiles)
check_c_source_compiles("int main() { return __builtin_constant_p(0);}" HAS_BUILTIN_CONSTANT_P)
if (HAS_BUILTIN_CONSTANT_P)
    add_definitions("-D HAS_BUILTIN_CONSTANT_P")
endif()

OPTION(WITH_PIE "Build a position independent executable" ON)
if (NOT "$ENV{NO_PIE}" STREQUAL "")
    set(WITH_PIE FALSE)
endif()
if(WITH_PIE)
    message("Building a position independent executable (ASLR security enabled).")
    message("NOTE: PIE executables cannot run in a debugger.")
    message("")
    add_definitions(-fPIE)
    set(PIE "-pie")
else()
    message("Building a position dependent executable (ASLR security disabled).")
    set(PIE "")
endif()

if ("$ENV{NO_DEBUG}" STREQUAL "")
    add_definitions(-g)
endif()

set(LDFLAGS ${CMAKE_EXE_LINKER_FLAGS})
if(Windows)
elseif(ILLUMOS)
    # TODO make this less ugly
    set(LDFLAGS "${LDFLAGS} ${PIE} -lsocket")
elseif(APPLE)
    # apple seems to set noexecstack by default and not support relro.
    set(LDFLAGS "${LDFLAGS} ${PIE}")
else()
    set(LDFLAGS "${LDFLAGS} ${PIE} -Wl,-z,relro,-z,now,-z,noexecstack")
endif()

# logging
SET(Log_LEVEL DEBUG CACHE STRING "Log level (KEYS, DEBUG, INFO, WARN, ERROR, CRITICAL)")
if (NOT "$ENV{Log_LEVEL}" STREQUAL "")
    set(Log_LEVEL "$ENV{Log_LEVEL}")
endif()
if (${Log_LEVEL} STREQUAL "KEYS")
    message("\n\nEXPECT TO SEE PRIVATE KEYS PRINTED IN YOUR LOGS!\n\n")
endif (${Log_LEVEL} STREQUAL "KEYS")
add_definitions("-D Log_${Log_LEVEL}")

# vrooooooom
#add_definitions(-O2)

if (NOT $ENV{GCOV} STREQUAL "")
    add_definitions(-fprofile-arcs -ftest-coverage)
    set(LDFLAGS "${LDFLAGS} -fprofile-arcs -ftest-coverage")
endif()

#IF(NOT CMAKE_BUILD_TYPE)
# default to RelWithDebInfo (-g -O2)
#    SET(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING
#        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
#        FORCE)
#ENDIF(NOT CMAKE_BUILD_TYPE)

OPTION(WITH_LTO "Building with link time optimization.")
if(WITH_LTO)
    add_definitions(-flto)
    set(LDFLAGS "${LDFLAGS} -flto -fuse-linker-plugin -fwhole-program")
endif()


# Max direct connections to the switch.
SET(CJDNS_MAX_PEERS 256 CACHE STRING "maximum number of peers")
add_definitions("-D CJDNS_MAX_PEERS=${CJDNS_MAX_PEERS}")

# make sure casts are done properly.
add_definitions("-D Identity_CHECK=1")

# add in a billion checks which slow things down but catch bugs.
add_definitions("-D PARANOIA=1")

include_directories(${CMAKE_SOURCE_DIR})

#if NEON is specified then we will force NACL to use NEON.
if (NOT $ENV{NEON} STREQUAL "")
    set(NEON TRUE)
endif ()
# Look for ARM NEON if it's available
find_package(NEON)
if (NEON)
    # needed because the NEON asm files do not support
    # THUMB and linking to them causes bad things.
    add_definitions(-marm)
endif()


if(HAVE_LIBUV)
    find_package(Libuv REQUIRED)
else()
    find_package(LibuvBundled REQUIRED)
endif()

if(HAVE_NACL)
    find_package(NACL REQUIRED)
elseif(HAVE_SODIUM)
    find_package(Sodium REQUIRED)
else()
    if (NOT $ENV{NO_EXTERNAL_NACL})
        find_package(NACL)
    endif ()
    find_package(NACLBundled REQUIRED)
endif()
include_directories(${NACL_INCLUDE_DIRS})

set(CMAKE_EXE_LINKER_FLAGS "${LDFLAGS}")

add_subdirectory(admin)
add_subdirectory(benc)
add_subdirectory(crypto)
add_subdirectory(dht)
add_subdirectory(interface)
add_subdirectory(io)
add_subdirectory(memory)
add_subdirectory(util)
add_subdirectory(switch)
add_subdirectory(net)
add_subdirectory(exception)
add_subdirectory(tunnel)

add_executable(cleanconfig
    cleanconfig.c
)
target_link_libraries(cleanconfig
    cjdbenc_JsonBencSerializer
    cjdbenc
    cjdmemory
    cjdio
)
add_executable(benc2json
    benc2json.c
)
target_link_libraries(benc2json
    cjdbenc_StandardBencSerializer
    cjdbenc_JsonBencSerializer
    cjdbenc
    cjdmemory
    cjdio
)
add_executable(privatetopublic
    privatetopublic.c
)
target_link_libraries(privatetopublic
    util
    crypto
)
add_executable(publictoip6
    publictoip6.c
)
target_link_libraries(publictoip6
    util
    cjdns-crypto-key
)

enable_testing()
add_subdirectory(test)
