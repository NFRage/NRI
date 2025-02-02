cmake_minimum_required(VERSION 3.15)

message("Processing 'NRI'...")

option(NRI_STATIC_LIBRARY "Build static lib" OFF)
option(NRI_DISABLE_INTERPROCEDURAL_OPTIMIZATION "Disable interprocedural optimization" OFF)
option(NRI_DISABLE_XLIB_SUPPORT "Disable xlib support" OFF)
option(NRI_DISABLE_WAYLAND_SUPPORT "Disable wayland support" OFF)

option(NRI_ENABLE_D3D11 "Enable D3D11 support" OFF)
option(NRI_ENABLE_D3D12 "Enable D3D12 support" OFF)
option(NRI_ENABLE_VULKAN "Enable Vulkan support" ON)

if (NRI_ENABLE_VULKAN)
    find_package(Vulkan REQUIRED FATAL_ERROR)
endif()

if (NOT NRI_ENABLE_D3D11 AND NOT NRI_ENABLE_D3D12 AND NOT NRI_ENABLE_VULKAN)
    message(FATAL_ERROR "You must have at least one selected API in configuration. Please, check your CMake options.")
endif()

file(READ "${CMAKE_CURRENT_SOURCE_DIR}/Include/NRI.h" ver_h)
string(REGEX MATCH "NRI_VERSION_MAJOR ([0-9]*)" _ ${ver_h})
set(ver_major ${CMAKE_MATCH_1})
string(REGEX MATCH "NRI_VERSION_MINOR ([0-9]*)" _ ${ver_h})
set(ver_minor ${CMAKE_MATCH_1})

project(NRI VERSION ${ver_major}.${ver_minor} LANGUAGES CXX)
message(STATUS "${PROJECT_NAME} version: ${ver_major}.${ver_minor}")

if (WIN32)
    set(NRI_PACKMAN_PLATFORM "windows-x86_64")

    execute_process(COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/External/Packman/packman.cmd" pull "${CMAKE_CURRENT_SOURCE_DIR}/Dependencies.xml" -p ${NRI_PACKMAN_PLATFORM}
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        RESULT_VARIABLE NRI_PACKMAN_RESULT)

    if (${NRI_PACKMAN_RESULT} EQUAL 0)
        message(STATUS "Packman result: success (platform: ${NRI_PACKMAN_PLATFORM})")
    else()
        message(FATAL_ERROR "Packman result: ${NRI_PACKMAN_RESULT}")
    endif()
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_POSITION_INDEPENDENT_BINARIES ON)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

if (NOT NRI_DISABLE_INTERPROCEDURAL_OPTIMIZATION)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
endif()

file(GLOB NRI_HEADERS "Include/*.h" "Include/*.hpp")
file(GLOB NRI_HEADERS_EXTENSIONS "Include/Extensions/*.h" "Include/Extensions/*.hpp")
file(GLOB_RECURSE NRI_SHARED_SOURCE "Source/Shared/*.cpp" "Source/Shared/*.h" "Source/Shared/*.hpp")
file(GLOB_RECURSE NRI_D3D11_SOURCE "Source/D3D11/*.cpp" "Source/D3D11/*.h" "Source/D3D11/*.hpp")
file(GLOB_RECURSE NRI_D3D12_SOURCE "Source/D3D12/*.cpp" "Source/D3D12/*.h" "Source/D3D12/*.hpp")
file(GLOB_RECURSE NRI_VK_SOURCE "Source/VK/*.cpp" "Source/VK/*.h" "Source/VK/*.hpp")
file(GLOB_RECURSE NRI_VALIDATION_SOURCE "Source/Validation/*.cpp" "Source/Validation/*.h" "Source/Validation/*.hpp")
file(GLOB_RECURSE NRI_SOURCE "Source/Creation/*.cpp" "Source/Creation/*.h")
file(GLOB NRI_RESOURCE "Resources/*")

source_group("Include" FILES ${NRI_HEADERS})
source_group("Include/Extensions" FILES ${NRI_HEADERS_EXTENSIONS})
source_group("Sources" FILES ${NRI_SOURCE})
source_group("Resources" FILES ${NRI_RESOURCE})
source_group("" FILES ${NRI_SHARED_SOURCE})
source_group("" FILES ${NRI_D3D11_SOURCE})
source_group("" FILES ${NRI_D3D12_SOURCE})
source_group("" FILES ${NRI_VK_SOURCE})
source_group("" FILES ${NRI_VALIDATION_SOURCE})

if (${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_CURRENT_SOURCE_DIR})
    set(NRI_IS_SUBMODULE OFF)
else()
    set(NRI_IS_SUBMODULE ON)
endif()

# defines
if (WIN32)
    set(NRI_COMPILE_DEFINITIONS WIN32_LEAN_AND_MEAN NOMINMAX _CRT_SECURE_NO_WARNINGS _UNICODE UNICODE _ENFORCE_MATCHING_ALLOCATORS=0)
endif()

if (NRI_ENABLE_D3D11)
    set(NRI_PUBLIC_COMPILE_DEFINITIONS ${NRI_PUBLIC_COMPILE_DEFINITIONS} NRI_USE_D3D11=1)
endif()

if (NRI_ENABLE_D3D12)
    set(NRI_PUBLIC_COMPILE_DEFINITIONS ${NRI_PUBLIC_COMPILE_DEFINITIONS} NRI_USE_D3D12=1)
endif()

if (NRI_ENABLE_VULKAN)
    set(NRI_PUBLIC_COMPILE_DEFINITIONS ${NRI_PUBLIC_COMPILE_DEFINITIONS} NRI_USE_VULKAN=1)
endif()

# Dependencies
if (WIN32)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(NRI_WINDOWS_SDK_BIN_ARCHITECTURE "x64")
        set(NRI_NVAPI_BIN_ARCHITECTURE "amd64")
        set(NRI_AGS_BIN_ARCHITECTURE "x64")
    elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
        set(NRI_WINDOWS_SDK_BIN_ARCHITECTURE "x86")
        set(NRI_NVAPI_BIN_ARCHITECTURE "x86")
        set(NRI_AGS_BIN_ARCHITECTURE "x86")
    endif()

    if ("$ENV{WIN10_SDK_PATH}$ENV{WIN10_SDK_VERSION}" STREQUAL "")
        get_filename_component(NRI_WINDOWS_SDK_PATH "[HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Windows Kits\\Installed Roots;KitsRoot10]" ABSOLUTE CACHE)
        set(NRI_WINDOWS_SDK_VERSION ${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION})
        if ("${NRI_WINDOWS_SDK_VERSION}" STREQUAL "")
            set(NRI_WINDOWS_SDK_VERSION ${CMAKE_SYSTEM_VERSION})
            message(STATUS "Using CMAKE_SYSTEM_VERSION as WindowsSDK version. (${CMAKE_SYSTEM_VERSION})")
        endif()
    else()
        set(NRI_WINDOWS_SDK_PATH $ENV{WIN10_SDK_PATH})
        set(NRI_WINDOWS_SDK_VERSION $ENV{WIN10_SDK_VERSION})
    endif()

    if ("${NRI_WINDOWS_SDK_PATH}" STREQUAL "")
        message(FATAL_ERROR "Failed to detect WindowsSDK path. Please add enviroment variables WIN10_SDK_PATH and WIN10_SDK_VERSION or use MSVC.")
    endif()
    if ("${NRI_WINDOWS_SDK_VERSION}" STREQUAL "")
        message(FATAL_ERROR "Failed to detect WindowsSDK version. Please add enviroment variables WIN10_SDK_PATH and WIN10_SDK_VERSION or use MSVC.")
    endif()
    if (IS_DIRECTORY "${NRI_WINDOWS_SDK_PATH}/Include/${NRI_WINDOWS_SDK_VERSION}.0")
        set(NRI_WINDOWS_SDK_VERSION "${NRI_WINDOWS_SDK_VERSION}.0")
    endif()

    set(NRI_WINDOWS_SDK_LIB_PATH "${NRI_WINDOWS_SDK_PATH}/Lib/${NRI_WINDOWS_SDK_VERSION}/um/${NRI_WINDOWS_SDK_BIN_ARCHITECTURE}")

    if (NRI_ENABLE_D3D11)
        find_library(NRI_INPUT_LIB_D3D11 NAMES d3d11 HINTS ${NRI_WINDOWS_SDK_LIB_PATH} REQUIRED)
    endif()

    if (NRI_ENABLE_D3D12)
        find_library(NRI_INPUT_LIB_D3D12 NAMES d3d12 HINTS ${NRI_WINDOWS_SDK_LIB_PATH} REQUIRED)
    endif()

    find_library(NRI_INPUT_LIB_DXGI NAMES dxgi HINTS ${NRI_WINDOWS_SDK_LIB_PATH} REQUIRED)
    find_library(NRI_INPUT_LIB_DXGUID NAMES dxguid HINTS ${NRI_WINDOWS_SDK_LIB_PATH} REQUIRED)
    find_library(NRI_INPUT_LIB_NVAPI NAMES nvapi64 nvapi PATHS "External/NVAPI/${NRI_NVAPI_BIN_ARCHITECTURE}" REQUIRED)
    find_library(NRI_INPUT_LIB_AGS NAMES amd_ags_${NRI_AGS_BIN_ARCHITECTURE} PATHS "External/AGS/lib" REQUIRED)
endif()

# Shared
add_library(NRI_Shared STATIC ${NRI_SHARED_SOURCE})
target_include_directories(NRI_Shared PRIVATE "Include" "Source/Shared")
target_compile_definitions(NRI_Shared PRIVATE ${NRI_COMPILE_DEFINITIONS})
target_compile_definitions(NRI_Shared PUBLIC ${NRI_PUBLIC_COMPILE_DEFINITIONS})
set_property(TARGET NRI_Shared PROPERTY FOLDER "${NRI_PROJECT_FOLDER}")

if (NRI_ENABLE_D3D11)
    add_library(NRI_D3D11 STATIC ${NRI_D3D11_SOURCE})
    target_include_directories(NRI_D3D11 PRIVATE "Include" "Source/Shared" "External")
    target_compile_definitions(NRI_D3D11 PRIVATE ${NRI_COMPILE_DEFINITIONS})
    target_compile_definitions(NRI_D3D11 PUBLIC ${NRI_PUBLIC_COMPILE_DEFINITIONS})
    target_link_libraries(NRI_D3D11 PRIVATE ${NRI_INPUT_LIB_D3D11})
    target_link_libraries(NRI_D3D11 PRIVATE NRI_Shared ${NRI_INPUT_LIB_DXGI} ${NRI_INPUT_LIB_DXGUID} ${NRI_INPUT_LIB_NVAPI})
    set_property(TARGET NRI_D3D11 PROPERTY FOLDER "${NRI_PROJECT_FOLDER}")
endif()

if (NRI_ENABLE_D3D12)
    add_library(NRI_D3D12 STATIC ${NRI_D3D12_SOURCE})
    target_include_directories(NRI_D3D12 PRIVATE "Include" "Source/Shared")
    target_compile_definitions(NRI_D3D12 PRIVATE ${NRI_COMPILE_DEFINITIONS})
    target_compile_definitions(NRI_D3D12 PUBLIC ${NRI_PUBLIC_COMPILE_DEFINITIONS})
    target_link_libraries(NRI_D3D12 PRIVATE ${NRI_INPUT_LIB_D3D12})
    target_link_libraries(NRI_D3D12 PRIVATE NRI_Shared ${NRI_INPUT_LIB_DXGI} ${NRI_INPUT_LIB_DXGUID} ${NRI_INPUT_LIB_NVAPI})
    set_property(TARGET NRI_D3D12 PROPERTY FOLDER "${NRI_PROJECT_FOLDER}")
endif()

# Vulkan
if (NRI_ENABLE_VULKAN)
    add_library(NRI_VK STATIC ${NRI_VK_SOURCE})
    target_include_directories(NRI_VK PRIVATE "Include" "Source/Shared" "External/vulkan/include")
    target_compile_definitions(NRI_VK PRIVATE ${NRI_COMPILE_DEFINITIONS})
    target_compile_definitions(NRI_VK PUBLIC ${NRI_PUBLIC_COMPILE_DEFINITIONS})
    
    if (WIN32)
        target_link_libraries(NRI_VK PRIVATE NRI_Shared ${NRI_INPUT_LIB_DXGI} ${NRI_INPUT_LIB_DXGUID})
    else()
        target_link_libraries(NRI_VK PRIVATE NRI_Shared)
    endif()

    set_property(TARGET NRI_VK PROPERTY FOLDER "${NRI_PROJECT_FOLDER}")

    # Window system defines
    if (WIN32)
        target_compile_definitions(NRI_VK PRIVATE VK_USE_PLATFORM_WIN32_KHR)
    elseif(APPLE)
        target_compile_definitions(NRI_VK PRIVATE VK_USE_PLATFORM_METAL_EXT)
    else()
        if (NOT NRI_DISABLE_XLIB_SUPPORT)
            find_path(NRI_XLIB_HEADERS NAMES "X11/Xlib.h")
            if (NOT NRI_XLIB_HEADERS)
                message(FATAL_ERROR "Can't find 'X11/xlib.h'. (libx11-dev is not installed?)")
            endif()
            target_compile_definitions(NRI_VK PRIVATE VK_USE_PLATFORM_XLIB_KHR)
        endif()
        if (NOT NRI_DISABLE_WAYLAND_SUPPORT)
            find_path(NRI_WAYLAND_HEADERS NAMES "wayland-client.h")
            if (NOT NRI_WAYLAND_HEADERS)
                message(FATAL_ERROR "Can't find 'wayland-client.h'. (libwayland-dev is not installed?)")
            endif()
            target_compile_definitions(NRI_VK PRIVATE VK_USE_PLATFORM_WAYLAND_KHR)
        endif()
    endif()
endif()

# Validation
add_library(NRI_Validation STATIC ${NRI_VALIDATION_SOURCE})
target_include_directories(NRI_Validation PRIVATE "Include" "Source/Shared")
target_compile_definitions(NRI_Validation PRIVATE ${NRI_COMPILE_DEFINITIONS})
target_compile_definitions(NRI_Validation PUBLIC ${NRI_PUBLIC_COMPILE_DEFINITIONS})
target_link_libraries(NRI_Validation PRIVATE NRI_Shared)
set_property(TARGET NRI_Validation PROPERTY FOLDER "${NRI_PROJECT_FOLDER}")

# NRI
if (NRI_STATIC_LIBRARY)
    add_library(${PROJECT_NAME} STATIC ${NRI_SOURCE} ${NRI_HEADERS} ${NRI_RESOURCE} ${NRI_HEADERS_EXTENSIONS})
else()
    add_library(${PROJECT_NAME} SHARED ${NRI_SOURCE} ${NRI_HEADERS} ${NRI_RESOURCE} ${NRI_HEADERS_EXTENSIONS})
endif()

if (UNIX)
    target_link_libraries(${PROJECT_NAME} PRIVATE ${CMAKE_DL_LIBS})
endif()

if (NRI_ENABLE_D3D11)
    target_link_libraries(${PROJECT_NAME} PRIVATE NRI_D3D11)
endif()

if (NRI_ENABLE_D3D12)
    target_link_libraries(${PROJECT_NAME} PRIVATE NRI_D3D12)
endif()

if (NRI_ENABLE_VULKAN)
    target_link_libraries(${PROJECT_NAME} PRIVATE NRI_VK)
endif()

target_link_libraries(${PROJECT_NAME} PRIVATE NRI_Shared NRI_Validation)

target_include_directories(${PROJECT_NAME} PRIVATE "Source/Shared" "External/vulkan/include")
target_include_directories(${PROJECT_NAME} PUBLIC "Include")
target_compile_definitions(${PROJECT_NAME} PRIVATE ${NRI_COMPILE_DEFINITIONS})
target_compile_definitions(${PROJECT_NAME} PUBLIC ${NRI_PUBLIC_COMPILE_DEFINITIONS})
set_property(TARGET ${PROJECT_NAME} PROPERTY FOLDER "${NRI_PROJECT_FOLDER}")
set_target_properties(${PROJECT_NAME} PROPERTIES VERSION ${PROJECT_VERSION})

if (NOT NRI_STATIC_LIBRARY)
    if(WIN32)
        target_compile_definitions(${PROJECT_NAME} PRIVATE "NRI_API=extern \"C\" __declspec(dllexport)")
    else()
        target_compile_definitions(${PROJECT_NAME} PRIVATE "NRI_API=extern \"C\" __attribute__((visibility(\"default\")))")
    endif()
endif()

if (NOT NRI_IS_SUBMODULE)
    set(NRI_OUTPUT_BIN_PATH "${CMAKE_SOURCE_DIR}/_Build")
endif()

if (NOT ${NRI_OUTPUT_BIN_PATH} STREQUAL "")
    message(STATUS "Output path: '${NRI_OUTPUT_BIN_PATH}'")
    set_target_properties(${PROJECT_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${NRI_OUTPUT_BIN_PATH}/$<CONFIG>")
    set_target_properties(${PROJECT_NAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${NRI_OUTPUT_BIN_PATH}/$<CONFIG>")
    set_target_properties(${PROJECT_NAME} PROPERTIES ARCHIVE_OUTPUT_DIRECTORY "${NRI_OUTPUT_BIN_PATH}/$<CONFIG>")
endif()

# warning level 4 and all warnings as errors (with some exceptions)
if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
    set(NRI_COMPILE_OPTIONS -Werror -Wextra -Wno-missing-braces -Wno-unknown-pragmas -Wno-switch -Wno-missing-field-initializers -Wno-deprecated-copy)
elseif (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    #TODO: give up due to "error: enumeral and non-enumeral type in conditional expression [-Werror=extra]" in absolutely valid code!
    set(NRI_COMPILE_OPTIONS -Wextra -Wno-missing-braces -Wno-unknown-pragmas -Wno-switch -Wno-missing-field-initializers -Wno-deprecated-copy)
elseif (MSVC)
    set(NRI_COMPILE_OPTIONS /W4 /WX /wd4324)
endif()

if (NRI_ENABLE_D3D11)
    target_compile_options(NRI_D3D11 PRIVATE ${NRI_COMPILE_OPTIONS})
endif()

if (NRI_ENABLE_D3D12)
    target_compile_options(NRI_D3D12 PRIVATE ${NRI_COMPILE_OPTIONS})
endif()

if (NRI_ENABLE_VULKAN)
    target_compile_options(NRI_VK PRIVATE ${NRI_COMPILE_OPTIONS})
endif()

target_compile_options(NRI_Validation PRIVATE ${NRI_COMPILE_OPTIONS})
target_compile_options(${PROJECT_NAME} PRIVATE ${NRI_COMPILE_OPTIONS})
