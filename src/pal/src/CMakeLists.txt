cmake_minimum_required(VERSION 2.8.12.2)

project(coreclrpal)

include_directories(SYSTEM /usr/local/include)

add_compile_options(-fPIC)

include(configure.cmake)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

if(CORECLR_SET_RPATH)
    # Enable @rpath support for shared libraries.
    set(MACOSX_RPATH ON)
endif(CORECLR_SET_RPATH)

if(CMAKE_VERSION VERSION_EQUAL 3.0 OR CMAKE_VERSION VERSION_GREATER 3.0)
    cmake_policy(SET CMP0042 NEW)
endif()

# Include directories

include_directories(include)

# Compile options

if(CLR_CROSS_COMPONENTS_BUILD)
    if(CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL x86_64 OR CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL amd64)
        if(CLR_CMAKE_TARGET_ARCH STREQUAL "arm")
            set(PAL_CMAKE_PLATFORM_ARCH_I386 1)
            add_definitions(-D_X86_)
        else()
            set(PAL_CMAKE_PLATFORM_ARCH_AMD64 1)
            add_definitions(-D_AMD64_)
        endif()
    elseif(CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL i686)
        set(PAL_CMAKE_PLATFORM_ARCH_I386 1)
        add_definitions(-D_X86_)
    else()
        message(FATAL_ERROR "Only AMD64, I386 host for cross-architecture component is supported")
    endif()
else()
    if(CLR_CMAKE_PLATFORM_ARCH_AMD64)
        set(PAL_CMAKE_PLATFORM_ARCH_AMD64 1)
    elseif(CLR_CMAKE_PLATFORM_ARCH_ARM)
        set(PAL_CMAKE_PLATFORM_ARCH_ARM 1)
    elseif(CLR_CMAKE_PLATFORM_ARCH_ARM64)
        set(PAL_CMAKE_PLATFORM_ARCH_ARM64 1)
    elseif(CLR_CMAKE_PLATFORM_ARCH_I386)
        set(PAL_CMAKE_PLATFORM_ARCH_I386 1)
    elseif(CLR_CMAKE_PLATFORM_ARCH_MIPS64)
        set(PAL_CMAKE_PLATFORM_ARCH_MIPS64 1)
    else()
        message(FATAL_ERROR "Only ARM, AMD64, ARM64, MIPS64 and I386 is supported")
    endif()
endif()

add_definitions(-DPLATFORM_UNIX=1)
add_definitions(-DLP64COMPATIBLE=1)
add_definitions(-DFEATURE_PAL=1)
add_definitions(-DCORECLR=1)
add_definitions(-DPIC=1)
add_definitions(-D_FILE_OFFSET_BITS=64)
if(PAL_CMAKE_PLATFORM_ARCH_AMD64)
  add_definitions(-DBIT64=1)
  add_definitions(-D_WIN64=1)
  set(PAL_ARCH_SOURCES_DIR amd64)
elseif(PAL_CMAKE_PLATFORM_ARCH_ARM)
  add_definitions(-DBIT32=1)
  set(PAL_ARCH_SOURCES_DIR arm)
elseif(PAL_CMAKE_PLATFORM_ARCH_ARM64)
  add_definitions(-DBIT64=1)
  add_definitions(-D_WIN64=1)
  set(PAL_ARCH_SOURCES_DIR arm64)
elseif(PAL_CMAKE_PLATFORM_ARCH_I386)
  add_definitions(-DBIT32=1)
  set(PAL_ARCH_SOURCES_DIR i386)
elseif(PAL_CMAKE_PLATFORM_ARCH_MIPS64)
  add_definitions(-DBIT64=1)
  add_definitions(-D_WIN64=1)
  set(PAL_ARCH_SOURCES_DIR mips64)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL Darwin)
  add_definitions(-D_TARGET_MAC64)
  add_definitions(-DXSTATE_SUPPORTED)
  set(PLATFORM_SOURCES
    arch/${PAL_ARCH_SOURCES_DIR}/context.S
  )
endif(CMAKE_SYSTEM_NAME STREQUAL Darwin)

if(PAL_CMAKE_PLATFORM_ARCH_AMD64 AND CMAKE_SYSTEM_NAME STREQUAL Linux AND NOT CLR_CMAKE_PLATFORM_ALPINE_LINUX)
  # Currently the _xstate is not available on Alpine Linux
  add_definitions(-DXSTATE_SUPPORTED)
endif(PAL_CMAKE_PLATFORM_ARCH_AMD64 AND CMAKE_SYSTEM_NAME STREQUAL Linux AND NOT CLR_CMAKE_PLATFORM_ALPINE_LINUX)

if(CLR_CMAKE_PLATFORM_ALPINE_LINUX)
  # Setting RLIMIT_NOFILE breaks debugging of coreclr on Alpine Linux for some reason
  add_definitions(-DDONT_SET_RLIMIT_NOFILE)
  # On Alpine Linux, we need to ensure that the reported stack range for the primary thread is
  # larger than the initial committed stack size.
  add_definitions(-DENSURE_PRIMARY_STACK_SIZE)
endif(CLR_CMAKE_PLATFORM_ALPINE_LINUX)

# turn off capability to remove unused functions (which was enabled in debug build with sanitizers)
set(CMAKE_SHARED_LINKER_FLAGS_DEBUG "${CMAKE_SHARED_LINKER_FLAGS_DEBUG} -Wl,--no-gc-sections")

set(ARCH_SOURCES
  arch/${PAL_ARCH_SOURCES_DIR}/debugbreak.S
  arch/${PAL_ARCH_SOURCES_DIR}/processor.cpp
)

set(SOURCES
  cruntime/file.cpp
  cruntime/filecrt.cpp
  cruntime/lstr.cpp
  cruntime/malloc.cpp
  cruntime/math.cpp
  cruntime/mbstring.cpp
  cruntime/misc.cpp
  cruntime/misctls.cpp
  cruntime/path.cpp
  cruntime/printf.cpp
  cruntime/printfcpp.cpp
  cruntime/silent_printf.cpp
  cruntime/string.cpp
  cruntime/stringtls.cpp
  cruntime/wchar.cpp
  cruntime/wchartls.cpp
  debug/debug.cpp
  file/directory.cpp
  file/file.cpp
  file/filetime.cpp
  file/find.cpp
  file/path.cpp
  handlemgr/handleapi.cpp
  handlemgr/handlemgr.cpp
  init/pal.cpp
  loader/module.cpp
  loader/modulename.cpp
  locale/unicode.cpp
  locale/unicode_data.cpp
  locale/utf8.cpp
  map/common.cpp
  map/map.cpp
  map/virtual.cpp
  memory/heap.cpp
  memory/local.cpp
  misc/cgroup.cpp
  misc/dbgmsg.cpp
  misc/environ.cpp
  misc/error.cpp
  misc/errorstrings.cpp
  misc/fmtmessage.cpp
  misc/miscpalapi.cpp
  misc/msgbox.cpp
  misc/strutil.cpp
  misc/sysinfo.cpp
  misc/time.cpp
  misc/utils.cpp
  objmgr/palobjbase.cpp
  objmgr/shmobject.cpp
  objmgr/shmobjectmanager.cpp
  safecrt/makepath_s.cpp
  safecrt/memcpy_s.cpp
  safecrt/memmove_s.cpp
  safecrt/mbusafecrt.cpp
  safecrt/safecrt_input_s.cpp
  safecrt/safecrt_output_l.cpp
  safecrt/safecrt_output_s.cpp
  safecrt/safecrt_winput_s.cpp
  safecrt/safecrt_woutput_s.cpp
  safecrt/splitpath_s.cpp
  safecrt/sprintf_s.cpp
  safecrt/sscanf_s.cpp
  safecrt/strcat_s.cpp
  safecrt/strcpy_s.cpp
  safecrt/strlen_s.cpp
  safecrt/strncat_s.cpp
  safecrt/strncpy_s.cpp
  safecrt/strtok_s.cpp
  safecrt/swprintf.cpp
  safecrt/vsprintf.cpp
  safecrt/vswprint.cpp
  safecrt/wcscat_s.cpp
  safecrt/wcscpy_s.cpp
  safecrt/wcslen_s.cpp
  safecrt/wcsncat_s.cpp
  safecrt/wcsncpy_s.cpp
  safecrt/wcstok_s.cpp
  safecrt/wmakepath_s.cpp
  safecrt/wsplitpath_s.cpp
  safecrt/xtoa_s.cpp
  safecrt/xtow_s.cpp
  shmemory/shmemory.cpp
  sync/cs.cpp
  thread/thread.cpp
)

_add_library(coreclrpal
  STATIC
  ${SOURCES}
  ${ARCH_SOURCES}
  ${PLATFORM_SOURCES}
)

if(CMAKE_SYSTEM_NAME STREQUAL Darwin)
  find_library(COREFOUNDATION CoreFoundation)
  find_library(CORESERVICES CoreServices)
  find_library(SECURITY Security)
  find_library(SYSTEM System)
  target_link_libraries(coreclrpal
    ${COREFOUNDATION}
    ${CORESERVICES}
    ${SECURITY}
    ${SYSTEM}
  )
endif(CMAKE_SYSTEM_NAME STREQUAL Darwin)

if(CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
  target_link_libraries(coreclrpal
    pthread
    rt
  )
endif(CMAKE_SYSTEM_NAME STREQUAL FreeBSD)

if(CMAKE_SYSTEM_NAME STREQUAL Linux)
  # On Android, we don't need to link with gcc_s, pthread and rt
  if(NOT CLR_CMAKE_PLATFORM_ANDROID)
    target_link_libraries(coreclrpal
      gcc_s
      pthread
      rt
    )
  endif()

  if(CLR_CMAKE_PLATFORM_ANDROID)
    find_library(LZMA NAMES lzma)

    if(LZMA STREQUAL LZMA-NOTFOUND)
      message(FATAL_ERROR "Cannot find liblzma.")
    endif(LZMA STREQUAL LZMA-NOTFOUND)

    target_link_libraries(coreclrpal
      gnustl_shared
      android-support
      android-glob
      ${LZMA})
  endif()

  if(CLR_MAKE_PLATFORM_ANDROID)
    find_library(ANDROID_SUPPORT NAMES android-support)
    find_library(ANDROID_GLOB NAMES android-glob)

    if(ANDROID_SUPPORT STREQUAL ANDROID_SUPPORT-NOTFOUND)
      message(FATAL_ERROR "Cannot find android-support.")
    endif()

    if(ANDROID_GLOB STREQUAL ANDROID_GLOB-NOTFOUND)
      message(FATAL_ERROR "Cannot find android-glob.")
    endif()
  endif()

  target_link_libraries(coreclrpal
    dl
  )
endif(CMAKE_SYSTEM_NAME STREQUAL Linux)

if(CMAKE_SYSTEM_NAME STREQUAL NetBSD)
  add_definitions(-D_KMEMUSER)
  find_library(KVM kvm)
  target_link_libraries(coreclrpal
    pthread
    rt
    ${KVM}
  )
endif(CMAKE_SYSTEM_NAME STREQUAL NetBSD)
