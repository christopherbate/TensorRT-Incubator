cmake_minimum_required(VERSION 3.25)
project(mlir-executor LANGUAGES C CXX)

#-------------------------------------------------------------------------------
# Options and settings
#-------------------------------------------------------------------------------

macro(mlir_executor_option name)
  option(${name} ${ARGN})
  list(APPEND MLIR_EXECUTOR_FEATURE_FLAGS ${name})
endmacro()

mlir_executor_option(MLIR_EXECUTOR_ENABLE_CUBLAS "Enable use of CUBLAS in the runtime" ON)
mlir_executor_option(MLIR_EXECUTOR_ENABLE_NCCL "Enable use of NCCL in the runtime" ON)
mlir_executor_option(MLIR_EXECUTOR_ENABLE_MPI "Enable use of MPI in the runtime" ON)

#-------------------------------------------------------------------------------
# Project Setup
#-------------------------------------------------------------------------------

set(CMAKE_BUILD_WITH_INSTALL_NAME_DIR ON)

set(CMAKE_CXX_STANDARD 17 CACHE STRING "C++ standard to conform to")

if(PROJECT_IS_TOP_LEVEL)
  message(STATUS "Building MLIR-Executor as standalone project")
  find_package(MLIR REQUIRED CONFIG)

  message(STATUS "Using MLIRConfig.cmake in: ${MLIR_DIR}")
  message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

  set(LLVM_RUNTIME_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/bin)
  set(LLVM_LIBRARY_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/lib)
  set(MLIR_BINARY_DIR ${CMAKE_BINARY_DIR})

  list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
  list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
elseif(NOT PROJECT_IS_TOP_LEVEL AND NOT MLIR_BINARY_DIR)
  # Build via external projects mechanism
  message(STATUS "Building MLIR-Executor via LLVM_EXTERNAL_PROJECTS")
  set(MLIR_MAIN_SRC_DIR ${LLVM_MAIN_SRC_DIR}/../mlir)
  set(MLIR_INCLUDE_DIR ${MLIR_MAIN_SRC_DIR}/include)
  set(MLIR_GENERATED_INCLUDE_DIR ${LLVM_BINARY_DIR}/tools/mlir/include)
  set(MLIR_INCLUDE_DIRS "${MLIR_INCLUDE_DIR};${MLIR_GENERATED_INCLUDE_DIR}")
else()
  message(STATUS "Building MLIR-Executor as sub-project")
  set(LLVM_RUNTIME_OUTPUT_INTDIR ${CMAKE_CURRENT_BINARY_DIR}/bin)
  set(LLVM_LIBRARY_OUTPUT_INTDIR ${CMAKE_CURRENT_BINARY_DIR}/lib)
endif()

include(TableGen)
include(AddLLVM)
include(AddMLIR)
include(HandleLLVMOptions)

if(MLIR_ENABLE_BINDINGS_PYTHON)
  include(MLIRDetectPythonEnv)
  mlir_configure_python_dev_packages()
endif()

include(cmake/ExecutorDependencies.cmake)
include(cmake/ExecutorFunctions.cmake)

#-------------------------------------------------------------------------------
# Dependencies
#-------------------------------------------------------------------------------
find_package(CUDAToolkit REQUIRED)

mlir_executor_add_nvtx()
mlir_executor_add_lua()
mlir_executor_add_sol2()
mlir_executor_add_flatbuffers()
mlir_executor_find_and_patch_libnvptxcompiler(CUDANVPTXCompilerLibraryPatched)
mlir_tensorrt_find_dlpack()

find_package(MPI COMPONENTS C)
mlir_executor_find_nccl()

if(MLIR_EXECUTOR_ENABLE_NCCL AND NOT TARGET NCCL)
  message(FATAL_ERROR "MLIR_EXECUTOR_ENABLE_NCCL=ON but NCCL was not found")
endif()
if(MLIR_EXECUTOR_ENABLE_MPI AND NOT MPI_FOUND)
  message(FATAL_ERROR "MLIR_EXECUTOR_ENABLE_MPI=ON but MPI was not found")
endif()

#-------------------------------------------------------------------------------
# Project Targets
#-------------------------------------------------------------------------------
set(MLIR_EXECUTOR_DEFINITIONS)
foreach(feature_flag IN LISTS MLIR_EXECUTOR_FEATURE_FLAGS)
  if(${feature_flag})
    list(APPEND MLIR_EXECUTOR_DEFINITIONS "-D${feature_flag}")
  endif()
endforeach()

set(MLIR_EXECUTOR_SOURCE_DIR ${PROJECT_SOURCE_DIR})
set(MLIR_EXECUTOR_BINARY_DIR ${PROJECT_BINARY_DIR})
include_directories(${LLVM_INCLUDE_DIRS})
include_directories(${MLIR_INCLUDE_DIRS})
include_directories(${MLIR_EXECUTOR_SOURCE_DIR}/include)
include_directories(${MLIR_EXECUTOR_BINARY_DIR}/include)
link_directories(${LLVM_BUILD_LIBRARY_DIR})
add_definitions(${LLVM_DEFINITIONS} ${MLIR_EXECUTOR_DEFINITIONS})

add_subdirectory(executor-tblgen)
add_subdirectory(include/mlir-executor)
add_subdirectory(lib)
add_subdirectory(executor-opt)
add_subdirectory(executor-translate)
add_subdirectory(executor-runner)
add_subdirectory(test)

get_property(MLIR_EXECUTOR_LIBS GLOBAL PROPERTY MLIR_EXECUTOR_LIBS)
add_library(MLIRTensorRTExecutorLibraries INTERFACE)
target_link_libraries(MLIRTensorRTExecutorLibraries INTERFACE
  ${MLIR_EXECUTOR_LIBS})
add_library(MLIRTensorRT::Executor ALIAS MLIRTensorRTExecutorLibraries)
