cmake_minimum_required(VERSION 3.21)

# Project details
set(PROJECT_AUTHOR "SeaSparrow" CACHE STRING "")
set(NAME "ContainerDistributionFramework" CACHE STRING "")
set(VERSION 2.2.1 CACHE STRING "")

# Options
option(BUILD_TEST "Sets log level to debug." OFF)

# Cache build vars
macro(set_from_environment VARIABLE)
    if(NOT DEFINED ${VARIABLE} AND DEFINED ENV{${VARIABLE}})
        set(${VARIABLE} $ENV{${VARIABLE}})
    endif()
endmacro()

# Find CommonLib path
macro(find_commonlib_path)
    if(DEFINED ENV{CommonLibSSE})
        set(CommonLibPath $ENV{CommonLibSSE})
    elseif(DEFINED ENV{CommonLibSSEPath})
        set(CommonLibPath $ENV{CommonLibSSEPath})
    else()
        # Check extern
        find_path(CommonLibPath
            include/REL/Relocation.h
            PATHS extern/CommonLibSSE
        )
    endif()

    if(NOT CommonLibPath)
        message(FATAL_ERROR "CommonLibSSE path not found. Please set CommonLibSSE or CommonLibSSEPath environment variable.")
    endif()
endmacro()

set_from_environment(VCPKG_ROOT)
find_commonlib_path()

# VCPKG and compiler settings
set(CMAKE_TOOLCHAIN_FILE "${VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
set(VCPKG_TARGET_TRIPLET "x64-windows-static" CACHE STRING "")
set(Boost_USE_STATIC_RUNTIME OFF CACHE BOOL "")
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "")

# Compile definitions
add_compile_definitions(SKYRIM_SE)
add_compile_definitions(SKYRIM_SUPPORT_SE)
if(BUILD_TEST)
    add_compile_definitions(DEBUG)
endif()

# Project configuration
project(
    ${NAME}
    VERSION ${VERSION}
    LANGUAGES CXX
)

# Configure version files
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/include/Version.h
    @ONLY
)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.rc.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.rc
    @ONLY
)

# Include guards
if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
    message(FATAL_ERROR "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there.")
endif()

# Global settings
if(MSVC)
    if(NOT ${CMAKE_GENERATOR} STREQUAL "Ninja")
        add_compile_options(/MP) # Build with Multiple Processes
    endif()
endif()

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DEBUG OFF)
set(Boost_USE_STATIC_LIBS ON)

# Dependencies
add_subdirectory(${CommonLibPath} CommonLibSSE)

find_path(CLIB_UTIL_INCLUDE_DIRS "ClibUtil/utils.hpp")
find_package(jsoncpp CONFIG REQUIRED)

# Source files
include(cmake/headerlist.cmake)
include(cmake/sourcelist.cmake)

# Group source files
source_group(
    TREE ${CMAKE_CURRENT_SOURCE_DIR}
    FILES ${headers} ${sources}
)

source_group(
    TREE ${CMAKE_CURRENT_BINARY_DIR}
    FILES ${CMAKE_CURRENT_BINARY_DIR}/include/Version.h
)

# Create DLL
add_library(
    ${PROJECT_NAME}
    SHARED
    ${headers}
    ${sources}
    ${CMAKE_CURRENT_BINARY_DIR}/include/Version.h
    ${CMAKE_CURRENT_BINARY_DIR}/version.rc
)

# Compiler features and options
target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_23)
target_compile_definitions(${PROJECT_NAME} PRIVATE _UNICODE)

# Include directories
target_include_directories(
    ${PROJECT_NAME}
    PRIVATE
        ${CMAKE_CURRENT_BINARY_DIR}/include
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${CLIB_UTIL_INCLUDE_DIRS}
)

# Link libraries
target_link_libraries(
    ${PROJECT_NAME}
    PRIVATE
        CommonLibSSE::CommonLibSSE
        JsonCpp::JsonCpp
)

# Precompiled headers
target_precompile_headers(
    ${PROJECT_NAME}
    PRIVATE
        include/PCH.h
)

# MSVC-specific options
if(MSVC)
    target_compile_options(
        ${PROJECT_NAME}
        PRIVATE
            /sdl /utf-8 /Zi /permissive- /Zc:preprocessor
            /wd4200
            "$<$<CONFIG:DEBUG>:>"
            "$<$<CONFIG:RELEASE>:/Zc:inline;/JMC-;/Ob3>"
    )

    target_link_options(
        ${PROJECT_NAME}
        PRIVATE
            "$<$<CONFIG:DEBUG>:/INCREMENTAL;/OPT:NOREF;/OPT:NOICF>"
            "$<$<CONFIG:RELEASE>:/INCREMENTAL:NO;/OPT:REF;/OPT:ICF;/DEBUG:FULL>"
    )
endif()