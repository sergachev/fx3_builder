cmake_minimum_required(VERSION 3.6.3)



set(FX3_SIZE                "512k")

set(FX3_SDK_VERSION         "1.3.4")
set(ARM_TOOLCHAIN_VERSION   "gcc-arm-none-eabi-8-2018-q4-major")
set(GCC_VERSION             "8.2.1")

set(VERSION_INFO_MAJOR 1)
set(VERSION_INFO_MINOR 0)
set(VERSION_INFO_PATCH 0)



string(REPLACE "." "_" FX3_SDK_VERSION_PATH "${FX3_SDK_VERSION}")

set(ARM_NONE_EABI_PATH  "${FX3_INSTALL_PATH}/${ARM_TOOLCHAIN_VERSION}")
set(FX3_FWLIB_DIR "${FX3_INSTALL_PATH}/cyfx3sdk/fw_lib/${FX3_SDK_VERSION_PATH}")
set(FX3_FW_COMMON_DIR "${FX3_INSTALL_PATH}/cyfx3sdk/firmware/common")
set(FX3_ELF2IMG "${FX3_INSTALL_PATH}/cyfx3sdk/util/elf2img/elf2img.c")

if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
    set(FX3_LIBRARY_DIR "${FX3_FWLIB_DIR}/fx3_debug")
else()
    set(FX3_LIBRARY_DIR "${FX3_FWLIB_DIR}/fx3_release")
endif()

# set(FX3_INSTALL_PATH ${FX3_INSTALL_PATH}                      CACHE PATH "Path to FX3 SDK")
set(FX3_INCLUDE_DIR "${FX3_FWLIB_DIR}/inc"                      CACHE PATH "Path to FX3 header directory")
set(FX3_LIBRARY_DIR "${FX3_LIBRARY_DIR}"                        CACHE PATH "FX3 Library directory")
set(FX3_FW_COMMON_DIR "${FX3_FW_COMMON_DIR}"                    CACHE PATH "FX3 SDK directory containing common linker and startup source files")
set(FX3_LINKER_FILE "${FX3_FW_COMMON_DIR}/fx3_${FX3_SIZE}.ld"   CACHE PATH "FX3 Linker script")
set(FX3_ELF2IMG "${FX3_ELF2IMG}"                                CACHE PATH "FX3 ELF to boot image converter")

set(ARM_NONE_EABI_LIBGCC
    "${ARM_NONE_EABI_PATH}/lib/gcc/arm-none-eabi/${GCC_VERSION}/libgcc.a"
    CACHE PATH "Path to FX3 SDK's libgcc.a")

set(ARM_NONE_EABI_LIBC "${ARM_NONE_EABI_PATH}/arm-none-eabi/lib/libc.a"
    CACHE PATH "Path to FX3 SDK's libc.a")

set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
set(CMAKE_C_COMPILER "${ARM_NONE_EABI_PATH}/bin/arm-none-eabi-gcc")


set(CMAKE_SYSTEM_NAME Generic)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH ${ARM_NONE_EABI_PATH})

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/output)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/output)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/output)
set(CMAKE_HELPERS_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/cmake/helpers)
set(CMAKE_HELPERS_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/cmake/helpers)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/cmake)



project(fx3_fw C)


if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
    message(STATUS
        "Build type not specified. Defaulting to: ${CMAKE_BUILD_TYPE}"
    )
endif()
set(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "Build type")

if(NOT DEFINED VERSION_INFO_EXTRA)
    set(VERSION_INFO_EXTRA "git")
endif()
include(Version)
set(VERSION "${VERSION_INFO}")
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/include/version.h
    @ONLY
)

set(INCLUDE_DIRS
    "${CMAKE_CURRENT_SOURCE_DIR}"
    "${CMAKE_CURRENT_BINARY_DIR}/include"
    "${FX3_INCLUDE_DIR}")

set(FX3_FW fx3_fw_v${VERSION_INFO_BASE})

file(GLOB_RECURSE SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "src/*.c")

set(SRC_FX3
    "${FX3_FW_COMMON_DIR}/cyfx_gcc_startup.S"
    "${FX3_FW_COMMON_DIR}/cyfxtx.c"
    "${SOURCES}"
)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_C_FLAGS "-O0 -g")
else()
    set(CMAKE_C_FLAGS "-O2")
endif()

set(CMAKE_C_FLAGS
    "${CMAKE_C_FLAGS} -Wall -Wextra -Wno-unused-parameter -DCYU3P_FX3=1 \
     -D__CYU3P_TX__=1 -mcpu=arm926ej-s -mthumb-interwork"
)

include_directories(${INCLUDE_DIRS})
enable_language(ASM)

set(CMAKE_EXE_LINKER_FLAGS
    "-Wl,--entry,CyU3PFirmwareEntry \
     -Wl,-T,\"${FX3_LINKER_FILE}\" \
     -Wl,-d -Wl,--gc-sections \
     -Wl,--no-wchar-size-warning"
)

include(ExternalProject)
ExternalProject_Add(elf2img
    DOWNLOAD_COMMAND ""
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/util/elf2img/"
    CMAKE_ARGS "-DFX3_ELF2IMG=${FX3_ELF2IMG};-DCMAKE_RUNTIME_OUTPUT_DIRECTORY=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}"
    INSTALL_COMMAND ""
)
set(ELF2IMG ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/elf2img)

add_executable(${FX3_FW}.elf ${SRC_FX3})
target_link_libraries(${FX3_FW}.elf
	"${ARM_NONE_EABI_LIBC}"
	"${ARM_NONE_EABI_LIBGCC}"
    "${FX3_LIBRARY_DIR}/libcyfxapi.a"
    "${FX3_LIBRARY_DIR}/libcyu3lpp.a"
    "${FX3_LIBRARY_DIR}/libcyu3threadx.a"
)
set_target_properties(${FX3_FW}.elf
	PROPERTIES LINK_FLAGS "-Wl,-Map,output/${FX3_FW}.map")

set(ELF2IMG_WORK_DIR "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")

add_custom_command(
	OUTPUT ${ELF2IMG_WORK_DIR}/${FX3_FW}.img
	COMMAND ${ELF2IMG} -i ${ELF2IMG_WORK_DIR}/${FX3_FW}.elf -o ${ELF2IMG_WORK_DIR}/${FX3_FW}.img
	DEPENDS ${ELF2IMG_WORK_DIR}/${FX3_FW}.elf
	COMMENT "Creating ${FX3_FW}.img. Interrupt vector removal note is expected."
)

add_custom_target(${FX3_FW}.img ALL
	DEPENDS elf2img ${ELF2IMG_WORK_DIR}/${FX3_FW}.elf ${ELF2IMG_WORK_DIR}/${FX3_FW}.img)
