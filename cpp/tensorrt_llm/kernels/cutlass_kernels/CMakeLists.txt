#
# SPDX-FileCopyrightText: Copyright (c) 1993-2024 NVIDIA CORPORATION &
# AFFILIATES. All rights reserved. SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not
# use this file except in compliance with the License. You may obtain a copy of
# the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
# License for the specific language governing permissions and limitations under
# the License.
#

# The Python executable will only be defined if building with Torch support. If
# not, we need to find it here.
if(NOT Python3_EXECUTABLE)
  find_package(
    Python3
    COMPONENTS Interpreter
    REQUIRED)
endif()

execute_process(
  WORKING_DIRECTORY ${3RDPARTY_DIR}/cutlass/
  COMMAND ${Python3_EXECUTABLE} setup_library.py develop --user
  RESULT_VARIABLE _CUTLASS_LIBRARY_SUCCESS)

if(NOT _CUTLASS_LIBRARY_SUCCESS MATCHES 0)
  message(
    FATAL_ERROR
      "Failed to set up the CUTLASS library due to ${_CUTLASS_LIBRARY_SUCCESS}."
  )
endif()

set_directory_properties(
  PROPERTIES CMAKE_CONFIGURE_DEPENDS
             ${CMAKE_CURRENT_SOURCE_DIR}/python/generate_kernels.py)

set(INSTANTIATION_GENERATION_DIR
    ${CMAKE_CURRENT_BINARY_DIR}/cutlass_instantiations)
execute_process(
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/python/
  COMMAND
    ${Python3_EXECUTABLE} generate_kernels.py -a
    "${CMAKE_CUDA_ARCHITECTURES_ORIG};${CMAKE_CUDA_ARCHITECTURES_NATIVE}" -o
    ${INSTANTIATION_GENERATION_DIR}
  RESULT_VARIABLE _KERNEL_GEN_SUCCESS)

if(NOT _KERNEL_GEN_SUCCESS MATCHES 0)
  message(
    FATAL_ERROR
      "Failed to generate CUTLASS kernel instantiations due to ${_KERNEL_GEN_SUCCESS}."
  )
endif()

# Get the sources for Mixed Input GEMM launchers
file(GLOB_RECURSE MIXED_CU_INSTANTIATIONS
     ${INSTANTIATION_GENERATION_DIR}/gemm/*.cu)
file(GLOB_RECURSE MIXED_SRC_CPP fpA_intB_gemm/*.cpp)
file(GLOB_RECURSE MIXED_SRC_CU fpA_intB_gemm/*.cu)

# Get the sources for MOE Grouped GEMM launchers
file(GLOB_RECURSE GROUPED_CU_INSTANTIATIONS
     ${INSTANTIATION_GENERATION_DIR}/gemm_grouped/*.cu)
file(GLOB_RECURSE GROUPED_SRC_CPP moe_gemm/*.cpp)
file(GLOB_RECURSE GROUPED_SRC_CU moe_gemm/*.cu)

# Get the sources for FP8 Rowwise GEMM launchers
file(GLOB_RECURSE FBGEMM_CU_INSTANTIATIONS
     ${INSTANTIATION_GENERATION_DIR}/fp8_rowwise_gemm/*.cu)
file(GLOB_RECURSE FBGEMM_SRC_CU fp8_rowwise_gemm/*.cu)

# Get the sources for FP4 GEMM launchers
file(GLOB_RECURSE FP4_CU_INSTANTIATIONS
     ${INSTANTIATION_GENERATION_DIR}/gemm_fp4/*.cu)
file(GLOB_RECURSE FP4_SRC_CU fp4_gemm/*.cu)

# Get the sources for AllReduce GEMM
file(GLOB_RECURSE ARGEMM_SRC_CU allreduce_gemm/*.cu)

# Get the sources for all the remaining sources
file(GLOB_RECURSE SRC_CPP *.cpp)
file(GLOB_RECURSE SRC_CU *.cu)
set(ALL_SRCS ${SRC_CPP};${SRC_CU})
list(FILTER ALL_SRCS EXCLUDE REGEX "fpA_intB_gemm/.*")
list(FILTER ALL_SRCS EXCLUDE REGEX "moe_gemm/.*")
list(FILTER ALL_SRCS EXCLUDE REGEX "fp8_rowwise_gemm/.*")
list(FILTER ALL_SRCS EXCLUDE REGEX "fp4_gemm/.*")
list(FILTER ALL_SRCS EXCLUDE REGEX "allreduce_gemm/.*")
list(REMOVE_ITEM ALL_SRCS
     "${CMAKE_CURRENT_SOURCE_DIR}/fused_gated_gemm/gemm_swiglu_e4m3.cu")

message(
  VERBOSE
  "Mixed srcs ${MIXED_SRC_CPP} ${MIXED_SRC_CU} ${MIXED_CU_INSTANTIATIONS}")
message(
  VERBOSE
  "Group srcs ${GROUPED_SRC_CU} ${GROUPED_SRC_CPP} ${GROUPED_CU_INSTANTIATIONS}"
)
message(VERBOSE "Fbgemm srcs ${FBGEMM_SRC_CU} ${FBGEMM_CU_INSTANTIATIONS}")
message(VERBOSE "FP4 srcs ${FP4_SRC_CU} ${FP4_CU_INSTANTIATIONS}")
message(VERBOSE "ARgemm srcs ${ARGEMM_SRC_CU}")
message(VERBOSE "All srcs ${ALL_SRCS}")

add_library(cutlass_src STATIC ${ALL_SRCS})
set_property(TARGET cutlass_src PROPERTY POSITION_INDEPENDENT_CODE ON)
set_property(TARGET cutlass_src PROPERTY CUDA_RESOLVE_DEVICE_SYMBOLS ON)

add_library(fpA_intB_gemm_src STATIC ${MIXED_SRC_CPP} ${MIXED_SRC_CU}
                                     ${MIXED_CU_INSTANTIATIONS})
# WARNING: Building with `-G` flag may generate invalid results for this target
# add_library(moe_gemm_src STATIC ${GROUPED_SRC_CU} ${GROUPED_SRC_CPP}
# ${GROUPED_CU_INSTANTIATIONS})
add_library(fb_gemm_src STATIC ${FBGEMM_SRC_CU} ${FBGEMM_CU_INSTANTIATIONS})
add_library(
  ar_gemm_src STATIC
  ${ARGEMM_SRC_CU} ${CMAKE_CURRENT_SOURCE_DIR}/../../runtime/ipcNvlsMemory.cpp)

set(GEMM_SWIGLU_SM90_SRC_CU
    ${CMAKE_CURRENT_SOURCE_DIR}/fused_gated_gemm/gemm_swiglu_e4m3.cu)
add_library(gemm_swiglu_sm90_src STATIC ${GEMM_SWIGLU_SM90_SRC_CU})

function(process_target target_name enable_hopper enable_blackwell)
  set_property(TARGET ${target_name} PROPERTY POSITION_INDEPENDENT_CODE ON)
  set_property(TARGET ${target_name} PROPERTY CUDA_RESOLVE_DEVICE_SYMBOLS ON)

  # Note - we deliberately do not include 90a PTX (even when 9.0+PTX is
  # specified). This is because sm_90a has arch conditional instructions that
  # are not forward compatible. As a result, it does not make sense to embed PTX
  # into the binary anyway.
  if(${enable_hopper}
     AND ("90" IN_LIST CMAKE_CUDA_ARCHITECTURES_ORIG
          OR "90-real" IN_LIST CMAKE_CUDA_ARCHITECTURES_ORIG
          OR "90-real" IN_LIST CMAKE_CUDA_ARCHITECTURES_NATIVE))

    message(STATUS "MANUALLY APPENDING FLAG TO COMPILE FOR SM_90a.")
    target_compile_options(
      ${target_name}
      PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-gencode=arch=compute_90a,code=sm_90a>)

    # Hopper kernels require cuda lib for TMA APIs
    target_link_libraries(${target_name} PRIVATE CUDA::cuda_driver
                                                 ${MPI_C_LIBRARIES})

    # No kernels should be parsed, unless hopper is specified. This is a build
    # time improvement
    target_compile_definitions(${target_name} PRIVATE COMPILE_HOPPER_TMA_GEMMS)
    target_compile_definitions(${target_name}
                               PRIVATE COMPILE_HOPPER_TMA_GROUPED_GEMMS)
  endif()

  if(${enable_blackwell}
     AND ("100" IN_LIST CMAKE_CUDA_ARCHITECTURES_ORIG
          OR "100-real" IN_LIST CMAKE_CUDA_ARCHITECTURES_ORIG
          OR "100-real" IN_LIST CMAKE_CUDA_ARCHITECTURES_NATIVE))

    message(STATUS "MANUALLY APPENDING FLAG TO COMPILE FOR SM_100a.")
    target_compile_options(
      ${target_name}
      PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-gencode=arch=compute_100a,code=sm_100a
              -Xcicc --uumn -Xptxas -uumn>)

    # No kernels should be parsed, unless blackwell is specified. This is a
    # build time improvement
    target_compile_definitions(${target_name}
                               PRIVATE COMPILE_BLACKWELL_TMA_GEMMS)
    target_compile_definitions(${target_name}
                               PRIVATE COMPILE_BLACKWELL_TMA_GROUPED_GEMMS)
  endif()

  if(${enable_blackwell}
     AND ("120" IN_LIST CMAKE_CUDA_ARCHITECTURES_ORIG
          OR "120-real" IN_LIST CMAKE_CUDA_ARCHITECTURES_ORIG
          OR "120-real" IN_LIST CMAKE_CUDA_ARCHITECTURES_NATIVE))

    message(STATUS "MANUALLY APPENDING FLAG TO COMPILE FOR SM_120a.")
    target_compile_options(
      ${target_name}
      PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-gencode=arch=compute_120a,code=sm_120a
              -Xcicc --uumn -Xptxas -uumn>)
  endif()

  # Suppress GCC note: the ABI for passing parameters with 64-byte alignment has
  # changed in GCC 4.6 This note appears for kernels using TMA and clutters the
  # compilation output.
  if(NOT WIN32)
    target_compile_options(
      ${target_name} PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=-Wno-psabi>)
  endif()

  target_compile_options(${target_name}
                         PRIVATE "-DENABLE_MULTI_DEVICE=${ENABLE_MULTI_DEVICE}")
  if(ENABLE_MULTI_DEVICE)
    target_link_libraries(${target_name} PRIVATE ${MPI_C_LIBRARIES})
  endif()

endfunction()

foreach(target_name
        ar_gemm_src;fpA_intB_gemm_src;fb_gemm_src;gemm_swiglu_sm90_src)
  process_target(${target_name} true false) # Name, Hopper, Blackwell
endforeach()

foreach(target_name fpA_intB_gemm_src;fb_gemm_src)
  process_target(${target_name} false true) # Name, Hopper, Blackwell
endforeach()

# foreach(target_name moe_gemm_src) process_target(${target_name} true true) #
# Name, Hopper, Blackwell endforeach()
