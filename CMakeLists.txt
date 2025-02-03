cmake_minimum_required(VERSION 3.17.2)
# This file is used to build and install a customized version of FFmpeg for ILLIXR
set(CMAKE_VERBOSE_MAKEFILE True)

project(ILLIXR-ffmpeg)

set(PACKAGE_STRING "ILLIXR-ffmpeg 6.1")
set(PACKAGE_VERSION_MAJOR "6")
set(PACKAGE_VERSION_MINOR "1")
set(PACKAGE_VERSION_PATCH "0")
set(PACKAGE_VERSION_LABEL "")

if(PACKAGE_VERSION_LABEL)
    set(PACKAGE_VERSION "${PACKAGE_VERSION_MAJOR}.${PACKAGE_VERSION_MINOR}.${PACKAGE_VERSION_PATCH}-${PACKAGE_VERSION_LABEL}")
else()
    set(PACKAGE_VERSION "${PACKAGE_VERSION_MAJOR}.${PACKAGE_VERSION_MINOR}.${PACKAGE_VERSION_PATCH}")
endif()

project(ILLIXR_ffmpeg VERSION 6.1)  # mirror the FFmpeg version this is based on
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
find_package(PkgConfig REQUIRED)
option(BUILD_PARALLEL_LEVEL "Number of parallel build jobs")
if(NOT BUILD_PARALLEL_LEVEL)
    include(ProcessorCount)
    ProcessorCount(N)
    if(N GREATER 3)
        math(EXPR N "${N} - 2")
        set(BUILD_PARALLEL_LEVEL ${N})
    else()
        set(BUILD_PARALLEL_LEVEL 1)
    endif()
endif()
# add any custom module paths for finding cmake and pkg-config files
set(CMAKE_MODULE_PATH "${CMAKE_INSTALL_PREFIX}/lib/cmake ${CMAKE_INSTALL_PREFIX}/share/cmake ${CMAKE_MODULE_PATH}")
set(ENV{PKG_CONFIG_PATH} "${CMAKE_INSTALL_PREFIX}/lib/pkgconfig:${CMAKE_INSTALL_PREFIX}/share/pkgconfig")

# function to download, build, and install ffnvcodec 11.1.5 if needed
function (build_ext_project url tag)
    set(PRJ_BUILD_DIR ${CMAKE_BINARY_DIR}/_deps/ffnvcodec)
    FILE(MAKE_DIRECTORY ${PRJ_BUILD_DIR} ${PRJ_BUILD_DIR}/build)

    set(CMAKE_PATCH "--- src/Makefile.orig	2023-11-21 10:12:20.529663643 -0600
+++ src/Makefile	2023-11-21 10:13:14.640609797 -0600
@@ -1,4 +1,4 @@
-PREFIX = /usr/local
+PREFIX = ${CMAKE_INSTALL_PREFIX}
 LIBDIR = lib
 INSTALL = install
 SED = sed
")
    FILE(WRITE ${PRJ_BUILD_DIR}/Makefile.patch "${CMAKE_PATCH}")

    set(DO_PATCH "#!/usr/bin/env sh

while getopts d:p: flag
do
  case \"\${flag}\" in
    d) wkdir=\${OPTARG};;
    p) patchfile=\${OPTARG};;
    *) print_error;;
  esac
done

if [ -z \${patchfile+x} ]
  then
    echo \"No patch file given, use -p <patchfile>\"
    exit 1;
fi

patch -p0 -N -d $wkdir --input=\"\$patchfile\"
retCode=\$?
if [ \$retCode -gt 1 ]
 then
   exit \$retCode
fi
exit 0
")

    FILE(WRITE ${PRJ_BUILD_DIR}/do_patch.sh "${DO_PATCH}")
    FILE(CHMOD ${PRJ_BUILD_DIR}/do_patch.sh PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE)
    set(CMAKE_EXT_BUILD "cmake_minimum_required(VERSION 3.17.2)
project(ffnvcodec)
include(FetchContent)
FetchContent_Declare(ffnvcodec
    GIT_REPOSITORY ${url}
    GIT_TAG ${tag}
    PATCH_COMMAND ${PRJ_BUILD_DIR}/do_patch.sh -p ${PRJ_BUILD_DIR}/Makefile.patch -d ${PRJ_BUILD_DIR}
    SOURCE_DIR ${PRJ_BUILD_DIR}/src
    )
FetchContent_MakeAvailable(ffnvcodec)
")
    FILE(WRITE ${PRJ_BUILD_DIR}/CMakeLists.txt "${CMAKE_EXT_BUILD}")

    execute_process(COMMAND ${CMAKE_COMMAND} ..
                    WORKING_DIRECTORY ${PRJ_BUILD_DIR}/build
                    RESULT_VARIABLE PROC_RES)
    if (NOT ${PROC_RES} EQUAL 0)
        message(FATAL_ERROR "Configuring ffnvcodec failed.")
    endif()

    execute_process(COMMAND make install
                    WORKING_DIRECTORY ${PRJ_BUILD_DIR}/src
                    RESULT_VARIABLE PROC_RES)
    if (NOT ${PROC_RES} EQUAL 0)
        message(FATAL_ERROR "Building ffnvcodec failed.")
    endif()
endfunction()

find_package(Vulkan 1.3.255)

if (NOT Vulkan_FOUND)
    message(STATUS "Checking for Vulkan with pkg-config")
    pkg_check_modules(Vulkan REQUIRED vulkan>=1.3.255)
endif()


pkg_check_modules(ffnvcodec ffnvcodec>=11.1.5)
if (NOT ffnvcodec_FOUND)
    message(STATUS "Building ffnvcodec 11.1.5")
    build_ext_project(https://github.com/FFmpeg/nv-codec-headers.git
                      43d91706e097565f57b311e567f0219838bcc2f6
                      )
endif()

pkg_check_modules(fontconfig REQUIRED fontconfig)
pkg_check_modules(gnutls REQUIRED gnutls)
pkg_check_modules(aom REQUIRED aom>=1.0.0)
pkg_check_modules(libass REQUIRED libass>=0.11.0)
pkg_check_modules(libbluray REQUIRED libbluray)
pkg_check_modules(libbs2b REQUIRED libbs2b)
pkg_check_modules(libdrm REQUIRED libdrm)
pkg_check_modules(freetype2 REQUIRED freetype2)
pkg_check_modules(fribidi REQUIRED fribidi)
pkg_check_modules(libmodplug REQUIRED libmodplug)
pkg_check_modules(libopenjp2 REQUIRED libopenjp2>=2.1.0)
pkg_check_modules(libopenmpt REQUIRED libopenmpt>=0.2.6557)
pkg_check_modules(opus REQUIRED opus)
pkg_check_modules(libpulse REQUIRED libpulse)
pkg_check_modules(soxr REQUIRED soxr)
pkg_check_modules(speex REQUIRED speex)
pkg_check_modules(libssh REQUIRED libssh>=0.6.0)
pkg_check_modules(theora REQUIRED theora)
pkg_check_modules(libv4l2 REQUIRED libv4l2)
pkg_check_modules(vorbis REQUIRED vorbis)
pkg_check_modules(vorbisenc REQUIRED vorbisenc)
pkg_check_modules(vpx REQUIRED vpx>=1.4.0)
pkg_check_modules(libwebp REQUIRED libwebp>=0.2.0)
pkg_check_modules(xcb REQUIRED xcb>=1.4)
pkg_check_modules(libxml-2.0 REQUIRED libxml-2.0)
pkg_check_modules(OpenCL REQUIRED OpenCL)
pkg_check_modules(opengl REQUIRED opengl)

find_library(mp3lame_FOUND mp3lame)
if (NOT mp3lame_FOUND)
    message(FATAL_ERROR "libmp3lame-dev appears to be missing")
endif()

# have to loop over versions because of the way cuda_rt libraries are named
foreach (VMAJOR RANGE 11 12)
    foreach (VMINOR RANGE 9)
        set(VER_STR "cudart-${VMAJOR}.${VMINOR}")
        pkg_check_modules(cuda-rt QUIET ${VER_STR})
        if (cuda-rt_FOUND)
            break()
        endif()
    endforeach (VMINOR)
    if (cuda-rt_FOUND)
        break()
    endif()
endforeach (VMAJOR)
if (NOT cuda-rt_FOUND)
    message(FATAL_ERROR "Could not find cuda.h")
endif()

find_program(yasm_FOUND yasm)
if (NOT yasm_FOUND)
    find_program(yasm_FOUND nasm)
    if (NOT yasm_FOUND)
        message(FATAL_ERROR "Could not find yasm/nasm on the system.")
    endif()
endif()

# have to manually run configure
execute_process(COMMAND ./configure --disable-static --enable-shared --disable-stripping --disable-doc --enable-pic --enable-cuda-llvm --enable-lto --enable-fontconfig --enable-gnutls --enable-libaom --enable-libass --enable-libbluray --enable-libbs2b --enable-libdrm --enable-libfreetype --enable-libfribidi --enable-libmodplug --enable-libmp3lame --enable-libopenjpeg --enable-libopenmpt --enable-libopus --enable-libpulse --enable-libsoxr --enable-libspeex --enable-libssh --enable-libtheora --enable-libv4l2 --enable-libvorbis --enable-libvpx --enable-libwebp --enable-libxcb --enable-libxml2 --enable-nvdec --enable-nvenc --enable-opencl --enable-opengl --enable-vulkan --disable-programs --build-suffix=_illixr --prefix=${CMAKE_INSTALL_PREFIX}
                WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                RESULT_VARIABLE PROC_RES
                )

if (NOT ${PROC_RES} EQUAL 0)
    message(FATAL_ERROR "Configure failed")
endif()

# add build target, with both build and install
add_custom_target(FFMPEG
                  ALL
                  COMMAND make install -j${BUILD_PARALLEL_LEVEL}
                  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

# custom clean command since ffmpeg does not use cmake
add_custom_target(do_clean
                  COMMAND make clean
                  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

# custom distclean command since ffmpeg does not use cmake
add_custom_target(do_distclean
                  COMMAND make distclean
                  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
