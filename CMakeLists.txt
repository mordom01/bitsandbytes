# This CMake config hopefully makes it easier to compile.
# Ensure the CUDA Toolkit is available on your path. Then run:
#   For  GCC: `cmake -B build . && cmake --build build`
#   For MSVC: `cmake -B build . && cmake --build build --config Release`
# You can also use the following options and variables
#  - COMPUTE_BACKEND: Set to `cpu`, `cuda`, or `mps` to select the backend
#  - CUDA_VERSION: The expected CUDA version, for sanity checking. The actual version
#                  is whatever CMake finds on your path.
#  - COMPUTE_CAPABILITY: Which GPU Arch/Compute codes to provide to NVCC.
#                        Separate by semicolons, i.e. `-DCOMPUTE_CAPABILITY=89;90;100;120`
#                        Check your compute capability here: https://developer.nvidia.com/cuda-gpus
#  - PTXAS_VERBOSE: Pass the `-v` option to the PTX Assembler
cmake_minimum_required(VERSION 3.22.1)

project(bitsandbytes LANGUAGES CXX)

# If run without specifying a build type, default to using the Release configuration:
#    optimizing the generated binaries for performance and also adds the `-DNDEBUG` flag,
#    which turns off a bunch of asserts which seem to link to new symbols in libstdc++,
#    worsening our many_linux compliance..
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# Define included source files
set(CPP_FILES csrc/common.cpp csrc/cpu_ops.cpp csrc/pythonInterface.cpp)
set(CUDA_FILES csrc/ops.cu csrc/kernels.cu)
set(HIP_FILES csrc/ops.hip csrc/kernels.hip)
set(MPS_FILES csrc/mps_ops.mm)
set(METAL_FILES csrc/mps_kernels.metal)
# C++ sources are always included
list(APPEND SRC_FILES ${CPP_FILES})

set(COMPUTE_BACKEND "cpu" CACHE STRING "The compute backend to use (cpu, cuda, hip, mps)")
set_property(CACHE COMPUTE_BACKEND PROPERTY STRINGS cpu cuda hip mps)
option(PTXAS_VERBOSE "Pass through -v flag to PTX Assembler" OFF)

if(APPLE)
  set(CMAKE_OSX_DEPLOYMENT_TARGET 13.1)
endif()

set(BNB_OUTPUT_NAME "bitsandbytes")

message(STATUS "Configuring ${PROJECT_NAME} (Backend: ${COMPUTE_BACKEND})")

if(${COMPUTE_BACKEND} STREQUAL "cuda")
    if(APPLE)
        message(FATAL_ERROR "CUDA is not supported on macOS" )
    endif()
    set(BUILD_CUDA ON)
    set(BUILD_HIP OFF)
    set(BUILD_MPS OFF)
elseif(${COMPUTE_BACKEND} STREQUAL "hip")
    if(APPLE)
        message(FATAL_ERROR "HIP is not supported on macOS" )
    endif()
    set(BUILD_CUDA OFF)
    set(BUILD_HIP ON)
    set(BUILD_MPS OFF)
elseif(${COMPUTE_BACKEND} STREQUAL "mps")
    if(NOT APPLE)
        message(FATAL_ERROR "MPS is only supported on macOS" )
    endif()
    set(BUILD_CUDA OFF)
    set(BUILD_HIP OFF)
    set(BUILD_MPS ON)
else()
    set(BUILD_CUDA OFF)
    set(BUILD_HIP OFF)
    set(BUILD_MPS OFF)
endif()


if(BUILD_CUDA)
    # NVCC normally will only work with MSVC up to 1939. VS2022 17.10+ starts using versions 1940+.
    # Workaround: use --allow-unsupported-compiler
    # This needs to be added *before* we try to enable the CUDA language so CMake's compiler check passes.
    if(MSVC AND MSVC_VERSION VERSION_GREATER_EQUAL 1940)
        string(APPEND CMAKE_CUDA_FLAGS " --allow-unsupported-compiler")

        # This is needed to build with VS2022 17.11+ and CUDA < 12.4.
        if (MSVC_VERSION VERSION_GREATER_EQUAL 1941)
            string(APPEND CMAKE_CUDA_FLAGS " -D_ALLOW_COMPILER_AND_STL_VERSION_MISMATCH")
        endif()
    endif()

    enable_language(CUDA) # This will fail if CUDA is not found
    find_package(CUDAToolkit REQUIRED)

    # Convert the CUDA version from X.Y.z to XY. There's probably a shorter way of doing this
    string(REGEX MATCH "^[0-9]+.[0-9]+" _CUDA_VERSION_FIRST_TWO "${CMAKE_CUDA_COMPILER_VERSION}")
    string(REPLACE "." "" CUDA_VERSION_SHORT "${_CUDA_VERSION_FIRST_TWO}")

    # Expose a cache variable that the user can set to ensure the correct version of CUDA is found
    set(CUDA_VERSION "${CUDA_VERSION_SHORT}" CACHE STRING "Expected CUDA Version Shortcode")

    message(STATUS "CUDA Version: ${CUDA_VERSION_SHORT} (${CMAKE_CUDA_COMPILER_VERSION})")
    message(STATUS "CUDA Compiler: ${CMAKE_CUDA_COMPILER}")

    # It should match the discovered version
    if(NOT CUDA_VERSION STREQUAL "${CUDA_VERSION_SHORT}")
        message(FATAL_ERROR "You've specified CUDA version ${CUDA_VERSION} however the CUDA compiler found is ${CUDA_VERSION_SHORT}."
            " Ensure the desired CUDA compiler is the first one available on your PATH."
        )
    endif()

    if(CMAKE_CUDA_COMPILER_VERSION VERSION_LESS "11.4")
        message(FATAL_ERROR "CUDA Version < 11.4 is not supported")
    elseif(CMAKE_CUDA_COMPILER_VERSION VERSION_GREATER_EQUAL "13.0")
        message(FATAL_ERROR "CUDA Version > 12 is not supported")
    endif()

    # CMake < 3.23.0 does not define CMAKE_CUDA_ARCHITECTURES_ALL.
    if(CMAKE_VERSION VERSION_LESS "3.23.0")
        message(STATUS "CMake < 3.23.0; determining CUDA architectures supported...")

        # 11.4+ supports these at a minimum.
        set(CMAKE_CUDA_ARCHITECTURES_ALL 50 52 53 60 61 62 70 72 75 80 86 87)
        set(CMAKE_CUDA_ARCHITECTURES_ALL_MAJOR 50 60 70 80)

        # CUDA 11.8 adds support for Ada and Hopper.
        if (CMAKE_CUDA_COMPILER_VERSION VERSION_GREATER_EQUAL "11.8")
            list(APPEND CMAKE_CUDA_ARCHITECTURES_ALL 89 90)
            list(APPEND CMAKE_CUDA_ARCHITECTURES_ALL_MAJOR 90)
        endif()

        # CUDA 12.8 adds support for Blackwell.
        if (CMAKE_CUDA_COMPILER_VERSION VERSION_GREATER_EQUAL "12.8")
            list(APPEND CMAKE_CUDA_ARCHITECTURES_ALL 100 101 120)
            list(APPEND CMAKE_CUDA_ARCHITECTURES_ALL_MAJOR 100 120)
        endif()
    endif()

    string(APPEND CMAKE_CUDA_FLAGS " --use_fast_math")

    if(PTXAS_VERBOSE)
        # Verbose? Outputs register usage information, and other things...
        string(APPEND CMAKE_CUDA_FLAGS " -Xptxas=-v")
    endif()

    foreach(capability ${CMAKE_CUDA_ARCHITECTURES_ALL})
        # Most of the items here are like: `xx-real`, so we just extract the `xx` portion
        string(REGEX MATCH "[0-9]+" capability_id "${capability}")
        if(capability_id GREATER 0)
            list(APPEND POSSIBLE_CAPABILITIES ${capability_id})
        endif()
    endforeach()

    # This can be changed via -D argument to CMake
    # By default all possible capabilities are compiled
    set(COMPUTE_CAPABILITY "${POSSIBLE_CAPABILITIES}" CACHE STRING "Compute Capabilities Targeted")

    message(STATUS "CUDA Capabilities Available: ${POSSIBLE_CAPABILITIES}")
    message(STATUS "CUDA Capabilities  Selected: ${COMPUTE_CAPABILITY}")

    # Use the "real" option to build native cubin for all selections.
    # Ensure we build the PTX for the latest version.
    # This behavior of adding a PTX (virtual) target for the highest architecture
    # is similar to how the "all" and "all-major" options would behave in CMake >= 3.23.
    # TODO: Consider bumping CMake requirement and using CMAKE_CUDA_ARCHITECTURES=[all | native] by default
    list(REMOVE_DUPLICATES COMPUTE_CAPABILITY)
    list(SORT COMPUTE_CAPABILITY COMPARE NATURAL)
    list(POP_BACK COMPUTE_CAPABILITY _LATEST_CAPABILITY)
    list(TRANSFORM COMPUTE_CAPABILITY APPEND "-real" OUTPUT_VARIABLE CMAKE_CUDA_ARCHITECTURES)
    list(APPEND CMAKE_CUDA_ARCHITECTURES ${_LATEST_CAPABILITY})

    message(STATUS "CUDA Targets: ${CMAKE_CUDA_ARCHITECTURES}")
    message(STATUS "CUDA NVCC Flags: ${CMAKE_CUDA_FLAGS}")

    list(APPEND SRC_FILES ${CUDA_FILES})

    string(APPEND BNB_OUTPUT_NAME "_cuda${CUDA_VERSION_SHORT}")
    add_compile_definitions(BUILD_CUDA)
elseif(BUILD_HIP)
    enable_language(HIP)
    message(STATUS "HIP Compiler: ${CMAKE_HIP_COMPILER}")
    if(DEFINED BNB_ROCM_ARCH)
      set(CMAKE_HIP_ARCHITECTURES ${BNB_ROCM_ARCH})
    else()
      if (NOT AMDGPU_TARGETS AND NOT CMAKE_HIP_ARCHITECTURES)
        set(CMAKE_HIP_ARCHITECTURES "gfx90a;gfx942;gfx1100")
      elseif (AMDGPU_TARGETS AND NOT CMAKE_HIP_ARCHITECTURES)
        set(CMAKE_HIP_ARCHITECTURES ${AMDGPU_TARGETS})
      endif()
    endif()
    message(STATUS "HIP Targets: ${CMAKE_HIP_ARCHITECTURES}")

    list(APPEND SRC_FILES ${HIP_FILES})

    string(APPEND BNB_OUTPUT_NAME "_rocm")

    # get hip version
    execute_process(COMMAND hipconfig --version OUTPUT_VARIABLE HIP_CONFIG_VERSION)
    string(REGEX MATCH "[0-9]+\\.[0-9]+" HIP_VERSION "${HIP_CONFIG_VERSION}")
    string(REPLACE "." "" HIP_VERSION_SHORT "${HIP_VERSION}")

    string(APPEND BNB_OUTPUT_NAME "${HIP_VERSION_SHORT}")
    add_compile_definitions(__HIP_PLATFORM_AMD__)
    add_compile_definitions(__HIP_PLATFORM_HCC__)
    add_compile_definitions(BUILD_HIP)
elseif(BUILD_MPS)
    if(NOT APPLE)
        message(FATAL_ERROR "MPS is only supported on macOS" )
    endif()

    enable_language(OBJCXX)

    list(APPEND SRC_FILES ${MPS_FILES})

    string(APPEND BNB_OUTPUT_NAME "_mps")
    add_compile_definitions(BUILD_MPS)
    file(MAKE_DIRECTORY "build")
    add_custom_command(OUTPUT "bitsandbytes/bitsandbytes.metallib"
                COMMAND xcrun metal -c -o "build/bitsandbytes.air" ${METAL_FILES}
                COMMAND xcrun metallib "build/bitsandbytes.air" -o "bitsandbytes/bitsandbytes.metallib"
                DEPENDS "${METAL_FILES}"
                COMMENT "Compiling Metal kernels"
                VERBATIM)
    add_custom_target(metallib DEPENDS "bitsandbytes/bitsandbytes.metallib")
else()
    string(APPEND BNB_OUTPUT_NAME "_cpu")
    set(GPU_SOURCES)
endif()

# Add this near the bottom of CMakeLists.txt, before any target creation
if(DEFINED ENV{BNB_ENABLE_INSTRUMENTATION})
    message(STATUS "🔬 Enabling BitsAndBytes kernel instrumentation")
    add_compile_definitions(BNB_ENABLE_INSTRUMENTATION)
endif()

if(WIN32)
    # Export all symbols
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

if(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:AVX2 /fp:fast")
endif()

set_source_files_properties(${CPP_FILES} PROPERTIES LANGUAGE CXX)
add_library(bitsandbytes SHARED ${SRC_FILES})
target_compile_features(bitsandbytes PUBLIC cxx_std_14)
target_include_directories(bitsandbytes PUBLIC csrc include)


if(BUILD_CUDA)
    target_include_directories(bitsandbytes PUBLIC ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES})
    target_link_libraries(bitsandbytes PUBLIC CUDA::cudart CUDA::cublas CUDA::cublasLt CUDA::cusparse)
    set_target_properties(bitsandbytes
        PROPERTIES
            CUDA_SEPARABLE_COMPILATION ON
    )
endif()
if(BUILD_HIP)
    if(NOT DEFINED ENV{ROCM_PATH})
      set(ROCM_PATH /opt/rocm)
    else()
      set(ROCM_PATH $ENV{ROCM_PATH})
    endif()
    list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH})
    macro(find_package_and_print_version PACKAGE_NAME)
      find_package("${PACKAGE_NAME}" ${ARGN})
      message("${PACKAGE_NAME} VERSION: ${${PACKAGE_NAME}_VERSION}")
    endmacro()
    find_package_and_print_version(hipblas REQUIRED)
    find_package_and_print_version(hiprand REQUIRED)
    find_package_and_print_version(hipsparse REQUIRED)

    ## hacky way of excluding hip::amdhip64 (with it linked many tests unexpectedly fail e.g. adam8bit because of inaccuracies)
    set_target_properties(hip::host PROPERTIES INTERFACE_LINK_LIBRARIES "")
    set_target_properties(hip-lang::host PROPERTIES INTERFACE_LINK_LIBRARIES "")
    set(CMAKE_HIP_IMPLICIT_LINK_LIBRARIES "")

    target_include_directories(bitsandbytes PRIVATE ${CMAKE_SOURCE_DIR} ${CMAKE_SOURCE_DIR}/include ${ROCM_PATH}/include /include)
    target_link_directories(bitsandbytes PRIVATE ${ROCM_PATH}/lib /lib)
    target_link_libraries(bitsandbytes PUBLIC roc::hipblas hip::hiprand roc::hipsparse)

    target_compile_definitions(bitsandbytes PUBLIC BNB_USE_HIP)
    set_source_files_properties(${HIP_FILES} PROPERTIES LANGUAGE HIP)
    set_target_properties(bitsandbytes PROPERTIES LINKER_LANGUAGE CXX)

    if(HIP_VERSION VERSION_LESS "6.1")
	target_compile_definitions(bitsandbytes PUBLIC NO_HIPBLASLT)
    else()
	find_package(hipblaslt)
        target_link_libraries(bitsandbytes PUBLIC roc::hipblaslt)
    endif()
endif()
if(BUILD_MPS)
    add_dependencies(bitsandbytes metallib)
    target_link_libraries(bitsandbytes objc "-framework Foundation" "-framework Metal" "-framework MetalPerformanceShaders" "-framework MetalPerformanceShadersGraph")
endif()

if(WIN32)
    set_target_properties(bitsandbytes PROPERTIES PREFIX "lib")
endif()
set_target_properties(bitsandbytes PROPERTIES OUTPUT_NAME ${BNB_OUTPUT_NAME})
if(MSVC)
    set_target_properties(bitsandbytes PROPERTIES LIBRARY_OUTPUT_DIRECTORY_RELEASE "${PROJECT_SOURCE_DIR}/bitsandbytes")
    set_target_properties(bitsandbytes PROPERTIES LIBRARY_OUTPUT_DIRECTORY_DEBUG "${PROJECT_SOURCE_DIR}/bitsandbytes")
    set_target_properties(bitsandbytes PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELEASE "${PROJECT_SOURCE_DIR}/bitsandbytes")
    set_target_properties(bitsandbytes PROPERTIES RUNTIME_OUTPUT_DIRECTORY_DEBUG "${PROJECT_SOURCE_DIR}/bitsandbytes")
endif()

set_target_properties(bitsandbytes PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bitsandbytes")
