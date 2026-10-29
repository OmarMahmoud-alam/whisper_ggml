# The Flutter tooling requires that developers have CMake 3.10 or later
# installed. You should not increase this version, as doing so will cause
# the plugin to fail to compile for some customers of the plugin.
cmake_minimum_required(VERSION 3.10)

project(whisper_flutter_library LANGUAGES C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(W ${CMAKE_CURRENT_SOURCE_DIR}/whisper.cpp)

# Vendored whisper.cpp v1.9.1, CPU backend only. The arch/ sources are
# internally guarded per architecture, so compiling arm + x86 on every ABI
# is safe; the wrong-arch files compile to nothing.
file(GLOB GGML_SRC
    ${W}/ggml/src/*.c
    ${W}/ggml/src/*.cpp
    ${W}/ggml/src/ggml-cpu/*.c
    ${W}/ggml/src/ggml-cpu/*.cpp
    ${W}/ggml/src/ggml-cpu/arch/arm/*.c
    ${W}/ggml/src/ggml-cpu/arch/arm/*.cpp
    ${W}/ggml/src/ggml-cpu/arch/x86/*.c
    ${W}/ggml/src/ggml-cpu/arch/x86/*.cpp
    ${W}/ggml/src/ggml-cpu/amx/*.cpp
)

# One copy of whisper + the plugin per set of CPU flags: NAME is the static
# core, NAME_flutter the shared library loaded as lib<OUT>.so
function(whisper_library NAME OUT FLAGS)
    add_library(${NAME} STATIC ${GGML_SRC} ${W}/src/whisper.cpp)
    target_include_directories(${NAME} PUBLIC
        ${W}/include
        ${W}/ggml/include
        ${W}/ggml/src
        ${W}/ggml/src/ggml-cpu
        ${W}/src
    )
    target_compile_definitions(${NAME} PUBLIC GGML_USE_CPU)
    # Upstream generates these from git; we vendor a fixed release.
    target_compile_definitions(${NAME} PRIVATE
        GGML_VERSION="1.9.1"
        GGML_COMMIT="whisper.cpp-v1.9.1"
        WHISPER_VERSION="1.9.1"
    )
    target_compile_options(${NAME} PRIVATE -O3 -ffunction-sections -fdata-sections ${FLAGS})

    add_library(${NAME}_flutter SHARED main.cpp)
    set_target_properties(${NAME}_flutter PROPERTIES OUTPUT_NAME ${OUT})
    target_compile_definitions(${NAME}_flutter PUBLIC DART_SHARED_LIB)
    target_compile_options(${NAME}_flutter PRIVATE -O3 -ffunction-sections -fdata-sections ${FLAGS})
    target_link_options(${NAME}_flutter PRIVATE -Wl,--gc-sections)
    target_link_libraries(${NAME}_flutter PRIVATE ${NAME} ${CMAKE_THREAD_LIBS_INIT})
endfunction()

# No -march override: the previous armv8.2-a+fp16 flags SIGILLed on
# armv8.0 devices (issue #15). Baseline armv8-a runs everywhere.
whisper_library(whisper whisper "")

# Phones from about 2018 on (ARMv8.2 with dot product and fp16: Cortex-A55,
# A75 and newer) get a second copy built for them, which runs ggml's int8
# (q4/q8 models) and fp16 kernels several times faster. Callers load
# libwhisper_v82.so only when /proc/cpuinfo lists asimddp and asimdhp, so
# older phones keep using libwhisper.so.
if(ANDROID_ABI STREQUAL "arm64-v8a")
    whisper_library(whisper_v82 whisper_v82 "-march=armv8.2-a+dotprod+fp16")
endif()
