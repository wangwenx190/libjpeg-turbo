cmake_minimum_required(VERSION 3.25)
project(JpegTurbo VERSION 3.2.2 LANGUAGES C)
include(GNUInstallDirs)
include(cmake/Sources.cmake)

option(ENABLE_STATIC "Build the static library" ON)
option(ENABLE_SHARED "Build the shared library" ON)
option(JPEG_INSTALL "Install JPEG headers and libraries" ON)

set(VERSION ${PROJECT_VERSION})
set(BUILD ${PROJECT_VERSION})
set(PACKAGE_NAME "libjpeg-turbo")
set(JPEG_LIB_VERSION 62)
set(LIBJPEG_TURBO_VERSION_NUMBER 3002002)
set(BITS_IN_JSAMPLE 8)
set(WITH_SIMD OFF)
set(SIMD_ARCHITECTURE 0)
set(jpeg_arch "${CMAKE_SYSTEM_PROCESSOR}")
if(CMAKE_C_COMPILER_ARCHITECTURE_ID)
    set(jpeg_arch "${CMAKE_C_COMPILER_ARCHITECTURE_ID}")
endif()
if(APPLE AND CMAKE_OSX_ARCHITECTURES)
    set(jpeg_arch "${CMAKE_OSX_ARCHITECTURES}")
endif()
string(TOLOWER "${jpeg_arch}" jpeg_arch)
if(jpeg_arch MATCHES "^(arm64|aarch64)$")
    set(WITH_SIMD ON)
    set(SIMD_ARCHITECTURE ARM64)
    configure_file(simd/arm/neon-compat.h.in neon-compat.h @ONLY)
    list(APPEND codec_sources
        simd/arm/aarch64/jchuff-neon.c
        simd/arm/aarch64/jsimdcpu.c
        simd/arm/jccolor-neon.c
        simd/arm/jcgray-neon.c
        simd/arm/jcphuff-neon.c
        simd/arm/jcsample-neon.c
        simd/arm/jdcolor-neon.c
        simd/arm/jdmerge-neon.c
        simd/arm/jdsample-neon.c
        simd/arm/jfdctfst-neon.c
        simd/arm/jfdctint-neon.c
        simd/arm/jidctfst-neon.c
        simd/arm/jidctint-neon.c
        simd/arm/jidctred-neon.c
        simd/arm/jquanti-neon.c
        simd/jsimd.c)
elseif(jpeg_arch MATCHES "^(x86_64|amd64|x64|i[3-6]86|x86)$")
    set(WITH_SIMD ON)
    enable_language(ASM_NASM)
    include(cmake/X86Sources.cmake)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(SIMD_ARCHITECTURE X86_64)
        set(jpeg_asm_dir x86_64)
        set(jpeg_nasm_definitions __x86_64__)
        if(WIN32)
            list(APPEND jpeg_nasm_definitions WIN64)
        endif()
    else()
        set(SIMD_ARCHITECTURE I386)
        set(jpeg_asm_dir i386)
        if(WIN32)
            set(jpeg_nasm_definitions WIN32)
        endif()
    endif()
    if(APPLE)
        list(APPEND jpeg_nasm_definitions MACHO PIC)
    elseif(UNIX)
        list(APPEND jpeg_nasm_definitions ELF PIC)
    endif()
    set(jpeg_asm_sources ${jpeg_${jpeg_asm_dir}_sources})
    file(GLOB jpeg_asm_includes CONFIGURE_DEPENDS simd/nasm/*.inc "simd/${jpeg_asm_dir}/*ext*.asm")
    set_source_files_properties(${jpeg_asm_sources} PROPERTIES
        COMPILE_DEFINITIONS "${jpeg_nasm_definitions}" OBJECT_DEPENDS "${jpeg_asm_includes}")
    list(APPEND codec_sources simd/jsimd.c ${jpeg_asm_sources})
endif()
set(C_ARITH_CODING_SUPPORTED ON)
set(D_ARITH_CODING_SUPPORTED ON)
set(SIZE_T ${CMAKE_SIZEOF_VOID_P})
if(MSVC)
    set(INLINE "__forceinline")
    set(THREAD_LOCAL "__declspec(thread)")
    set(HAVE_INTRIN_H ON)
else()
    set(HIDDEN "__attribute__((visibility(\"hidden\")))")
    set(INLINE "__inline__ __attribute__((always_inline))")
    set(THREAD_LOCAL "__thread")
    if(NOT WIN32)
        set(HAVE_BUILTIN_CTZL ON)
    endif()
endif()
foreach(header IN ITEMS jconfig jconfigint jversion)
    configure_file("src/${header}.h.in" "${header}.h" @ONLY)
endforeach()
file(CONFIGURE OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/NOTICE" CONTENT
    "This software is based in part on the work of the Independent JPEG Group.
")

function(add_jpeg target type)
    add_library(${target} ${type} ${codec_sources} ${ARGN})
    target_include_directories(${target} PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>"
        "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>" PRIVATE simd simd/arm)
    if(jpeg_asm_dir)
        target_compile_options(${target} PRIVATE
            "$<$<COMPILE_LANGUAGE:ASM_NASM>:-I${CMAKE_CURRENT_SOURCE_DIR}/simd/nasm/>"
            "$<$<COMPILE_LANGUAGE:ASM_NASM>:-I${CMAKE_CURRENT_SOURCE_DIR}/simd/${jpeg_asm_dir}/>"
        )
    endif()
    set_target_properties(${target} PROPERTIES POSITION_INDEPENDENT_CODE ON)
    if(JPEG_INSTALL)
        install(TARGETS ${target})
    endif()
endfunction()
if(ENABLE_STATIC)
    add_jpeg(jpeg-static STATIC)
    if(NOT WIN32)
        set_target_properties(jpeg-static PROPERTIES OUTPUT_NAME jpeg)
    endif()
endif()
if(ENABLE_SHARED)
    add_jpeg(jpeg SHARED)
    set_target_properties(jpeg PROPERTIES VERSION 62.4.0 SOVERSION 62
        WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()
if(PROJECT_IS_TOP_LEVEL)
    set(turbojpeg_sources src/turbojpeg.c src/transupp.c src/jdatadst-tj.c src/jdatasrc-tj.c
        src/rdbmp.c src/wrbmp.c
        src/wrapper/rdpng-8.c src/wrapper/rdpng-12.c src/wrapper/rdpng-16.c
        src/wrapper/wrpng-8.c src/wrapper/wrpng-12.c src/wrapper/wrpng-16.c
        src/wrapper/rdppm-8.c src/wrapper/rdppm-12.c src/wrapper/rdppm-16.c
        src/wrapper/wrppm-8.c src/wrapper/wrppm-12.c src/wrapper/wrppm-16.c
        src/spng/spng.c src/spng/zlib/adler32.c src/spng/zlib/compress.c
        src/spng/zlib/crc32.c src/spng/zlib/deflate.c src/spng/zlib/inffast.c
        src/spng/zlib/inflate.c src/spng/zlib/inftrees.c src/spng/zlib/trees.c src/spng/zlib/zutil.c)

    function(add_turbojpeg target type)
        add_jpeg(${target} ${type} ${turbojpeg_sources})
        target_include_directories(${target} PRIVATE src/spng src/spng/zlib)
        target_compile_definitions(${target} PRIVATE BMP_SUPPORTED PNG_SUPPORTED SPNG_STATIC PPM_SUPPORTED)
        if(NOT WIN32)
            set_target_properties(${target} PROPERTIES OUTPUT_NAME turbojpeg)
        endif()
    endfunction()
    if(ENABLE_STATIC)
        add_turbojpeg(turbojpeg-static STATIC)
    endif()
    if(ENABLE_SHARED)
        add_turbojpeg(turbojpeg SHARED)
        set_target_properties(turbojpeg PROPERTIES VERSION 0.5.0 SOVERSION 0 DEFINE_SYMBOL DLLDEFINE)
        if(UNIX AND NOT APPLE)
            target_link_options(turbojpeg PRIVATE
                "LINKER:--version-script,${CMAKE_CURRENT_SOURCE_DIR}/src/turbojpeg-mapfile")
        endif()
    endif()
    if(JPEG_INSTALL)
        install(FILES src/turbojpeg.h DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
        install(FILES src/spng/LICENSE src/spng/libpng-LICENSE.txt
            DESTINATION "${CMAKE_INSTALL_DATADIR}/licenses/libjpeg-turbo/spng")
        install(FILES src/spng/zlib/LICENSE
            DESTINATION "${CMAKE_INSTALL_DATADIR}/licenses/libjpeg-turbo/spng/zlib")
    endif()
endif()

if(JPEG_INSTALL)
    install(FILES src/jpeglib.h src/jerror.h src/jmorecfg.h "${CMAKE_CURRENT_BINARY_DIR}/jconfig.h"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
    install(FILES LICENSE.md README.ijg "${CMAKE_CURRENT_BINARY_DIR}/NOTICE"
        DESTINATION "${CMAKE_INSTALL_DATADIR}/licenses/libjpeg-turbo")
endif()

if(WIN32 AND TARGET jpeg)
    enable_language(RC)
    configure_file(cmake/LibraryVersion.rc.in LibraryVersion.rc @ONLY)
    target_sources(jpeg PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/LibraryVersion.rc")
    if(TARGET turbojpeg)
        target_sources(turbojpeg PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/LibraryVersion.rc")
    endif()
endif()
