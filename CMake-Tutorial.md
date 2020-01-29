# A very brief introduction of CMake

## What is CMake

In short, CMake is a cross-platform tool to control the compilation process. CMake can generate the native makefiles for you with `CMakeLists.txt`.

## Why do we want to use it

CMake helps us to manage the compilation process. For smaller projects, it is possible to manually write all of the compilation commands.
However, when the scale of the project increases, the complexity of the compilation process will also increase, especially when you want to support different platforms.
CMake helps to reduce the complexity of the compilation process with more readable syntax and its ability to automatically generate makefiles for different platform.

## Installation

```
# Using box ubuntu/bionic64
#cmake

# install PPA
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:george-edison55/cmake-3.x
sudo apt-get update

# if cmake is not installed
sudo apt-get install cmake

# else
sudo apt-get upgrade
[More Platforms]
```

## How to run it

# In folder containing CMakeList.txt

```
sudo cmake .
make
```

## A basic template

```
# Adapted From https://cliutils.gitlab.io/modern-cmake/chapters/basics/example.html
# Almost all CMake files should start with this
# You should always specify a range with the newest
# and oldest tested versions of CMake. This will ensure
# you pick up the best policies.
cmake_minimum_required(VERSION 3.1...3.16)

# This is your project statement. You should always list languages;
# Listing the version is nice here since it sets lots of useful variables
project(ModernCMakeExample VERSION 1.0 LANGUAGES CXX)

# Specify C++ version your are using
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -pthread")

# If you set any CMAKE_ variables, that can go here.
# (But usually don't do this, except maybe for C++ standard)

# Find packages go here.

# You should usually split this into folders, but this is a simple example

# This is a "default" library, and will match the *** variable setting.
# Other common choices are STATIC, SHARED, and MODULE
# Including header files here helps IDEs but is not required.
# Output libname matches target name, with the usual extensions on your systemadd_library(MyLibExample simple_lib.cpp simple_lib.hpp)

# Link each target with other targets or add options, etc.

# Adding something we can run - Output name matches target name
# To change the location of the executables, check https://stackoverflow.com/questions/9345792/cmake-executable-location
add_executable(MyExample simple_example.cpp)

# Make sure you link your targets with this command. It can also link libraries and
# even flags, so linking a target that does not exist will not give a configure-time error.
# For the meaning of PRIVATE, check https://cmake.org/pipermail/cmake/2016-May/063400.html
target_link_libraries(MyExample PRIVATE MyLibExample)
```

## How to separate your components

In previous example, the comment stated that part of the CMakeList.txt should go to different folders.
Since this project is composed of several components, separating different components into their own folders can help making the project structure clean.

Following examples are based on this project structure:

```
.
├── CMakeLists.txt
├── README.md
├── Vagrantfile
├── protos
├── store
└── third_party
```

(`store` folder contains its own `CMakeLists.txt`)

### Creating folders for different components

The first step is to separate files of different components into their own folders.
Then you can place `CMakeLists.txt` for each component into its own folder.

```
cmake_minimum_required(VERSION 3.3)

# Gets proto filenames
get_filename_component(store_proto "../protos/key_value_store.proto" ABSOLUTE)
get_filename_component(store_proto_path "${store_proto}" PATH)

# Generates gRPC codes
set(DIST_DIR "${CMAKE_CURRENT_BINARY_DIR}/dist")
file(MAKE_DIRECTORY ${DIST_DIR})
set(store_proto_srcs "${DIST_DIR}/key_value_store.pb.cc")
set(store_proto_hdrs "${DIST_DIR}/key_value_store.pb.h")
set(store_grpc_srcs "${DIST_DIR}/key_value_store.grpc.pb.cc")
set(store_grpc_hdrs "${DIST_DIR}/key_value_store.grpc.pb.h")
add_custom_command(
      OUTPUT "${store_proto_srcs}" "${store_proto_hdrs}" "${store_grpc_srcs}" "${store_grpc_hdrs}"
      COMMAND ${_PROTOBUF_PROTOC}
      ARGS --grpc_out "${DIST_DIR}"
        --cpp_out "${DIST_DIR}"
        -I "${store_proto_path}"
        --plugin=protoc-gen-grpc="${_GRPC_CPP_PLUGIN_EXECUTABLE}"
        "${store_proto}"
      DEPENDS ${store_proto})

# Include generated *.pb.h files
include_directories("${DIST_DIR}")

set(client_interface_hdrs "key_value_store_client_interface.h")

# Compiles store as a library
add_library(store STATIC store.cc store.h)

# Adds test
add_executable(store_test store_test.cc)
target_link_libraries(store_test PUBLIC
    gtest
    gtest_main
    store
)
add_test(
    NAME store_test
    COMMAND store_test
)

# Compiles key_value_store_server_sync
add_executable(key_value_store_server_sync key_value_store_server_sync.cc
    key_value_store_server_sync.h
    ${store_proto_srcs}
    ${store_grpc_srcs})
target_link_libraries(key_value_store_server_sync
    ${_GRPC_GRPCPP_UNSECURE}
    ${_PROTOBUF_LIBPROTOBUF}
    glog::glog
    store)

# ...
```

### How compile sub-directories

In root-level CMakeList.txt, you can use `add_subdirectory` to include the compilation of a specific folder into the overall compilation process.

```
cmake_minimum_required(VERSION 3.3)
project(projectname C CXX)

# use cpp 11
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -pthread")
include_directories(/usr/local/include)
link_directories(/usr/local/lib)

# Find Protobuf installation
# Looks for protobuf-config.cmake file installed by Protobuf's CMake installation.
set(protobuf_MODULE_COMPATIBLE TRUE)
find_package(Protobuf REQUIRED)
include_directories(${Protobuf_INCLUDE_DIRS})
message(STATUS "Using protobuf ${protobuf_VERSION}")
set(_PROTOBUF_LIBPROTOBUF protobuf::libprotobuf)
set(_PROTOBUF_PROTOC $<TARGET_FILE:protobuf::protoc>)

# Hardcode gRPC installation
set(_GRPC_GRPCPP_UNSECURE grpc++_unsecure)
set(_GRPC_CPP_PLUGIN_EXECUTABLE /usr/local/bin/grpc_cpp_plugin)

set(THIRD_PARTY_DIR "third_party")

# Add googletest
add_subdirectory(${THIRD_PARTY_DIR}/googletest)
enable_testing()
# Add gflag
# set(BENCHMARK_ENABLE_TESTING OFF CACHE BOOL "Suppressing benchmark's tests" FORCE)
find_package(gflags REQUIRED)
# Add benchmark
add_subdirectory(${THIRD_PARTY_DIR}/benchmark)
# Add glog
add_subdirectory(${THIRD_PARTY_DIR}/glog)

# add subdirectories
# Notice here
add_subdirectory(store)
```

## Iterating over similar files

If several executables/libraries have similar dependencies, you can use `foreach` to reduce duplications.

```
# From https://github.com/grpc/grpc/blob/master/examples/cpp/helloworld/CMakeLists.txt
foreach(\_target
greeter_client greeter_server
greeter_async_client greeter_async_client2 greeter_async_server)
add_executable(${_target} "${\_target}.cc"
${hw_proto_srcs}
    ${hw_grpc_srcs})
target_link_libraries(${_target}
    ${\_GRPC_GRPCPP_UNSECURE}
\${\_PROTOBUF_LIBPROTOBUF})
endforeach()
```

## How to use CMake with gRPC

Please refer to following links for how to use gRPC with CMake
[gRPC CPP basic](https://grpc.io/docs/tutorials/basic/cpp/)
[gRPC make tutorial](https://github.com/grpc/grpc/blob/master/src/cpp/readme.md#make)
[CMake from official example](https://github.com/grpc/grpc/blob/master/examples/cpp/helloworld/CMakeLists.txt)

## Resources

[Modern CMake](https://cliutils.gitlab.io/modern-cmake/)
[official tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)
[gRPC CMake Example](https://github.com/grpc/grpc/blob/master/examples/cpp/helloworld/CMakeLists.txt)
