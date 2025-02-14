cmake_minimum_required(VERSION 3.8 FATAL_ERROR)
set(PROJECT_NAME LgFaceRecDemoTCP_Jetson_NanoV2)
project(${PROJECT_NAME})# LANGUAGES CXX CUDA)

set (CMAKE_CXX_STANDARD 11)

# openssl
find_package(OpenSSL REQUIRED)
if(OPENSSL_FOUND)
message("-- OPENSSL version: ${OPENSSL_VERSION}")
endif()

# glib
find_package(PkgConfig REQUIRED)
pkg_search_module(GLIB REQUIRED glib-2.0)
message("-- GLIB version: ${GLIB_VERSION}")
include_directories(${GLIB_INCLUDE_DIRS})
lINK_DIRECTORIES(${GLIB_LIBRARY_DIRS})

# OpenCV
find_package(OpenCV REQUIRED)
find_package(CUDA)
message("-- CUDA version: ${CUDA_VERSION}")

set(
	CUDA_NVCC_FLAGS
	${CUDA_NVCC_FLAGS}; 
    -O3 
	-gencode arch=compute_53,code=sm_53
	-gencode arch=compute_62,code=sm_62
)

if(CUDA_VERSION_MAJOR GREATER 9)
	message("-- CUDA ${CUDA_VERSION_MAJOR} detected, enabling SM_72")

	set(
		CUDA_NVCC_FLAGS
		${CUDA_NVCC_FLAGS}; 
		-gencode arch=compute_72,code=sm_72
	)

endif()

if(USE_ADDRESS_SANITIZER)
    set(CMAKE_CXX_FLAGS  "-Wno-deprecated-declarations -fno-omit-frame-pointer -fsanitize=address")
else()
    set(CMAKE_CXX_FLAGS  "-Wno-deprecated-declarations")
endif()

# tensorRT
message("CUDA_TOOLKIT_ROOT_DIR = ${CUDA_TOOLKIT_ROOT_DIR}")

find_path(TENSORRT_INCLUDE_DIR NvInfer.h
  HINTS ${TENSORRT_ROOT} ${CUDA_TOOLKIT_ROOT_DIR} /usr/include
  PATH_SUFFIXES include)
find_path(TENSORRT_INCLUDE_DIR NvInferPlugin.h
  HINTS ${TENSORRT_ROOT} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES include)
MESSAGE(STATUS "Found TensorRT headers at ${TENSORRT_INCLUDE_DIR}")
find_library(TENSORRT_LIBRARY_INFER nvinfer
  HINTS ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64 lib/aarch64-linux-gnu)
find_library(TENSORRT_LIBRARY_INFER_PLUGIN nvinfer_plugin
  HINTS  ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64 lib/aarch64-linux-gnu)
  find_library(TENSORRT_LIBRARY_PARSER nvparsers
  HINTS  ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64 lib/aarch64-linux-gnu)
set(TENSORRT_LIBRARY ${TENSORRT_LIBRARY_INFER} ${TENSORRT_LIBRARY_INFER_PLUGIN} ${TENSORRT_LIBRARY_PARSER})
message(STATUS "Find TensorRT libs at ${TENSORRT_LIBRARY}")
include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(
  TENSORRT DEFAULT_MSG TENSORRT_INCLUDE_DIR TENSORRT_LIBRARY)
if(NOT TENSORRT_FOUND)
  message(ERROR
    "Cannot find TensorRT library.")
endif()
find_library(LIBSSL NAMES libssl.so HINTS lib/aarch64-linux-gnu REQUIRED)
find_library(LIBCRYPTO NAMES libcrypto.so HINTS lib/aarch64-linux-gnu REQUIRED)

# l2norm_helper plugin
add_subdirectory(trt_l2norm_helper)
include_directories(
  trt_l2norm_helper
  ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}
  ${TENSORRT_INCLUDE_DIR}
)

message("TENSORRT_LIBRARY = ${TENSORRT_LIBRARY}")

AUX_SOURCE_DIRECTORY(./src DIR_SRCS)
message("DIR_SRCS = ${DIR_SRCS}")
cuda_add_executable(${PROJECT_NAME} ${DIR_SRCS})


# AUX_SOURCE_DIRECTORY(./UserRegister DIR_USERREGISTER)
# message("DIR_USERREGISTER = ${DIR_USERREGISTER}")
cuda_add_executable(UserRegister 
  ./UserRegister/main.cpp
  ./src/accessHistory.cpp
  ./src/logger.cpp
  ./src/faceNet.cpp
  ./src/mtcnn.cpp
  ./src/pnet_rt.cpp
  ./src/common.cpp
  ./src/baseEngine.cpp
  ./src/rnet_rt.cpp
  ./src/onet_rt.cpp
  ./src/network.cpp
  ./src/cctvCrypto.cpp
  ./src/key_manager.cpp
)

include_directories(
  UserRegister
  ./src
  ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}
  ${TENSORRT_INCLUDE_DIR}
  
)

target_link_libraries(${PROJECT_NAME}
  trt_l2norm_helper
  ${TENSORRT_LIBRARY}
  ${LIBSSL}
  ${LIBCRYPTO}
  keyutils
)

target_link_libraries(UserRegister
  trt_l2norm_helper
  ${TENSORRT_LIBRARY}
  ${LIBCRYPTO}
  ${OpenCV_LIBS}
  ${GLIB_LIBRARIES}
  stdc++fs
  keyutils
)

target_link_libraries(${PROJECT_NAME} ${OpenCV_LIBS} ${GLIB_LIBRARIES})

add_executable(cryptotest
  ./src/cryptoTest/cryptoTest.cpp
  ./src/cctvCrypto.cpp
  ./src/key_manager.cpp
)
target_link_libraries(cryptotest
  ssl
  crypto
  ${GLIB_LIBRARIES}
  keyutils
)
