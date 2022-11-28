cmake_minimum_required(VERSION 3.16...3.21)

# Change obs-plugintemplate to your plugin's name in a machine-readable format (e.g.:
# obs-myawesomeplugin) and set
project(obs-hyper-plugin VERSION 0.0.1)
add_library(${CMAKE_PROJECT_NAME} MODULE)

# Replace `Your Name Here` with the name (yours or your organization's) you want to see as the
# author of the plugin (in the plugin's metadata itself and in the installers)
set(PLUGIN_AUTHOR "Hyper Online, Inc.")

# Replace `com.example.obs-plugin-template` with a unique Bundle ID for macOS releases (used both in
# the installer and when submitting the installer for notarization)
set(MACOS_BUNDLEID "com.hyperonline.${CMAKE_PROJECT_NAME}")

# Replace `me@contoso.com` with the maintainer email address you want to put in Linux packages
set(LINUX_MAINTAINER_EMAIL "hello@hyper.online")

# Add your custom source files here - header files are optional and only required for visibility
# e.g. in Xcode or Visual Studio target_sources(${CMAKE_PROJECT_NAME} PRIVATE src/plugin-main.c)

# Import libobs as main plugin dependency
find_package(libobs REQUIRED)
include(cmake/ObsPluginHelpers.cmake)

# Uncomment these lines if you want to use the OBS Frontend API in your plugin
find_package(obs-frontend-api REQUIRED)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::obs-frontend-api)

# Uncomment those lines if you want to use Qt in your plugin
find_qt(COMPONENTS Widgets Core)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE Qt::Core Qt::Widgets)
set_target_properties(
  ${CMAKE_PROJECT_NAME}
  PROPERTIES AUTOMOC ON
             AUTOUIC ON
             AUTORCC ON)

# ============================================
# ===== Plugin specific settings (Hyper) =====
# ============================================

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_CXX_STANDARD 17)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "../obs-studio/cmake/Modules/")
find_package(FFmpeg REQUIRED COMPONENTS avcodec avutil)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(ARCH 64)
else()
  set(ARCH 32)
endif()

if(WIN32)
  find_library(WS2_32_LIBRARY ws2_32)

  # Provide dirent path hint for the libimobiledevice CMakeLists
  set(DIRENT_DIR "deps/dirent/")
  cmake_path(ABSOLUTE_PATH DIRENT_DIR OUTPUT_VARIABLE DIRENT_DIR)
endif()

# ---- libcnary

set(libcnary_HEADERS deps/libplist/libcnary/include/node_list.h
                     deps/libplist/libcnary/include/node.h deps/libplist/libcnary/include/object.h)

set(libcnary_SOURCES deps/libplist/libcnary/cnary.c deps/libplist/libcnary/node_list.c
                     deps/libplist/libcnary/node.c)

include_directories(libcnary deps/libplist/libcnary/include)

add_library(libcnary STATIC ${libcnary_SOURCES} ${libcnary_HEADERS})

# ------- libplist

set(libplist_HEADERS
    deps/libplist/src/base64.h
    deps/libplist/src/bytearray.h
    deps/libplist/src/hashtable.h
    deps/libplist/src/plist.h
    deps/libplist/src/ptrarray.h
    deps/libplist/src/strbuf.h
    deps/libplist/src/time64_limits.h
    deps/libplist/src/time64.h
    deps/libplist/include/plist/Array.h
    deps/libplist/include/plist/Boolean.h
    deps/libplist/include/plist/Data.h
    deps/libplist/include/plist/Date.h
    deps/libplist/include/plist/Dictionary.h
    deps/libplist/include/plist/Integer.h
    deps/libplist/include/plist/Key.h
    deps/libplist/include/plist/Node.h
    deps/libplist/include/plist/plist.h
    deps/libplist/include/plist/plist++.h
    deps/libplist/include/plist/Real.h
    deps/libplist/include/plist/String.h
    deps/libplist/include/plist/Structure.h
    deps/libplist/include/plist/Uid.h)

include_directories(libplist deps/libplist/include deps/libplist/src deps/libplist/libcnary/include)

set(libplist_SOURCES
    deps/libplist/src/Array.cpp
    deps/libplist/src/base64.c
    deps/libplist/src/Boolean.cpp
    deps/libplist/src/bplist.c
    deps/libplist/src/bytearray.c
    deps/libplist/src/Data.cpp
    deps/libplist/src/Date.cpp
    deps/libplist/src/Dictionary.cpp
    deps/libplist/src/hashtable.c
    deps/libplist/src/Integer.cpp
    deps/libplist/src/Key.cpp
    deps/libplist/src/Node.cpp
    deps/libplist/src/plist.c
    deps/libplist/src/ptrarray.c
    deps/libplist/src/Real.cpp
    deps/libplist/src/String.cpp
    deps/libplist/src/Structure.cpp
    deps/libplist/src/time64.c
    deps/libplist/src/Uid.cpp
    deps/libplist/src/xplist.c)

add_library(libplist STATIC ${libplist_SOURCES} ${libplist_HEADERS})

target_link_libraries(libplist libcnary)

# ------- libusbmuxd

set(libusbmuxd_HEADERS deps/libusbmuxd/common/collection.h deps/libusbmuxd/common/socket.h
                       deps/libusbmuxd/common/thread.h)

set(libusbmuxd_SOURCES deps/libusbmuxd/common/collection.c deps/libusbmuxd/common/socket.c
                       deps/libusbmuxd/common/thread.c deps/libusbmuxd/src/libusbmuxd.c)

include_directories(libusbmuxd include deps/libusbmuxd/common deps/libusbmuxd/include)

add_library(libusbmuxd STATIC ${libusbmuxd_SOURCES} ${libusbmuxd_HEADERS})

# Cmake version of configure.ac
include(CheckIncludeFiles)
include(CheckFunctionExists)

check_include_files(sys/inotify.h HAVE_INOTIFY_H)
if(${HAVE_INOTIFY_H})
  add_compile_definitions(libusbmuxd HAVE_INOTIFY)
endif()

check_function_exists(stpncpy HAVE_STPNCPY)
if(${HAVE_STPNCPY})
  add_compile_definitions(libusbmuxd HAVE_STPNCPY)
endif()

add_compile_definitions(libusbmuxd PACKAGE_STRING="obs-hyper-plugin")

# Ignore multiple definitions of thread_once from libusbmuxd and libplist
if(MSVC)
  target_link_options(libusbmuxd PUBLIC /FORCE:MULTIPLE)
endif()

target_link_libraries(libusbmuxd libplist)

if(WIN32)
  target_link_libraries(libusbmuxd wsock32 ws2_32)
endif()

# -------- Portal Lib

set(portal_HEADERS
    deps/portal/src/Channel.hpp deps/portal/src/Device.hpp deps/portal/src/Protocol.hpp
    deps/portal/src/logging.h deps/portal/src/DeviceConnection.hpp
    deps/portal/src/DeviceManager.hpp)

set(portal_SOURCES
    deps/portal/src/Channel.cpp deps/portal/src/Device.cpp deps/portal/src/Protocol.cpp
    deps/portal/src/DeviceConnection.cpp deps/portal/src/DeviceManager.cpp)

include_directories(portal include deps/portal/src/)

add_library(portal STATIC ${portal_SOURCES} ${portal_HEADERS})

add_subdirectory(deps/libimobiledevice)

target_link_libraries(portal libusbmuxd libimobiledevice)

# --

set(ENABLE_PROGRAMS false)

# set(obs-hyper-plugin_SOURCES
target_sources(
  ${CMAKE_PROJECT_NAME}
  PRIVATE src/obs-hyper-plugin.cpp
          src/ffmpeg-decode.c
          src/VideoDecoder.cpp
          src/FFMpegVideoDecoder.cpp
          src/FFMpegAudioDecoder.cpp
          src/Thread.cpp
          src/DeviceApplicationConnectionController.cpp
          src/plugin-main.c)

# set(obs-hyper-plugin_HEADERS
target_sources(
  ${CMAKE_PROJECT_NAME}
  PRIVATE src/obs-hyper-plugin.h
          src/obs-hyper-plugin2.h
          src/ffmpeg-decode.h
          src/VideoDecoder.h
          src/FFMpegVideoDecoder.h
          src/FFMpegAudioDecoder.h
          src/Thread.hpp
          src/Queue.hpp
          src/DeviceApplicationConnectionController.hpp)

if(APPLE)

  message("Linking Video Toolbox")

  find_library(AVFOUNDATION AVFoundation)
  find_library(COCOA Cocoa ONLY)
  find_library(COREFOUNDATION CoreFoundation)
  find_library(COREMEDIA CoreMedia)
  find_library(COREVIDEO CoreVideo)
  find_library(COREMEDIAIO CoreMediaIO)
  find_library(VIDEOTOOLBOX VideoToolbox)

  include_directories(
    ${AVFOUNDATION}
    ${COCOA}
    ${COREFOUNDATION}
    ${COREMEDIA}
    ${COREVIDEO}
    ${COREMEDIAIO}
    ${VIDEOTOOLBOX})

  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE ${COCOA} ${COREMEDIA} ${COREVIDEO}
                                                      ${COREMEDIAIO} ${VIDEOTOOLBOX})

  # list(APPEND obs-hyper-plugin_SOURCES
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE src/VideoToolboxVideoDecoder.cpp)

  # list(APPEND obs-hyper-plugin_HEADERS
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE src/VideoToolboxVideoDecoder.h)

  add_definitions(-fobjc-arc)

endif()

# --- Platform-independent build settings ---
# add_library(${CMAKE_PROJECT_NAME} MODULE ${obs-hyper-plugin_SOURCES} ${obs-hyper-plugin_HEADERS})

# find_package(FFmpeg REQUIRED COMPONENTS avcodec avutil)

include_directories(${FFMPEG_INCLUDE_DIRS})

if(MSVC)
  # https://stackoverflow.com/q/33290879 needed for some MSVC installations
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SAFESEH:NO")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /SAFESEH:NO")
  set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} /SAFESEH:NO")

endif()

target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE portal ${FFMPEG_LIBRARIES})

# ==================================
# ===== End of section (Hyper) =====
# ==================================

configure_file(src/plugin-macros.h.in ${CMAKE_SOURCE_DIR}/src/plugin-macros.generated.h)

target_sources(${CMAKE_PROJECT_NAME} PRIVATE src/plugin-macros.generated.h)

# /!\ TAKE NOTE: No need to edit things past this point /!\

# --- Platform-independent build settings ---

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${CMAKE_SOURCE_DIR}/src)

target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::libobs)

# --- End of section ---

# --- Windows-specific build settings and tasks ---
if(OS_WINDOWS)
  configure_file(cmake/bundle/windows/installer-Windows.iss.in
                 ${CMAKE_BINARY_DIR}/installer-Windows.generated.iss)

  configure_file(cmake/bundle/windows/resource.rc.in ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.rc)
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.rc)

  if(MSVC)
    target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE /W4)
  endif()
  # --- End of section ---

  # -- macOS specific build settings and tasks --
elseif(OS_MACOS)
  configure_file(cmake/bundle/macos/installer-macos.pkgproj.in
                 ${CMAKE_BINARY_DIR}/installer-macos.generated.pkgproj)

  set(MACOSX_PLUGIN_GUI_IDENTIFIER "${MACOS_BUNDLEID}")
  set(MACOSX_PLUGIN_BUNDLE_VERSION "${CMAKE_PROJECT_VERSION}")
  set(MACOSX_PLUGIN_SHORT_VERSION_STRING "1")

  target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE -Wall)
  # --- End of section ---

  # --- Linux-specific build settings and tasks ---
else()
  target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE -Wall)
endif()
# --- End of section ---

setup_plugin_target(${CMAKE_PROJECT_NAME})
