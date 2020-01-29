# How to set up your project

You can find resources about CMake in `CMake-Tutorial.md`.

Following are instructions about how to set up project environment from the last semester.
(This setup guide is for building gRPC from source. If you are installing gRPC with `apt`, the installation and the compilation parts will be different.)

## Environment

Using Vagrant box:
ubuntu/bionic64 (virtualbox, 20190109.0.0)
Vagrant file is attached.

## Project Dependency

### Globally installed

1. gRPC
2. Protobuf
3. CMake - as build tool

### Git submodule

(For people who are not familiar with submodule: https://www.atlassian.com/git/tutorials/git-submodule)

1. googletest
2. gflags
3. benchmark
4. glog

## Install

### Install global modules

```bash
#  gRPC
# Pre-requisites
[sudo] apt-get install build-essential autoconf libtool pkg-config
# Install from source
git clone -b $(curl -L https://grpc.io/release) https://github.com/grpc/grpc
cd grpc
git submodule update --init
# From the grpc repository root
make
make install
# Protobuf
cd grpc/third_party/protobuf
sudo make install   # 'make' should have been run by core grpc

#cmake
# install PPA
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:george-edison55/cmake-3.x
sudo apt-get update
# if cmake is not installed
sudo apt-get install cmake
# else
sudo apt-get upgrade
```

```bash
# install submodule dependency
git submodule init
git submodule update
# Install packages
# google test
cd third_party/googletest
cmake .
make
sudo make install
# gflags
cd third_party/gflags
cmake .
make
sudo make install
# in root folder
sudo cmake .
make
```
