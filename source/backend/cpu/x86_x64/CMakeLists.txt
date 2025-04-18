# Process asm file on Windows, then subsitute *.S by *.S.obj as source file of add_library
# If MNN_ASSEMBLER env var is not set, ignore *.S file, which may cause low performance
set(EXTRA_OBJS "")
IF(MSVC AND (DEFINED ENV{MNN_ASSEMBLER}) AND "${CMAKE_SIZEOF_VOID_P}" STREQUAL "8")
    set(WIN_USE_ASM ON)
ENDIF()
message(STATUS "WIN_USE_ASM: ${WIN_USE_ASM}")
if (EMSCRIPTEN)
    set(MNN_AVX2 OFF)
endif()
if (NOT MNN_AVX2)
set(MNN_AVX512 OFF)
endif()
function (process_asm TARGET_NAME FILE_SRCS)
    if(NOT MSVC)
        return()
    endif()
    set(FILE_DESTS "")
    foreach(SRC ${${FILE_SRCS}})
        get_filename_component(SRC_EXT ${SRC} EXT)
        if(NOT ${SRC_EXT} STREQUAL ".S")
            list(APPEND FILE_DESTS ${SRC})
            continue()
        elseif(NOT WIN_USE_ASM)
            continue()
        endif()
        string(REPLACE ${CMAKE_CURRENT_SOURCE_DIR} "${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/${TARGET_NAME}.dir" DEST ${SRC})
        add_custom_command(
            OUTPUT ${DEST}.obj
            # *.S -> *.S.i: do preprocess(define/ifdef macro) by cl.exe
            COMMAND "${CMAKE_C_COMPILER}" /DWIN32 /experimental:preprocessor /P /Fi"${DEST}.i" "${SRC}"
            # *.S.i -> *.S.obj, use gnu assembler which support (AT&T syntax)
            COMMAND "$ENV{MNN_ASSEMBLER}" -o "${DEST}.obj" "${DEST}.i"
        )
        list(APPEND EXTRA_OBJS ${DEST}.obj)
    endforeach()
    set(${FILE_SRCS} ${FILE_DESTS} PARENT_SCOPE)
    set(EXTRA_OBJS ${EXTRA_OBJS} PARENT_SCOPE)
endfunction()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86_64)|(X86_64)|(x64)|(X64)|(amd64)|(AMD64)|(i686)|(x86)")
    message(STATUS "${CMAKE_SYSTEM_PROCESSOR}: Open SSE")
    target_compile_options(MNNCPU PRIVATE -DMNN_USE_SSE)
    option(MNN_AVX512_VNNI "Enable AVX512 VNNI" ON)
    FILE(GLOB MNN_X8664_SRC ${CMAKE_CURRENT_LIST_DIR}/*)
    FILE(GLOB MNN_AVX_SRC ${CMAKE_CURRENT_LIST_DIR}/avx/*)
    FILE(GLOB MNN_AVXFMA_SRC ${CMAKE_CURRENT_LIST_DIR}/avxfma/*)
    message(STATUS "MNN_AVX512:${MNN_AVX512}")
    if (MNN_AVX512 AND ((NOT MSVC) OR WIN_USE_ASM))
        FILE(GLOB MNN_AVX512_SRC ${CMAKE_CURRENT_LIST_DIR}/avx512/*)
        SET(MNNAVX512_VNNI_SRC ${CMAKE_CURRENT_LIST_DIR}/avx512/GemmInt8_VNNI.cpp)
        LIST(REMOVE_ITEM MNN_AVX512_SRC ${MNNAVX512_VNNI_SRC})
        process_asm(MNNAVX512 MNN_AVX512_SRC)
        add_library(MNNAVX512 OBJECT ${MNN_AVX512_SRC})
        target_compile_options(MNNAVX512 PRIVATE -DMNN_USE_SSE -DMNN_X86_USE_ASM)
        if (MSVC)
            target_compile_options(MNNAVX512 PRIVATE /arch:AVX512)
        else()
            target_compile_options(MNNAVX512 PRIVATE -m64 -mavx512f -mavx512dq -mavx512vl -mavx512bw -mfma)
        endif()
        if (MNN_AVX512_VNNI)
            target_compile_options(MNNAVX512 PRIVATE -DMNN_AVX512_VNNI)
            add_library(MNNAVX512_VNNI OBJECT ${MNNAVX512_VNNI_SRC})
            target_compile_options(MNNAVX512_VNNI PRIVATE -DMNN_USE_SSE)
            target_compile_options(MNNAVX512_VNNI PRIVATE -DMNN_AVX512_VNNI)
            if (MSVC)
                target_compile_options(MNNAVX512 PRIVATE /arch:AVX512)
            else()
                target_compile_options(MNNAVX512_VNNI PRIVATE -m64 -mavx512f -mavx512dq -mavx512vl -mavx512bw -mfma -mavx512vnni)
            endif()
        endif()
    endif()
    FILE(GLOB MNN_SSE_SRC ${CMAKE_CURRENT_LIST_DIR}/sse/*)
    process_asm(MNNAVX MNN_AVX_SRC)
    process_asm(MNNAVXFMA MNN_AVXFMA_SRC)
    process_asm(MNNSSE MNN_SSE_SRC)
    add_library(MNNX8664 OBJECT ${MNN_X8664_SRC})
    add_library(MNNSSE OBJECT ${MNN_SSE_SRC})
    target_compile_options(MNNX8664 PRIVATE -DMNN_USE_SSE)
    target_compile_options(MNNSSE PRIVATE -DMNN_USE_SSE)
    if (MNN_AVX2)
        target_compile_options(MNNX8664 PRIVATE -DMNN_USE_AVX)
        add_library(MNNAVX OBJECT ${MNN_AVX_SRC})
        add_library(MNNAVXFMA OBJECT ${MNN_AVXFMA_SRC})
        target_compile_options(MNNAVX PRIVATE -DMNN_USE_SSE)
        target_compile_options(MNNAVXFMA PRIVATE -DMNN_USE_SSE)
    endif()
    if(MSVC)
        if (MNN_AVX2)
            target_compile_options(MNNAVX PRIVATE /arch:AVX)
            target_compile_options(MNNAVXFMA PRIVATE /arch:AVX2)
        endif()
    else()
        target_compile_options(MNNSSE PRIVATE -msse4.1)
        if (MNN_AVX2)
            target_compile_options(MNNAVX PRIVATE -m64 -mavx2 -DMNN_X86_USE_ASM)
            target_compile_options(MNNAVXFMA PRIVATE -m64 -mavx2 -mfma -DMNN_X86_USE_ASM)
        endif()
    endif()

    if (MNN_LOW_MEMORY)
        target_compile_options(MNNX8664 PRIVATE -DMNN_LOW_MEMORY)
        target_compile_options(MNNSSE PRIVATE -DMNN_LOW_MEMORY)
        if (MNN_AVX2)
            target_compile_options(MNNAVX PRIVATE -DMNN_LOW_MEMORY)
            target_compile_options(MNNAVXFMA PRIVATE -DMNN_LOW_MEMORY)
        endif()
    endif()
    if (MNN_CPU_WEIGHT_DEQUANT_GEMM)
        target_compile_options(MNNX8664 PRIVATE -DMNN_CPU_WEIGHT_DEQUANT_GEMM)
        target_compile_options(MNNSSE PRIVATE -DMNN_CPU_WEIGHT_DEQUANT_GEMM)
        if (MNN_AVX2)
            target_compile_options(MNNAVX PRIVATE -DMNN_CPU_WEIGHT_DEQUANT_GEMM)
            target_compile_options(MNNAVXFMA PRIVATE -DMNN_CPU_WEIGHT_DEQUANT_GEMM)
        endif()
    endif()
    list(APPEND MNN_OBJECTS_TO_LINK $<TARGET_OBJECTS:MNNX8664> $<TARGET_OBJECTS:MNNSSE>)
    if (MNN_AVX2)
        list(APPEND MNN_OBJECTS_TO_LINK $<TARGET_OBJECTS:MNNAVXFMA> $<TARGET_OBJECTS:MNNAVX>)
    endif()

    if (MSVC AND WIN_USE_ASM AND MNN_AVX2)
        target_compile_options(MNNAVX PRIVATE -DMNN_X86_USE_ASM)
        target_compile_options(MNNAVXFMA PRIVATE -DMNN_X86_USE_ASM)
        list(APPEND MNN_OBJECTS_TO_LINK ${EXTRA_OBJS})
    endif()
    if (MNN_AVX512 AND ((NOT MSVC) OR WIN_USE_ASM))
        target_compile_options(MNNCPU PRIVATE -DMNN_AVX512)
        target_compile_options(MNNX8664 PRIVATE -DMNN_AVX512)
        if (MNN_AVX512_VNNI)
            target_compile_options(MNNX8664 PRIVATE -DMNN_AVX512_VNNI)
            list(APPEND MNN_OBJECTS_TO_LINK $<TARGET_OBJECTS:MNNAVX512_VNNI>)
        endif()
        list(APPEND MNN_OBJECTS_TO_LINK $<TARGET_OBJECTS:MNNAVX512>)
    endif()
endif()
