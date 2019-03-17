cmake_minimum_required(VERSION 2.8.3)
project(elf2img C)

if(NOT FX3_ELF2IMG)
    message("FX3_ELF2IMG must be defined as the location to elf2img.c")
endif()

add_executable(elf2img ${FX3_ELF2IMG})
