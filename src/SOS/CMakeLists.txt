if(CLR_CMAKE_PLATFORM_UNIX)
    include_directories(${ROOT_DIR}/src/pal/inc)
    include_directories(${ROOT_DIR}/src/pal/inc/rt)

    add_subdirectory(lldbplugin)
endif(CLR_CMAKE_PLATFORM_UNIX)

# lldbplugin doesn't build with these options
if(WIN32)
  message(STATUS "CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION: ${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION}")
  include_directories("$ENV{ProgramFiles(x86)}/Windows Kits/10/Include/${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION}")

  message(STATUS "VSInstallDir: $ENV{VSInstallDir}")
  include_directories("$ENV{VSInstallDir}/DIA SDK/include")

  add_compile_options(/FIWarningControl.h) # force include of WarningControl.h
  add_compile_options(/Zl) # omit default library name in .OBJ

  add_subdirectory(runcommand)
  if(NOT CLR_CMAKE_TARGET_ARCH_ARM64)
    add_subdirectory(SOS.UnitTests/Debuggees/DesktopClrHost)
  endif()
endif(WIN32)

add_definitions(-D_SECURE_SCL=0)

add_subdirectory(extensions)
add_subdirectory(dbgutil)
add_subdirectory(SOS.Extensions)
add_subdirectory(Strike)

if(CLR_CMAKE_PLATFORM_UNIX)
  add_compile_options(-fPIC)

  # Include the dummy c++ include files
  include_directories(${ROOT_DIR}/src/pal/inc/rt/cpp)

  # This prevents inclusion of standard C compiler headers
  add_compile_options(-nostdinc)
endif(CLR_CMAKE_PLATFORM_UNIX)
