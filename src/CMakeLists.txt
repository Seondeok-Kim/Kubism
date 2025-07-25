cmake_minimum_required(VERSION 3.2)
project(Kubism)

set(CMAKE_MODULE_PATH ${CMAKE_HOME_DIRECTORY}/cmake)

# Find OpenMP and add it if requiredfind_package(OpenMP REQUIRED)

if (APPLE AND NOT CUDA_HOST_COMPILER)
  # https://gitlab.kitware.com/cmake/cmake/issues/13674
  message(WARNING "CUDA_HOST_COMPILER is set to the default /usr/bin/clang; CUDA 8.0 requires 7.3 or older.")
  set(CUDA_HOST_COMPILER "/usr/bin/clang" CACHE FILEPATH "" FORCE)
endif()

find_package(CUDA QUIET REQUIRED)

if (NOT DISABLE_PYTHON)
  if (APPLE)
    # workaround
    # https://github.com/Homebrew/legacy-homebrew/issues/25118
    # https://cmake.org/Bug/view.php?id=14809
    find_program(PYTHON_CONFIG_EXECUTABLE python3-config)
    message("-- Found python3-config: ${PYTHON_CONFIG_EXECUTABLE}")
    execute_process(COMMAND ${PYTHON_CONFIG_EXECUTABLE} --prefix OUTPUT_VARIABLE PYTHON_PREFIX OUTPUT_STRIP_TRAILING_WHITESPACE)
    message("-- Discovered Python 3.x prefix: ${PYTHON_PREFIX}")
    set(PYTHON_EXECUTABLE "${PYTHON_PREFIX}/bin/python3")
  endif()
  find_package(PythonInterp 3 REQUIRED)
  find_package(PythonLibs 3 REQUIRED)
  if (NOT NUMPY_INCLUDES)
    execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "import numpy; print(numpy.get_include())" OUTPUT_VARIABLE NUMPY_INCLUDES)
  endif()
endif()

if (NOT DISABLE_R)
  find_package(R)
endif()

if (PROFILE OR CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DPROFILE")
endif()

#set(CMAKE_VERBOSE_MAKEFILE on)

if (NOT DEFINED CUDA_ARCH)
  set(CUDA_ARCH "87")
endif()

# Add OpenMP flags to general C++ compiler flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native -Wall -Werror -DCUDA_ARCH=${CUDA_ARCH} -std=c++11 ${OpenMP_CXX_FLAGS}")

# Flags to use when compiling CUDA source files
set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -arch sm_${CUDA_ARCH} -Xptxas=-v -D_MWAITXINTRIN_H_INCLUDED -D_FORCE_INLINES -lineinfo")

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(NVCC_FLAGS "-G -g")
endif()

if (CMAKE_MAJOR_VERSION LESS 4 AND CMAKE_MINOR_VERSION LESS 3)
  # workaround https://github.com/Kitware/CMake/commit/99abebdea01b9ef73e091db5594553f7b1694a1b
  message(STATUS "Applied CUDA C++11 workaround on CMake < 3.3")
  set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} --std c++11")
endif()

if (APPLE)
  message(STATUS "Applied CUDA OpenMP macOS workaround")
  set(CUDA_PROPAGATE_HOST_FLAGS OFF)
  set(CMAKE_SHARED_LIBRARY_CXX_FLAGS_BACKUP "${CMAKE_SHARED_LIBRARY_CXX_FLAGS}")
  set(CMAKE_SHARED_LIBRARY_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_CXX_FLAGS} ${CMAKE_CXX_FLAGS} -Wno-unused-function")
  string(REGEX REPLACE "-fopenmp[^ ]*" "" CMAKE_SHARED_LIBRARY_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_CXX_FLAGS}")
endif()

set(SOURCE_FILES kmcuda.cc kmcuda.h wrappers.h private.h fp_abstraction.h tricks.cuh
                 metric_abstraction.h kmeans.cu knn.cu transpose.cu kmeans_cpu.cc kmeans_cpu.h)

if (PYTHONLIBS_FOUND)
  list(APPEND SOURCE_FILES python.cc)
endif()

if (R_FOUND)
  list(APPEND SOURCE_FILES r.cc)
endif()

cuda_add_library(KMCUDA SHARED ${SOURCE_FILES} OPTIONS ${NVCC_FLAGS})

if (APPLE)
  set(CMAKE_SHARED_LIBRARY_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_CXX_FLAGS_BACKUP}")
endif()

# Link the OpenMP library to the KMCUDA target.
# When using both CUDA and OpenMP, link only the OpenMP library with g++ or clang++.
target_link_libraries(KMCUDA ${CUDA_curand_LIBRARY} OpenMP::OpenMP_CXX)

if (PYTHONLIBS_FOUND)
  include_directories(${PYTHON_INCLUDE_DIRS} ${NUMPY_INCLUDES})
  target_link_libraries(KMCUDA ${PYTHON_LIBRARIES})
endif()

if (R_FOUND)
  include_directories(${R_INCLUDE_DIRS})
  target_link_libraries(KMCUDA ${R_LIBRARIES})
endif()

if (SUFFIX)
  set_target_properties(KMCUDA PROPERTIES SUFFIX ${SUFFIX})
endif()
