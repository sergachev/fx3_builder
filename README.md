**docker-based build environment for Cypress FX3 firmwares**

Requirements:
 - docker, docker-compose
 - builder/fx3_firmware_linux.tar.gz - unpack from FX3_SDK_1.3.4_Linux.tar.gz downloadable from 
 https://www.cypress.com/documentation/software-and-drivers/ez-usb-fx3-software-development-kit
 - builder/gcc-arm-none-eabi-8-2018-q4-major-linux.tar.bz2 downloadable from https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads
 
Usage:
 - `docker-compose build` - prepares a docker image with cmake, arm gcc and fx3 sdk
 - `docker-compose up` builds the firmware in project/build/output/
 - replace example.c in project/src/ with your fx3 sources (*.c, *.h)
 - change FX3 memory size (512k/256k) in CMakeLists.txt if needed
 
Credits:
    cmake files taken from https://github.com/Nuand/bladeRF