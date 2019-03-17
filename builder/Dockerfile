FROM ubuntu:18.04

RUN apt-get update && \
    apt-get install -y \
      cmake \
      git \
    && apt-get clean
	
WORKDIR /fx3

ADD fx3_firmware_linux.tar.gz ./
ADD gcc-arm-none-eabi-8-2018-q4-major-linux.tar.bz2 ./
