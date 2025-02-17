if (GGML_ALL_WARNINGS)
    if (NOT MSVC)
        add_compile_options(-Wunused -Wextra -Wcast-qual -Wdouble-promotion)
        add_compile_options("$<$<COMPILE_LANGUAGE:C>:-Wshadow;-Wno-unused-function;-Wmissing-prototypes>")
    else()
        # todo : windows
    endif()
endif()

# compiler flags

if (NOT MSVC)
    #set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-math-errno -ffinite-math-only -funsafe-math-optimizations")
endif()

message(STATUS "CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")

if (NOT UNAME_S)
    execute_process(COMMAND uname -s OUTPUT_VARIABLE UNAME_S)
endif()
if (NOT UNAME_P)
    execute_process(COMMAND uname -p OUTPUT_VARIABLE UNAME_P)
endif()
if (NOT UNAME_M)
    execute_process(COMMAND uname -m OUTPUT_VARIABLE UNAME_M)
endif()
#message(STATUS "UNAME_S: ${UNAME_S}  UNAME_P: ${UNAME_P}  UNAME_M: ${UNAME_M}")

# this version of Apple ld64 is buggy
execute_process(
    COMMAND ${CMAKE_C_COMPILER} ${CMAKE_EXE_LINKER_FLAGS} -Wl,-v
    ERROR_VARIABLE output
)
if (output MATCHES "dyld-1015\.7")
    add_compile_definitions(HAVE_BUGGY_APPLE_LINKER)
endif()

# Mac OS + Arm can report x86_64
# ref: https://github.com/ggerganov/whisper.cpp/issues/66#issuecomment-1282546789
if (UNAME_S MATCHES "Darwin")
    if (NOT UNAME_P MATCHES "arm")
        execute_process(COMMAND sysctl -n hw.optional.arm64 OUTPUT_VARIABLE SYSCTL_M)
	if (SYSCTL_M MATCHES "1")
            #set(UNAME_P "arm")
            #set(UNAME_M "arm64")
	    message(WARNING "Your arch is announced as x86_64, but it seems to actually be ARM64. Not fixing that can lead to bad performance. For more info see: https://github.com/ggerganov/whisper.cpp/issues/66\#issuecomment-#1282546789")
	endif()
    endif()
endif()

if (${CMAKE_SYSTEM_NAME} STREQUAL "Emscripten")
    message(STATUS "Emscripten detected")
elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm" OR ${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64")
    message(STATUS "ARM detected")
    #set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mcpu=apple-m1")
elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "ppc64le" OR ${CMAKE_SYSTEM_PROCESSOR} MATCHES "ppc64")
    message(STATUS "PPC64 detected")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mpower9-vector")
else()
    message(STATUS "x86 detected")
    #set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx -mavx2 -mfma -mf16c")
    if (UNAME_S MATCHES "Darwin")
        execute_process(COMMAND sysctl machdep.cpu.features OUTPUT_VARIABLE AVX1_M)
        if (AVX1_M MATCHES "AVX1.0")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx")
        endif()
        execute_process(COMMAND sysctl machdep.cpu.leaf7_features OUTPUT_VARIABLE AVX2_M)
        if (AVX2_M MATCHES "AVX2")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx2")
        endif()
        if (AVX1_M MATCHES "FMA")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfma")
        endif()
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mf16c")
    elseif (UNAME_S MATCHES "Linux")
        message(STATUS "Linux detected")
        execute_process(COMMAND grep "avx " /proc/cpuinfo OUTPUT_VARIABLE AVX1_M)
        if (AVX1_M MATCHES "avx")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx")
        endif()
        execute_process(COMMAND grep "avx2 " /proc/cpuinfo OUTPUT_VARIABLE AVX2_M)
        if (AVX2_M MATCHES "avx2")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx2")
        endif()
        execute_process(COMMAND grep "fma " /proc/cpuinfo OUTPUT_VARIABLE FMA_M)
        if (FMA_M MATCHES "fma")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfma")
        endif()
        execute_process(COMMAND grep "f16c " /proc/cpuinfo OUTPUT_VARIABLE F16C_M)
        if (F16C_M MATCHES "f16c")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mf16c")
        endif()
        execute_process(COMMAND grep "sse3 " /proc/cpuinfo OUTPUT_VARIABLE SSE3_M)
        if (SSE3_M MATCHES "sse3")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse3")
        endif()
    elseif (UNAME_S MATCHES "Haiku")
        message(STATUS "Haiku detected")
        execute_process(COMMAND sysinfo -cpu COMMAND grep "AVX " OUTPUT_VARIABLE AVX1_M)
        if (AVX1_M MATCHES "avx")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx")
        endif()
        execute_process(COMMAND sysinfo -cpu COMMAND grep "AVX2 " OUTPUT_VARIABLE AVX2_M)
        if (AVX2_M MATCHES "avx2")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx2")
        endif()
        execute_process(COMMAND sysinfo -cpu COMMAND grep "FMA " OUTPUT_VARIABLE FMA_M)
        if (FMA_M MATCHES "fma")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfma")
        endif()
        execute_process(COMMAND sysinfo -cpu COMMAND grep "F16C " OUTPUT_VARIABLE F16C_M)
        if (F16C_M MATCHES "f16c")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mf16c")
        endif()
    elseif (MSVC)
        if (GGML_AVX512)
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /arch:AVX512")
            # MSVC has no compile-time flags enabling specific
            # AVX512 extensions, neither it defines the
            # macros corresponding to the extensions.
            # Do it manually.
            if (GGML_AVX512_VBMI)
                add_compile_definitions(__AVX512VBMI__)
            endif()
            if (GGML_AVX512_VNNI)
                add_compile_definitions(__AVX512VNNI__)
            endif()
        elseif (GGML_AVX2)
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /arch:AVX2")
        elseif (GGML_AVX)
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /arch:AVX")
        endif()
    else()
        set(CMAKE_C_FLAGS  "${CMAKE_C_FLAGS} -mfma -mf16c -mavx -mavx2")
    endif()
endif()

# ggml

set(TARGET ggml)

# on APPLE - include Accelerate framework
if (APPLE AND NOT GGML_NO_ACCELERATE)
    find_library(ACCELERATE_FRAMEWORK Accelerate)
    if (ACCELERATE_FRAMEWORK)
        message(STATUS "Accelerate framework found")

        set(GGML_EXTRA_LIBS  ${GGML_EXTRA_LIBS}  ${ACCELERATE_FRAMEWORK})
        set(GGML_EXTRA_FLAGS ${GGML_EXTRA_FLAGS} -DGGML_USE_ACCELERATE -DACCELERATE_NEW_LAPACK -DACCELERATE_LAPACK_ILP64)
    else()
        message(WARNING "Accelerate framework not found")
    endif()
endif()

if (GGML_OPENBLAS)
    set(OPENBLAS_INCLUDE_SEARCH_PATHS
        /usr/include
        /usr/include/openblas
        /usr/include/openblas-base
        /usr/local/include
        /usr/local/include/openblas
        /usr/local/include/openblas-base
        /opt/OpenBLAS/include
        $ENV{OpenBLAS_HOME}
        $ENV{OpenBLAS_HOME}/include
        )
    find_path(OPENBLAS_INC NAMES cblas.h PATHS ${OPENBLAS_INCLUDE_SEARCH_PATHS})
    find_library(OPENBLAS_LIB NAMES openblas libopenblas)
    if (OPENBLAS_LIB)
        message(STATUS "OpenBLAS found")

        set(GGML_EXTRA_LIBS  ${GGML_EXTRA_LIBS}  ${OPENBLAS_LIB})
        set(GGML_EXTRA_INCS  ${GGML_EXTRA_INCS}  ${OPENBLAS_INC})
        set(GGML_EXTRA_FLAGS ${GGML_EXTRA_FLAGS} -DGGML_USE_OPENBLAS)
    else()
        message(WARNING "OpenBLAS not found")
    endif()
endif()

if (GGML_CLBLAST)
    if(NOT CLBLAST_HOME)
        set(CLBLAST_HOME
            /usr
            /usr/local
            $ENV{CLBLAST_HOME}
            )
    endif()

    find_path(CLBLAST_INC NAMES clblast.h PATHS ${CLBLAST_HOME} PATH_SUFFIXES include)
    find_library(CLBLAST_LIB NAMES clblast PATHS ${CLBLAST_HOME} PATH_SUFFIXES build)
    find_path(OPENCL_INC NAMES CL/opencl.h PATHS ${OPENCL_ROOT})
    find_library(OPENCL_LIB NAMES OpenCL)
    if (CLBLAST_LIB AND OPENCL_LIB AND CLBLAST_INC AND OPENCL_INC)
        message(STATUS "clBLAST found")

        set(GGML_EXTRA_INCS  ${GGML_EXTRA_INCS}  ${CLBLAST_INC} ${OPENCL_INC})
        set(GGML_EXTRA_LIBS  ${GGML_EXTRA_LIBS}  ${CLBLAST_LIB} ${OPENCL_LIB})
        set(GGML_EXTRA_FLAGS ${GGML_EXTRA_FLAGS} -DGGML_USE_CLBLAST)

        set(GGML_OPENCL_SOURCES ggml-opencl.cpp ggml-opencl.h)

        link_libraries("-Wl,--copy-dt-needed-entries")
    else()
        message(WARNING "clBLAST not found")
    endif()
endif()

if (GGML_CUBLAS)
    message(WARNING "GGML_CUBLAS is deprecated and will be removed in the future.\nUse GGML_CUDA instead")
    set(GGML_CUDA ON)
endif()

if (GGML_CUDA)
    cmake_minimum_required(VERSION 3.17)

    find_package(CUDAToolkit)
    if (CUDAToolkit_FOUND)
        message(STATUS "CUDA found")

        enable_language(CUDA)

        file(GLOB   GGML_CUDA_SOURCES "ggml-cuda/*.cu")
        list(APPEND GGML_CUDA_SOURCES  ggml-cuda.h)
        list(APPEND GGML_CUDA_SOURCES  ggml-cuda.cu)

        set(GGML_EXTRA_FLAGS ${GGML_EXTRA_FLAGS} -DGGML_USE_CUDA)

        if (GGML_CUDA_FORCE_DMMV)
            add_compile_definitions(GGML_CUDA_FORCE_DMMV)
        endif()
        if (GGML_CUDA_FORCE_MMQ)
            add_compile_definitions(GGML_CUDA_FORCE_MMQ)
        endif()

        # required for dynamic parallelism
        # set(CMAKE_CUDA_SEPARABLE_COMPILATION ON)

        if (GGML_STATIC)
            if (WIN32)
                # As of 12.3.1 CUDA Tookit for Windows does not offer a static cublas library
                set(GGML_EXTRA_LIBS ${GGML_EXTRA_LIBS} CUDA::cudart_static CUDA::cublas CUDA::cublasLt)
            else ()
                set(GGML_EXTRA_LIBS ${GGML_EXTRA_LIBS} CUDA::cudart_static CUDA::cublas_static CUDA::cublasLt_static)
            endif()
        else()
            set(GGML_EXTRA_LIBS ${GGML_EXTRA_LIBS} CUDA::cudart CUDA::cublas CUDA::cublasLt)
        endif()

        set(GGML_EXTRA_LIBS ${GGML_EXTRA_LIBS} CUDA::cuda_driver)

        if (CMAKE_BUILD_TYPE MATCHES Debug)
            set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -lineinfo")
        endif()
    else()
        message(WARNING "CUDA not found")
    endif()
endif()

# TODO: do not build separate ggml-rocm target (see CUDA build above, or llama.cpp for reference)
if (GGML_HIPBLAS)
    if (NOT EXISTS $ENV{ROCM_PATH})
        if (NOT EXISTS /opt/rocm)
            set(ROCM_PATH /usr)
        else()
            set(ROCM_PATH /opt/rocm)
        endif()
    else()
        set(ROCM_PATH $ENV{ROCM_PATH})
    endif()
    list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH})
    list(APPEND CMAKE_PREFIX_PATH "${ROCM_PATH}/lib64/cmake")

    # CMake on Windows doesn't support the HIP language yet
    if(WIN32)
        set(CXX_IS_HIPCC TRUE)
    else()
        string(REGEX MATCH "hipcc(\.bat)?$" CXX_IS_HIPCC "${CMAKE_CXX_COMPILER}")
    endif()

    if (CXX_IS_HIPCC)
        if(LINUX)
            if (NOT ${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
                message(WARNING "Only LLVM is supported for HIP, hint: CXX=/opt/rocm/llvm/bin/clang++")
            endif()

            message(WARNING "Setting hipcc as the C++ compiler is legacy behavior."
                    " Prefer setting the HIP compiler directly. See README for details.")
        endif()
    else()
        # Forward AMDGPU_TARGETS to CMAKE_HIP_ARCHITECTURES.
        if(AMDGPU_TARGETS AND NOT CMAKE_HIP_ARCHITECTURES)
            set(CMAKE_HIP_ARCHITECTURES ${AMDGPU_TARGETS})
        endif()
        cmake_minimum_required(VERSION 3.21)
        enable_language(HIP)
    endif()

    find_package(hip     REQUIRED)
    find_package(hipblas REQUIRED)
    find_package(rocblas REQUIRED)

    message(STATUS "HIP and hipBLAS found")

    add_compile_definitions(GGML_USE_HIPBLAS GGML_USE_CUDA)

    set(GGML_HEADERS_ROCM ggml-cuda.h)

    file(GLOB GGML_SOURCES_ROCM "ggml-cuda/*.cu")
    list(APPEND GGML_SOURCES_ROCM "ggml-cuda.cu")
    list(APPEND GGML_SOURCES_ROCM ${SRCS})

    add_compile_definitions(GGML_USE_HIPBLAS GGML_USE_CUDA)

    if (GGML_CUDA_FORCE_DMMV)
        add_compile_definitions(GGML_CUDA_FORCE_DMMV)
    endif()

    if (GGML_CUDA_FORCE_MMQ)
        add_compile_definitions(GGML_CUDA_FORCE_MMQ)
    endif()

    add_compile_definitions(GGML_CUDA_DMMV_X=${GGML_CUDA_DMMV_X})
    add_compile_definitions(GGML_CUDA_MMV_Y=${GGML_CUDA_MMV_Y})
    add_compile_definitions(K_QUANTS_PER_ITERATION=${GGML_CUDA_KQUANTS_ITER})

    add_library(ggml-rocm OBJECT ${GGML_SOURCES_ROCM} ${GGML_HEADERS_ROCM})

    if (CXX_IS_HIPCC)
        set_source_files_properties(${GGML_SOURCES_ROCM} PROPERTIES LANGUAGE CXX)
        target_link_libraries(ggml-rocm PRIVATE hip::device)
    else()
        set_source_files_properties(${GGML_SOURCES_ROCM} PROPERTIES LANGUAGE HIP)
    endif()

    target_link_libraries(ggml-rocm PRIVATE hip::device PUBLIC hip::host roc::rocblas roc::hipblas)
    target_include_directories(ggml-rocm PRIVATE . ../include ../include/ggml)

    if (BUILD_SHARED_LIBS)
        set_target_properties(ggml-rocm PROPERTIES POSITION_INDEPENDENT_CODE ON)
    endif()

    if (GGML_STATIC)
        message(FATAL_ERROR "Static linking not supported for HIP/ROCm")
    endif()
    set(GGML_EXTRA_LIBS ${GGML_EXTRA_LIBS} ggml-rocm)
endif()

if (GGML_METAL)
    find_library(FOUNDATION_LIBRARY         Foundation              REQUIRED)
    find_library(METAL_FRAMEWORK            Metal                   REQUIRED)
    find_library(METALKIT_FRAMEWORK         MetalKit                REQUIRED)
    find_library(METALPERFORMANCE_FRAMEWORK MetalPerformanceShaders REQUIRED)

    set(GGML_METAL_SOURCES ggml-metal.m ggml-metal.h)

    set(GGML_EXTRA_FLAGS ${GGML_EXTRA_FLAGS} -DGGML_USE_METAL)

    #add_compile_definitions(GGML_METAL_NDEBUG)

    # get full path to the file
    #add_compile_definitions(GGML_METAL_DIR_KERNELS="${CMAKE_CURRENT_SOURCE_DIR}/")

    # copy ggml-common.h and ggml-metal.metal to bin directory
    configure_file(ggml-common.h    ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-common.h    COPYONLY)
    configure_file(ggml-metal.metal ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.metal COPYONLY)

    if (GGML_METAL_EMBED_LIBRARY)
        enable_language(ASM)
        add_compile_definitions(GGML_METAL_EMBED_LIBRARY)

        set(METALLIB_COMMON "${CMAKE_CURRENT_SOURCE_DIR}/ggml-common.h")
        set(METALLIB_SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/ggml-metal.metal")

        file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/autogenerated")

        # merge ggml-common.h and ggml-metal.metal into a single file
        set(METALLIB_EMBED_ASM    "${CMAKE_BINARY_DIR}/autogenerated/ggml-metal-embed.s")
        set(METALLIB_SOURCE_EMBED "${CMAKE_BINARY_DIR}/autogenerated/ggml-metal-embed.metal")

        add_custom_command(
            OUTPUT ${METALLIB_EMBED_ASM}
            COMMAND echo "Embedding Metal library"
            COMMAND sed -e '/\#include \"ggml-common.h\"/r ${METALLIB_COMMON}' -e '/\#include \"ggml-common.h\"/d' < ${METALLIB_SOURCE} > ${METALLIB_SOURCE_EMBED}
            COMMAND echo ".section __DATA,__ggml_metallib"          >  ${METALLIB_EMBED_ASM}
            COMMAND echo ".globl _ggml_metallib_start"              >> ${METALLIB_EMBED_ASM}
            COMMAND echo "_ggml_metallib_start:"                    >> ${METALLIB_EMBED_ASM}
            COMMAND echo ".incbin \\\"${METALLIB_SOURCE_EMBED}\\\"" >> ${METALLIB_EMBED_ASM}
            COMMAND echo ".globl _ggml_metallib_end"                >> ${METALLIB_EMBED_ASM}
            COMMAND echo "_ggml_metallib_end:"                      >> ${METALLIB_EMBED_ASM}
            DEPENDS ggml-metal.metal ggml-common.h
            COMMENT "Generate assembly for embedded Metal library"
        )

        set(GGML_METAL_SOURCES ${GGML_METAL_SOURCES} ${METALLIB_EMBED_ASM})
    else()
        if (GGML_METAL_SHADER_DEBUG)
            # custom command to do the following:
            #   xcrun -sdk macosx metal    -fno-fast-math -c ggml-metal.metal -o ggml-metal.air
            #   xcrun -sdk macosx metallib                   ggml-metal.air   -o default.metallib
            #
            # note: this is the only way I found to disable fast-math in Metal. it's ugly, but at least it works
            #       disabling fast math is needed in order to pass tests/test-backend-ops
            # note: adding -fno-inline fixes the tests when using MTL_SHADER_VALIDATION=1
            # note: unfortunately, we have to call it default.metallib instead of ggml.metallib
            #       ref: https://github.com/ggerganov/whisper.cpp/issues/1720
            set(XC_FLAGS -fno-fast-math -fno-inline -g)
        else()
            set(XC_FLAGS -O3)
        endif()

        # Append macOS metal versioning flags
        if (GGML_METAL_MACOSX_VERSION_MIN)
            message(STATUS "Adding -mmacosx-version-min=${GGML_METAL_MACOSX_VERSION_MIN} flag to metal compilation")
            list(APPEND XC_FLAGS -mmacosx-version-min=${GGML_METAL_MACOSX_VERSION_MIN})
        endif()
        if (GGML_METAL_STD)
            message(STATUS "Adding -std=${GGML_METAL_STD} flag to metal compilation")
            list(APPEND XC_FLAGS -std=${GGML_METAL_STD})
        endif()

        add_custom_command(
            OUTPUT ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/default.metallib
            COMMAND xcrun -sdk macosx metal    ${XC_FLAGS} -c ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.metal -o ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.air
            COMMAND xcrun -sdk macosx metallib                ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.air   -o ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/default.metallib
            COMMAND rm -f ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.air
            COMMAND rm -f ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-common.h
            COMMAND rm -f ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.metal
            DEPENDS ggml-metal.metal ggml-common.h
            COMMENT "Compiling Metal kernels"
            )

        add_custom_target(
            ggml-metal ALL
            DEPENDS ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/default.metallib
            )
    endif() # GGML_METAL_EMBED_LIBRARY


    set(GGML_EXTRA_LIBS ${GGML_EXTRA_LIBS}
        ${FOUNDATION_LIBRARY}
        ${METAL_FRAMEWORK}
        ${METALKIT_FRAMEWORK}
        ${METALPERFORMANCE_FRAMEWORK}
        )
endif()

if (GGML_RPC)
    add_compile_definitions(GGML_USE_RPC)

    if (WIN32)
        set(GGML_EXTRA_LIBS ${GGML_EXTRA_LIBS} ws2_32)
    endif()

    set(GGML_RPC_SOURCES ggml-rpc.cpp)
endif()

if (GGML_VULKAN)
    find_package(Vulkan)
    if (Vulkan_FOUND)
        message(STATUS "Vulkan found")

        set(GGML_VULKAN_SOURCES ggml-vulkan.cpp ggml-vulkan.h)

        add_library(ggml-vulkan OBJECT ggml-vulkan.cpp ggml-vulkan.h)
        if (BUILD_SHARED_LIBS)
            set_target_properties(ggml-vulkan PROPERTIES POSITION_INDEPENDENT_CODE ON)
        endif()
        target_link_libraries(ggml-vulkan PRIVATE Vulkan::Vulkan)
        target_include_directories(ggml-vulkan PRIVATE . ../include ../include/ggml)

        add_compile_definitions(GGML_USE_VULKAN)

        set(GGML_EXTRA_LIBS ${GGMl_EXTRA_LIBS} ggml-vulkan)
    else()
        message(WARNING "Vulkan not found")
    endif()
endif()

if (GGML_PERF)
    set(GGML_EXTRA_FLAGS ${GGML_EXTRA_FLAGS} -DGGML_PERF)
endif()

add_library(${TARGET}
    ggml.c
    ggml-alloc.c
    ggml-backend.c
    ggml-quants.c
    ggml-impl.h
    ggml-backend-impl.h
    ../include/ggml/ggml.h
    ../include/ggml/ggml-alloc.h
    ../include/ggml/ggml-backend.h
    ${GGML_CUDA_SOURCES}
    ${GGML_OPENCL_SOURCES}
    ${GGML_METAL_SOURCES}
    ${GGML_RPC_SOURCES}
    )

target_include_directories(${TARGET} PUBLIC
    .
    ../include
    ../include/ggml
    ${GGML_EXTRA_INCS}
    )

find_library(MATH_LIBRARY m)
if (MATH_LIBRARY)
    target_link_libraries(${TARGET} PUBLIC ${MATH_LIBRARY})
endif()

target_link_libraries(${TARGET} PUBLIC ${GGML_EXTRA_LIBS} ${CMAKE_THREAD_LIBS_INIT})

if (BUILD_SHARED_LIBS)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

    target_link_libraries(${TARGET} PUBLIC
        ${CMAKE_DL_LIBS}
        )

    target_compile_definitions(${TARGET} PUBLIC
        GGML_SHARED
        )

    target_compile_definitions(${TARGET} PRIVATE
        GGML_BUILD
        )

    if (GGML_METAL)
        set_target_properties(${TARGET} PROPERTIES RESOURCE "${CMAKE_CURRENT_SOURCE_DIR}/ggml-metal.metal")
    endif()
endif()

target_compile_definitions(${TARGET} PUBLIC
    ${GGML_EXTRA_FLAGS}
    )

if (MINGW)
    target_link_libraries(${TARGET} PUBLIC
        stdc++
        )
endif()

if (GGML_CUDA_SOURCES)
    message(STATUS "GGML CUDA sources found")
    if (NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
        # Only configure gmml CUDA architectures is not globally set
        if (NOT DEFINED GGML_CUDA_ARCHITECTURES)
            # Not overriden by user, so set defaults
            set(GGML_CUDA_ARCHITECTURES 52 61 70)
        endif()
        message(STATUS "GGML Configuring CUDA architectures ${GGML_CUDA_ARCHITECTURES}")
        set_property(TARGET ggml  PROPERTY CUDA_ARCHITECTURES ${GGML_CUDA_ARCHITECTURES})
    endif()
    set_property(TARGET ggml  PROPERTY CUDA_SELECT_NVCC_ARCH_FLAGS "Auto")
    if (NOT MSVC)
        target_link_libraries(ggml PUBLIC stdc++)
    endif()
endif()

set (GGML_PUBLIC_HEADERS
     ${CMAKE_CURRENT_SOURCE_DIR}/../include/ggml/ggml.h
     ${CMAKE_CURRENT_SOURCE_DIR}/../include/ggml/ggml-alloc.h
     ${CMAKE_CURRENT_SOURCE_DIR}/../include/ggml/ggml-backend.h)

set_target_properties(${TARGET} PROPERTIES
                      PUBLIC_HEADER "${GGML_PUBLIC_HEADERS}")

install(TARGETS ${TARGET}
    LIBRARY DESTINATION lib
    PUBLIC_HEADER DESTINATION include/ggml
    )
