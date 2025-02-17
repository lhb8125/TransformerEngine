# Copyright (c) 2022-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
#
# See LICENSE for license information.

# Configure Transformer Engine library
set(transformer_engine_SOURCES)
list(APPEND transformer_engine_SOURCES
     transformer_engine.cpp
     transpose/cast_transpose.cu
     transpose/transpose.cu
     transpose/cast_transpose_fusion.cu
     transpose/transpose_fusion.cu
     transpose/multi_cast_transpose.cu
     activation/gelu.cu
     fused_attn/fused_attn_f16_max512_seqlen.cu
     fused_attn/fused_attn_f16_arbitrary_seqlen.cu
     activation/relu.cu
     activation/swiglu.cu
     fused_attn/fused_attn_fp8.cu
     fused_attn/fused_attn.cpp
     fused_attn/utils.cu
     gemm/cublaslt_gemm.cu
     layer_norm/ln_api.cpp
     layer_norm/ln_bwd_semi_cuda_kernel.cu
     layer_norm/ln_fwd_cuda_kernel.cu
     rmsnorm/rmsnorm_api.cpp
     rmsnorm/rmsnorm_bwd_semi_cuda_kernel.cu
     rmsnorm/rmsnorm_fwd_cuda_kernel.cu
     util/cast.cu
     util/cuda_driver.cpp
     util/cuda_runtime.cpp
     util/rtc.cpp
     util/system.cpp
     fused_softmax/scaled_masked_softmax.cu
     fused_softmax/scaled_upper_triang_masked_softmax.cu
     fused_softmax/scaled_masked_softmax.cu
     fused_softmax/scaled_upper_triang_masked_softmax.cu)
add_library(transformer_engine SHARED ${transformer_engine_SOURCES})
target_include_directories(transformer_engine PUBLIC
                           "${CMAKE_CURRENT_SOURCE_DIR}/include")

# Check for cuDNN frontend API
set(CUDNN_FRONTEND_INCLUDE_DIR
    "${CMAKE_SOURCE_DIR}/../3rdparty/cudnn-frontend/include")
if(NOT EXISTS "${CUDNN_FRONTEND_INCLUDE_DIR}")
    message(FATAL_ERROR
            "Could not find cuDNN frontend API. "
            "Try running 'git submodule update --init --recursive' "
            "within the Transformer Engine source.")
endif()

# Configure dependencies
target_link_libraries(transformer_engine PUBLIC
                      CUDA::cublas
                      CUDA::cuda_driver
                      CUDA::cudart
                      CUDA::nvrtc
                      CUDA::nvToolsExt
                      CUDNN::cudnn)
target_include_directories(transformer_engine PRIVATE
                           ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES})
target_include_directories(transformer_engine PRIVATE "${CUDNN_FRONTEND_INCLUDE_DIR}")

# Make header files with C++ strings
function(make_string_header STRING STRING_NAME)
    configure_file(util/string_header.h.in
                   "string_headers/${STRING_NAME}.h"
                   @ONLY)
endfunction()
function(make_string_header_from_file file_ STRING_NAME)
    file(READ "${file_}" STRING)
    configure_file(util/string_header.h.in
                   "string_headers/${STRING_NAME}.h"
                   @ONLY)
endfunction()
list(GET CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES 0 cuda_include_path)
make_string_header("${cuda_include_path}"
                   string_path_cuda_include)
make_string_header_from_file(utils.cuh
                             string_code_utils_cuh)
make_string_header_from_file(transpose/rtc/transpose.cu
                             string_code_transpose_rtc_transpose_cu)
target_include_directories(transformer_engine PRIVATE
                           "${CMAKE_CURRENT_BINARY_DIR}/string_headers")

# Compiler options
set_source_files_properties(fused_softmax/scaled_masked_softmax.cu
                            fused_softmax/scaled_upper_triang_masked_softmax.cu
                            PROPERTIES
                            COMPILE_OPTIONS "--use_fast_math")
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --expt-relaxed-constexpr")
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -O3")

# Install library
install(TARGETS transformer_engine DESTINATION .)
