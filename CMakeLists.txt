# Copyright 2023 Caleb Magruder

cmake_minimum_required(VERSION 3.21)

project(Matrix)

include(CTest)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(CMAKE_BUILD_TYPE Debug)

###############################################################################
############################  Configure Options  ##############################
###############################################################################

option(LINT "Run cpplint and cppcheck linting" OFF)

option(USE_ACC "BLAS: Apple's Accelerate Framework" OFF)

option(USE_OPB "BLAS: OpenBLAS Library" OFF)

option(USE_MKL "BLAS: Intel's Math Kernel Library" OFF)

if (${LINT})
    # Directory build/ does not need to be linted
    set(CMAKE_CXX_CPPLINT "cpplint;--quiet;--exclude=${CMAKE_BINARY_DIR}/*")
    set(CMAKE_CXX_CPPCHECK "cppcheck;--quiet;--suppress=*:${CMAKE_BINARY_DIR}/*")
endif (${LINT})

###############################################################################
##################################  Tests  ####################################
###############################################################################

# If submodule, skip tests and google test dependency.
if(${CMAKE_CURRENT_SOURCE_DIR} STREQUAL ${CMAKE_SOURCE_DIR})
    add_subdirectory(extern)
    add_subdirectory(tests)
endif()

###############################################################################
###############################  Matrix Library  ##############################
###############################################################################

add_library(Matrix SHARED ${CMAKE_CURRENT_SOURCE_DIR}/src/Matrix.cpp
                          ${CMAKE_CURRENT_SOURCE_DIR}/src/Logger.cpp
                          ${CMAKE_CURRENT_SOURCE_DIR}/src/TestWithLogging.cpp)

target_include_directories(Matrix PUBLIC
                           ${CMAKE_CURRENT_SOURCE_DIR}/include)

target_link_libraries(Matrix gtest gtest_main)

##############################  BLAS: Accelerate  #############################

if (${USE_ACC})
    find_library(ACC Accelerate)
    add_library(MatrixACC OBJECT ${CMAKE_CURRENT_SOURCE_DIR}/src/MatrixACC.cpp)
    target_include_directories(MatrixACC PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
                                                ${ACC}/Headers)
    target_link_libraries(Matrix $<TARGET_OBJECTS:MatrixACC>
                                        "-framework Accelerate")
endif ()

#################################  BLAS: MKL  #################################

if (${USE_MKL})
  find_package(MKL QUIET)
  if (${MKL_FOUND})
      find_library(MKL mkl_rt HINTS ${MKL_ROOT}/lib)
      add_library(MatrixMKL OBJECT ${CMAKE_CURRENT_SOURCE_DIR}/src/MatrixMKL.cpp)
      target_include_directories(MatrixMKL PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
                                                  ${MKL_ROOT}/include)
      target_link_libraries(Matrix $<TARGET_OBJECTS:MatrixMKL>
          ${MKL}
          ${MKL_THREAD_LIB})
  else ()
          message(FATAL_ERROR "  MKL Not Found. Set -DUSE_MKL=OFF to skip MKL.")
  endif ()
endif ()

###############################  BLAS: OpenBLAS  ##############################

if (${USE_OPB})
  find_library(OPB openblas HINTS /opt/homebrew/opt/openblas/lib
                                  /usr/local/opt/openblas/lib)
  add_library(MatrixOPB OBJECT ${CMAKE_CURRENT_SOURCE_DIR}/src/MatrixOPB.cpp)
  target_include_directories(MatrixOPB PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
                                              /opt/homebrew/opt/openblas/include
                                              /usr/local/opt/openblas/include)
  target_link_libraries(Matrix $<TARGET_OBJECTS:MatrixOPB>
                              ${OPB})
endif ()

###############################################################################
#############################  Benchmark Profiler  ############################
###############################################################################

configure_file(src/benchmark.cpp.in src/benchmark.cpp)

add_executable(benchmark ${CMAKE_CURRENT_BINARY_DIR}/src/benchmark.cpp)

target_link_libraries(benchmark Matrix)

enable_testing()
