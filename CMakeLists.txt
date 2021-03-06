cmake_minimum_required (VERSION 2.8.3)

project("NanoGUI-SDL")

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(CMAKE_CXX_FLAGS "-std=c++11")
endif()

if(NOT IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/ext/eigen/Eigen")
  message(FATAL_ERROR "The NanoGUI-SDL dependency repositories (Eigen, etc.) are missing! "
    "You probably did not clone the project with --recursive. It is possible to recover "
    "by calling \"git submodule update --init --recursive\"")
endif()

option(NANOGUISDL_BUILD_EXAMPLE "Build NanoGUI example application?" ON)
option(NANOGUISDL_BUILD_SHARED  "Build NanoGUI as a shared library?" ON)

if(NANOGUI_GL_IMPLEMENTATION MATCHES "gl2" OR NOT DEFINED NANOGUI_GL_IMPLEMENTATION)
    add_definitions(-DNANOVG_GL2_IMPLEMENTATION)
elseif(NANOGUI_GL_IMPLEMENTATION MATCHES "gl3")
    add_definitions(-DNANOVG_GL3_IMPLEMENTATION)
elseif(NANOGUI_GL_IMPLEMENTATION MATCHES "gles2")
    add_definitions(-DNANOVG_GLES2_IMPLEMENTATION)
elseif(NANOGUI_GL_IMPLEMENTATION MATCHES "gles3")
    add_definitions(-DNANOVG_GLES3_IMPLEMENTATION)
else()
    message(FATAL_ERROR "Unknown GL implementaion. Use one of gl2, gl3, gles2, gles3.")
endif()

# Required libraries for linking against nanogui (all targets)
set(NANOGUISDL_EXTRA_LIBS "")

# Platform-dependent files for libnanogui
set(LIBNANOGUISDL_EXTRA_SOURCE "")

if(APPLE AND NANOGUISDL_BUILD_SHARED)
  set(CMAKE_MACOSX_RPATH ON)
endif()

include(CheckCXXCompilerFlag)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()
string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)

set(BUILD_SHARED_LIBS ${NANOGUI_BUILD_SHARED} CACHE BOOL " " FORCE)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  # Quench annoying deprecation warnings when compiling GLFW on OSX
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-deprecated-declarations")
endif()

# Shared library mode: add NANOGUI_SHARED flag to all targets
if (NANOGUISDL_BUILD_SHARED)
  add_definitions ("-DNANOGUI_SHARED")
endif()

if (MSVC)
  # Disable annoying MSVC warnings (all targets)
  add_definitions(/D "_CRT_SECURE_NO_WARNINGS")

  # Parallel build on MSVC (all targets)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")

  if (NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:SSE2")

    # Disable Eigen vectorization for Windows 32 bit builds (issues with unaligned access segfaults)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /DEIGEN_DONT_ALIGN")
  endif()
endif()

# Compile with compiler warnings turned on
if(MSVC)
  if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
    string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
  endif()
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  # Enable C++11 mode on GCC / Clang
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  add_definitions( -DEIGEN_DONT_ALIGN )
endif()

# Required core libraries on various platforms
if (WIN32)
  list(APPEND NANOGUISDL_EXTRA_LIBS opengl32)
elseif (APPLE)
  find_library(cocoa_library Cocoa)
  find_library(opengl_library OpenGL)
  find_library(corevideo_library CoreVideo)
  find_library(iokit_library IOKit)
  list(APPEND NANOGUISDL_EXTRA_LIBS ${cocoa_library} ${opengl_library} ${corevideo_library} ${iokit_library})
  list(APPEND LIBNANOGUISDL_EXTRA_SOURCE src/darwin.mm)
elseif(CMAKE_SYSTEM MATCHES "Linux")
  list(APPEND NANOGUISDL_EXTRA_LIBS GL Xxf86vm Xrandr Xinerama Xcursor Xi X11 pthread dl rt SDL2)
  add_definitions(-DNANOGUI_LINUX)
endif()

include_directories(ext ext/eigen include ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR})

# Run simple C converter to put font files into the data segment
add_executable(bin2c resources/bin2c.c)
set(bin2c_cmdline nanogui_resources.cpp nanogui_resources.h)
file(GLOB resources "${CMAKE_CURRENT_SOURCE_DIR}/resources/*.ttf")
foreach(file ${resources})
  list(APPEND bin2c_cmdline ${file})
endforeach()

# Run bin2c on resource files
add_custom_command(
  OUTPUT nanogui_resources.cpp nanogui_resources.h
  COMMAND bin2c ARGS ${bin2c_cmdline}
  DEPENDS bin2c ${resources}
  COMMENT "Running bin2c"
  PRE_BUILD VERBATIM)

# Needed to generated files
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# Set library type
if(NANOGUISDL_BUILD_SHARED)
  set(NANOGUISDL_LIBRARY_TYPE "SHARED")
else()
  set(NANOGUISDL_LIBRARY_TYPE "STATIC")
endif()

# Compile main NanoGUI library
add_library(nanogui ${NANOGUISDL_LIBRARY_TYPE}
  # Merge NanoVG into the NanoGUI library
  ext/nanovg/nanovg.c
  # Merge GLEW into the NanoGUI library (only if needed)
  ${LIBNANOGUISDL_EXTRA_SOURCE}
  # Fonts etc.
  nanogui_resources.cpp
  src/serializer.cpp
  include/nanogui/common.h src/common.cpp
  include/nanogui/widget.h src/widget.cpp
  include/nanogui/theme.h src/theme.cpp
  include/nanogui/layout.h src/layout.cpp
  include/nanogui/screen.h src/screen.cpp
  include/nanogui/label.h src/label.cpp
  include/nanogui/window.h src/window.cpp
  include/nanogui/popup.h src/popup.cpp
  include/nanogui/checkbox.h src/checkbox.cpp
  include/nanogui/button.h src/button.cpp
  include/nanogui/popupbutton.h src/popupbutton.cpp
  include/nanogui/combobox.h src/combobox.cpp
  include/nanogui/progressbar.h src/progressbar.cpp
  include/nanogui/slider.h src/slider.cpp
  include/nanogui/messagedialog.h src/messagedialog.cpp
  include/nanogui/textbox.h src/textbox.cpp
  include/nanogui/imagepanel.h src/imagepanel.cpp
  include/nanogui/imageview.h src/imageview.cpp
  include/nanogui/vscrollpanel.h src/vscrollpanel.cpp
  include/nanogui/colorwheel.h src/colorwheel.cpp
  include/nanogui/colorpicker.h src/colorpicker.cpp
  include/nanogui/graph.h src/graph.cpp
  include/nanogui/glutil.h src/glutil.cpp
  include/nanogui/tabheader.h src/tabheader.cpp
  include/nanogui/tabwidget.h src/tabwidget.cpp
  include/nanogui/stackedwidget.h src/stackedwidget.cpp
  include/nanogui/formhelper.h
  include/nanogui/toolbutton.h
  include/nanogui/opengl.h
  include/nanogui/nanogui.h
)

# Compile/link flags for NanoGUI
set_property(TARGET nanogui APPEND PROPERTY COMPILE_DEFINITIONS "NANOGUI_BUILD ")

if (NANOGUISDL_BUILD_SHARED)
  target_link_libraries(nanogui ${NANOGUISDL_EXTRA_LIBS})
endif()

if (NOT ${U_CMAKE_BUILD_TYPE} MATCHES DEBUG AND NANOGUI_BUILD_SHARED)
  # Link-time code generation (only for shared library)
  if (MSVC)
    set_property(TARGET nanogui APPEND_STRING PROPERTY LINK_FLAGS "/LTCG ")
    set_property(TARGET nanogui APPEND_STRING PROPERTY COMPILE_FLAGS "/GL ")
  elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set_property(TARGET nanogui APPEND_STRING PROPERTY COMPILE_FLAGS "-fvisibility=hidden ")

    # Check for Link Time Optimization support
    CHECK_CXX_COMPILER_FLAG("-flto" HAS_LTO_FLAG)
    if (HAS_LTO_FLAG)
      set_property(TARGET nanogui APPEND_STRING PROPERTY COMPILE_FLAGS "-flto ")
    endif()
  endif()
endif()

if (NANOGUISDL_BUILD_SHARED AND NOT ${U_CMAKE_BUILD_TYPE} MATCHES DEBUG)
  if (APPLE)
    # Strip .dylib library on OSX
    add_custom_command(TARGET nanogui POST_BUILD COMMAND strip -u -r ${CMAKE_CURRENT_BINARY_DIR}/libnanogui.dylib)
  elseif(UNIX)
    # Strip .so library on Linux
    add_custom_command(TARGET nanogui POST_BUILD COMMAND strip ${CMAKE_CURRENT_BINARY_DIR}/libnanogui.so)
  endif()
endif()

# Quench warnings while compiling NanoVG
if (CMAKE_COMPILER_IS_GNUCC)
  set_source_files_properties(ext/nanovg/src/nanovg.c PROPERTIES COMPILE_FLAGS -Wno-unused-result)
elseif(MSVC)
  set_source_files_properties(ext/nanovg/src/nanovg.c PROPERTIES COMPILE_FLAGS "/wd4005 /wd4456 /wd4457")
endif()

if (NANOGUISDL_BUILD_SHARED)
  set_source_files_properties(ext/nanovg/src/nanovg.c PROPERTIES COMPILE_DEFINITIONS "NVG_BUILD;NVG_SHARED")
else()
  set_source_files_properties(ext/nanovg/src/nanovg.c PROPERTIES COMPILE_DEFINITIONS "NVG_BUILD")
endif()


# Build example application if desired
if(NANOGUISDL_BUILD_EXAMPLE)
  add_executable(example1 src/example1.cpp)
  add_executable(example2 src/example2.cpp)
  target_link_libraries(example1 nanogui ${NANOGUI_EXTRA_LIBS})
  target_link_libraries(example2 nanogui ${NANOGUI_EXTRA_LIBS})

  # Copy icons for example application
  file(COPY resources/icons DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
endif()
