# If not stated otherwise in this file or this component's LICENSE file the
# following copyright and licenses apply:
#
# Copyright 2020 Sky UK
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Sets the minimum version of CMake required to build the native library.
cmake_minimum_required( VERSION 3.7.0 )

# Include GNUInstallDirs module to access CMAKE_INSTALL_* directory variables
include(GNUInstallDirs)

# Project setup
project( Dobby VERSION "3.0" )


# Set the major and minor version numbers of dobby (also used by plugins)
set( DOBBY_MAJOR_VERSION 3 )
set( DOBBY_MINOR_VERSION 0 )

set(INSTALL_CMAKE_DIR lib/cmake/Dobby)

# Android Studio seems to mix between different case for the build type,
# so convert it to all uppercase and use that from now on
string( TOUPPER "${CMAKE_BUILD_TYPE}" BUILD_TYPE )


# Set the 'AI' build type defines for removing some of the code on release builds
add_definitions( -DAI_DEBUG=1 -DAI_RELEASE=2 )
if( BUILD_TYPE STREQUAL "RELEASE" )
    add_definitions( -DAI_BUILD_TYPE=AI_RELEASE )
elseif( BUILD_TYPE STREQUAL "DEBUG" )
    add_definitions( -DAI_BUILD_TYPE=AI_DEBUG )
else()
    message( FATAL_ERROR "BUILD_TYPE '${BUILD_TYPE}' not supported" )
endif()


# Add definition to determine RDK build
add_definitions( -DRDK )
add_definitions( -DXI1=1 -DLLAMA=2 -DXI6=6 )

# Add definition to differentiate between Dobby builds and external projects building DobbyProxy
add_definitions( -DDOBBY_BUILD )

# If no platform defined then default to XI6
set( RDK_PLATFORM "XI6" CACHE STRING "The target RDK platform, options are \"XI1\", \"XI6\", \"LLAMA\" or \"DEV_VM\"" )
string( TOUPPER "${RDK_PLATFORM}" RDK_PLATFORM )

# If developer VM defined, configure platform to be Ubuntu VM machine, else configure set-top box
if( RDK_PLATFORM STREQUAL "XI6" )
    add_definitions( -DRDK_PLATFORM=XI6 )
elseif( RDK_PLATFORM STREQUAL "XI1" )
    add_definitions( -DRDK_PLATFORM=XI1 )
elseif( RDK_PLATFORM STREQUAL "LLAMA" )
    add_definitions( -DRDK_PLATFORM=LLAMA )
elseif( RDK_PLATFORM STREQUAL "DEV_VM" )
    add_definitions( -DDEV_VM=9 )
    add_definitions( -DRDK_PLATFORM=DEV_VM )
else()
    message( FATAL_ERROR "Unknown RDK_PLATFORM '${RDK_PLATFORM}'" )
endif()

# Option to enable / disable the Perfetto tracing support
# Can only be enabled in DEBUG builds
set( ENABLE_PERFETTO_TRACING_DEFAULT OFF )
option( ENABLE_PERFETTO_TRACING "Enable Perfetto tracing" ${ENABLE_PERFETTO_TRACING_DEFAULT} )

if( BUILD_TYPE STREQUAL "RELEASE" AND ENABLE_PERFETTO_TRACING )
    message( WARNING "Cannot enable tracing on release builds ")
    set( ENABLE_PERFETTO_TRACING OFF )
endif()

# Add defines for DBUS path
add_definitions( -DDBUS_SYSTEM_ADDRESS="unix:path=/var/run/dbus/system_bus_socket" )

option( LEGACY_COMPONENTS "Enable Dobby legacy components" ON )
# Add #define if enabled
if ( LEGACY_COMPONENTS )
    add_definitions( -DLEGACY_COMPONENTS )
endif()

# Disable for now. LTO can cause issues with debugging symbols on GCC versions before 8.0.0.
# See http://hubicka.blogspot.com/2018/06/gcc-8-link-time-and-interprocedural.html
option( ENABLE_LTO "Enable link time optimisation compiler flags" OFF )

# Set default Dobby dbus service to org.rdk.dobby and add defines
set( DOBBY_SERVICE "org.rdk.dobby" CACHE STRING "Dobby dbus service name")
if(NOT DOBBY_SERVICE STREQUAL "org.rdk.dobby")
    add_definitions( -DDOBBY_SERVICE_OVERRIDE="${DOBBY_SERVICE}")
endif()

# Set default Dobby dbus object path to /org/rdk/dobby and add defines
set( DOBBY_OBJECT "/org/rdk/dobby" CACHE STRING "Dobby dbus object path")
if(NOT DOBBY_OBJECT STREQUAL "/org/rdk/dobby")
    add_definitions( -DDOBBY_OBJECT_OVERRIDE="${DOBBY_OBJECT}")
endif()

# Enable C++14 support. The following tells cmake to always add the -std=c++14
# flag (nb - if CMAKE_CXX_EXTENSIONS is ON then we'll get -std=gnu++14 instead)
set( CMAKE_CXX_STANDARD          14  )
set( CMAKE_CXX_STANDARD_REQUIRED ON  )
set( CMAKE_CXX_EXTENSIONS        OFF )


# Enable GNU extensions on linux builds
if( UNIX AND NOT APPLE )
    add_definitions( -D_GNU_SOURCE )
endif()


# Thread is just the generic lib link for the pthread libraries (on platforms
# that don't have a separate pthread library this is a NOP)
find_package( Threads REQUIRED )


# The following disables an annoying warning "<blah> will change in GCC X.XX"
add_compile_options( -Wno-psabi )


# Add our local cmake directory to search for components
set( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/cmake" )

# Find the 3rd party libraries and header files
find_package( libnl REQUIRED )
find_package( dbus REQUIRED )
find_package( jsoncpp REQUIRED )
if ( LEGACY_COMPONENTS )
    find_package( ctemplate REQUIRED )
endif()
find_package( Boost REQUIRED )

# Required for libocispec only
find_package( yajl REQUIRED )

# Set #define if tracing is enabled
if( ENABLE_PERFETTO_TRACING )
    find_package(PerfettoSdk REQUIRED)
    add_definitions( -DAI_ENABLE_TRACING=1 )

    # For tracing, the trace categories need to be supplied in a separate header
    # from the main trace code, the following tells cmake where to get that header
    set( TRACING_CATEGORIES_HEADER "${CMAKE_CURRENT_LIST_DIR}/tracing/include/DobbyTraceCategories.h" )
endif()

# Run libocispec to generate OCI config parsers and necessary C headers
include(cmake/libocispec.cmake)


# Tracing
if( ENABLE_PERFETTO_TRACING )
    add_subdirectory(tracing)
endif()

# Enable link time optimization if it's supported
if( ENABLE_LTO )
    if( ${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.9.0" )
        # use cmake's IPO setting to apply LTO flags to compiler
        cmake_policy( SET CMP0069 NEW )
        include( CheckIPOSupported )
        check_ipo_supported( RESULT ipo_supported OUTPUT error  )
        if( ipo_supported )
            set( CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE )
        else()
            message( WARNING "IPO flags not known for the current compiler" )
        endif()
    else()
        # apply LTO manually with gcc flags
        if( ${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU" AND ${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER_EQUAL "4.5.0" )
            add_compile_options( -flto -ffat-lto-objects )
        else()
            message( WARNING "gcc 4.5.0 or higher required for LTO flags" )
        endif()
    endif()
endif()

# Enable the start container hook (disabled by default)
option(USE_STARTCONTAINER_HOOK "Enable the startContainer OCI hook in RDK plugins" OFF)
if(USE_STARTCONTAINER_HOOK)
    add_definitions( -DUSE_STARTCONTAINER_HOOK )
endif()

# Import AppInfrastructure code
add_subdirectory( AppInfrastructure/Logging )
add_subdirectory( AppInfrastructure/Tracing )
add_subdirectory( AppInfrastructure/Common )
add_subdirectory( AppInfrastructure/IpcService )
add_subdirectory( AppInfrastructure/ReadLine )

# Add Utils
add_subdirectory(utils)
add_subdirectory(ipcUtils)
add_subdirectory(settings)

# Add Bundle Generator code
add_subdirectory(bundle/lib)
if ( LEGACY_COMPONENTS )
    add_subdirectory(bundle/tool)
endif()

# Add the dobby daemon code
add_subdirectory( daemon/init )
add_subdirectory( daemon/lib )
add_subdirectory( daemon/process )

# Add legacy plugins
if( LEGACY_COMPONENTS )
    add_subdirectory( plugins/Common )
    add_subdirectory( plugins/AppServices )
    add_subdirectory( plugins/EthanLog )
    add_subdirectory( plugins/OpenCDM )
    add_subdirectory( plugins/Perfetto )
endif()

# Add the plugin launcher
add_subdirectory(pluginLauncher/lib)
add_subdirectory(pluginLauncher/tool)

# Client library
add_subdirectory( client/lib )

# Add the dobby tool on debug builds
if( BUILD_TYPE STREQUAL "DEBUG" )
    add_subdirectory( client/tool )
endif()

# RDK Plugins
# ---------------------------------------------------------

# Add the common subdirectory
add_subdirectory(rdkPlugins/Common)

# Default RDK plugins
option(PLUGIN_LOGGING "Include Logging plugin" ON)
option(PLUGIN_NETWORKING "Include Networking plugin" ON)
option(PLUGIN_IPC "Include IPC plugin" ON)
option(PLUGIN_STORAGE "Include Storage plugin" ON)

# Optional RDK plugins
option(PLUGIN_TESTPLUGIN "Include TestPlugin plugin" OFF)
option(PLUGIN_GPU "Include GPU plugin" OFF)
option(PLUGIN_LOCALTIME "Include LocalTime plugin" OFF)
option(PLUGIN_RTSCHEDULING "Include RtScheduling plugin" OFF)
option(PLUGIN_HTTPPROXY "Include HttpProxy plugin" OFF)

if(PLUGIN_TESTPLUGIN)
    add_subdirectory(rdkPlugins/TestPlugin)
endif()

if(PLUGIN_LOGGING)
    add_subdirectory(rdkPlugins/Logging)
endif()

if(PLUGIN_NETWORKING)
    add_subdirectory(rdkPlugins/Networking)
endif()

if(PLUGIN_IPC)
    add_subdirectory(rdkPlugins/IPC)
endif()

if(PLUGIN_STORAGE)
    add_subdirectory(rdkPlugins/Storage)
endif()

if(PLUGIN_GPU)
    add_subdirectory(rdkPlugins/GPU)
endif()

if(PLUGIN_LOCALTIME)
    add_subdirectory(rdkPlugins/LocalTime)
endif()

if(PLUGIN_RTSCHEDULING)
    add_subdirectory(rdkPlugins/RtScheduling)
endif()

if(PLUGIN_HTTPPROXY)
    add_subdirectory(rdkPlugins/HttpProxy)
endif()

# Export targets in Dobby package
# ---------------------------------------------------------


# Register dobby build-tree with a global CMake registry
export( PACKAGE Dobby )

# Add targets to build-tree export set DobbyTargets.cmake
export( TARGETS AppInfraCommon AppInfraLogging IpcService DobbyClientLib
        FILE "${PROJECT_BINARY_DIR}/DobbyTargets.cmake"
)


# TODO: Instead of manually setting paths for exported target includes, get them automatically from targets
# Add the paths to CONF_INCLUDE_DIRS
set( CONF_INCLUDE_DIRS
      "${PROJECT_SOURCE_DIR}"
      "${PROJECT_SOURCE_DIR}/client/lib/include"
      "${PROJECT_SOURCE_DIR}/protocol/include"
      "${PROJECT_SOURCE_DIR}/AppInfrastructure/Public"
      "${PROJECT_SOURCE_DIR}/AppInfrastructure/IpcService/include"
      "${PROJECT_SOURCE_DIR}/AppInfrastructure/Common/include"
)

# Generate config files from .in files
configure_file( DobbyConfig.cmake.in "${PROJECT_BINARY_DIR}/DobbyConfig.cmake" @ONLY )
configure_file( DobbyConfigVersion.cmake.in "${PROJECT_BINARY_DIR}/DobbyConfigVersion.cmake" @ONLY )

# Install the DobbyConfig.cmake and DobbyConfigVersion.cmake
install( FILES
            "${PROJECT_BINARY_DIR}/DobbyConfig.cmake"
            "${PROJECT_BINARY_DIR}/DobbyConfigVersion.cmake"
        DESTINATION "${INSTALL_CMAKE_DIR}"
)

# Install the export set for use with the install-tree
install( EXPORT DobbyTargets
        DESTINATION "${INSTALL_CMAKE_DIR}"
)
