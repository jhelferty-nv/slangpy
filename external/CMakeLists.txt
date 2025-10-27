# Avoid warning about DOWNLOAD_EXTRACT_TIMESTAMP in CMake 3.24:
if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
    cmake_policy(SET CMP0135 NEW)
endif()

include(FetchContent)

# Use FetchContent to download and populate a package without calling add_subdirectory().
# This is used for downloading prebuilt external binaries.
macro(sgl_download_package name url)
    FetchContent_Declare(
        ${name}
        URL ${url}
        SOURCE_SUBDIR _does_not_exist_ # avoid adding contained CMakeLists.txt
        HTTP_HEADER "Authorization: token ${SLANG_GITHUB_TOKEN}"
    )
    FetchContent_GetProperties(${name})
    if(NOT ${name}_POPULATED)
        message(STATUS "Populating ${name} ...")
        FetchContent_MakeAvailable(${name})
    endif()
endmacro()

# d3d12

add_library(d3d12 INTERFACE)
target_link_libraries(d3d12 INTERFACE dxgi.lib d3d12.lib dxguid.lib)

# fmt

set(FMT_INSTALL OFF)
add_subdirectory(fmt)

# glfw

set(GLFW_BUILD_EXAMPLES OFF)
set(GLFW_BUILD_TESTS OFF)
set(GLFW_BUILD_DOCS OFF)
set(GLFW_INSTALL OFF)
add_subdirectory(glfw)

# tevclient

add_subdirectory(tevclient EXCLUDE_FROM_ALL)

# nanobind

if(SGL_BUILD_PYTHON)
    add_subdirectory(nanobind)
    nanobind_build_library(nanobind SHARED)
endif()

# nanothread

set(NANOTHREAD_STATIC ON)
add_subdirectory(nanothread)
# Disable LTO for nanothread as this causes problems when linking with sgl.
set_target_properties(nanothread PROPERTIES INTERPROCEDURAL_OPTIMIZATION_RELEASE FALSE)

# lmdb

add_library(lmdb STATIC)
target_include_directories(lmdb INTERFACE lmdb)
target_sources(lmdb PRIVATE lmdb/mdb.c lmdb/midl.c)

# slang

set(SGL_SLANG_DEBUG_INFO ON CACHE BOOL "Download debug info for slang")

set(SGL_LOCAL_SLANG OFF CACHE BOOL "Use a local build of slang instead of downloading a prebuilt binary")
set(SGL_LOCAL_SLANG_DIR "${CMAKE_SOURCE_DIR}/../slang" CACHE PATH "Path to a local slang build")
set(SGL_LOCAL_SLANG_BUILD_DIR "build/Debug" CACHE STRING "Build directory of the local slang build")

set(SLANG_VERSION "2025.19.1")
set(SLANG_URL_BASE "https://github.com/shader-slang/slang/releases/download/v${SLANG_VERSION}/slang-${SLANG_VERSION}")

if(SGL_WINDOWS)
    if(SGL_ARCHITECTURE MATCHES "x86_64")
        set(SLANG_URL "${SLANG_URL_BASE}-windows-x86_64.zip")
        set(SLANG_DEBUG_INFO_URL "${SLANG_URL_BASE}-windows-x86_64-debug-info.zip")
    elseif(SGL_ARCHITECTURE MATCHES "aarch64|arm64")
        set(SLANG_URL "${SLANG_URL_BASE}-windows-aarch64.zip")
        set(SLANG_DEBUG_INFO_URL "${SLANG_URL_BASE}-windows-aarch64-debug-info.zip")
    endif()
elseif(SGL_LINUX)
    if(SGL_ARCHITECTURE MATCHES "x86_64")
        set(SLANG_URL "${SLANG_URL_BASE}-linux-x86_64.tar.gz")
        set(SLANG_DEBUG_INFO_URL "${SLANG_URL_BASE}-linux-x86_64-debug-info.tar.gz")
    elseif(SGL_ARCHITECTURE MATCHES "aarch64|arm64")
        set(SLANG_URL "${SLANG_URL_BASE}-linux-aarch64.tar.gz")
        set(SLANG_DEBUG_INFO_URL "${SLANG_URL_BASE}-linux-aarch64-debug-info.tar.gz")
    endif()
elseif(SGL_MACOS)
    if(CMAKE_APPLE_SILICON_PROCESSOR MATCHES "x86_64")
        set(SLANG_URL "${SLANG_URL_BASE}-macos-x86_64.zip")
        set(SLANG_DEBUG_INFO_URL "${SLANG_URL_BASE}-macos-x86_64-debug-info.zip")
    else()
        set(SLANG_URL "${SLANG_URL_BASE}-macos-aarch64.zip")
        set(SLANG_DEBUG_INFO_URL "${SLANG_URL_BASE}-macos-aarch64-debug-info.zip")
    endif()
endif()

sgl_download_package(slang ${SLANG_URL})

set(SLANG_DIR ${slang_SOURCE_DIR})
set(SLANG_INCLUDE_DIR ${slang_SOURCE_DIR}/include)

if(SGL_LOCAL_SLANG)
    set(SLANG_DIR ${SGL_LOCAL_SLANG_DIR}/${SGL_LOCAL_SLANG_BUILD_DIR})
    set(SLANG_INCLUDE_DIR ${SGL_LOCAL_SLANG_DIR}/include)
endif()

# Download debug info package if needed
if(NOT SGL_LOCAL_SLANG AND SGL_SLANG_DEBUG_INFO)
    sgl_download_package(slang_debug_info ${SLANG_DEBUG_INFO_URL})
endif()

# Detect whether we have new (renamed) slang or old slang
# Check for slang-compiler existence to determine this
if(SGL_WINDOWS)
    if(EXISTS "${SLANG_DIR}/bin/slang-compiler.dll")
        set(SLANG_USE_NEW_NAMES ON)
    else()
        set(SLANG_USE_NEW_NAMES OFF)
    endif()
elseif(SGL_MACOS)
    # Check for either versioned or unversioned slang-compiler
    if(EXISTS "${SLANG_DIR}/lib/libslang-compiler.dylib")
        set(SLANG_USE_NEW_NAMES ON)
    else()
        set(SLANG_USE_NEW_NAMES OFF)
    endif()
else() # Linux
    # Check for either versioned or unversioned slang-compiler
    if(EXISTS "${SLANG_DIR}/lib/libslang-compiler.so")
        set(SLANG_USE_NEW_NAMES ON)
    else()
        set(SLANG_USE_NEW_NAMES OFF)
    endif()
endif()

if(SLANG_USE_NEW_NAMES)
    message(STATUS "Detected slang with new naming scheme (libslang-compiler)")
else()
    message(STATUS "Detected slang with old naming scheme (libslang)")
endif()

# Helper function to copy a file while preserving symlinks
function(sgl_copy_file_preserve_symlink src dst_dir)
    get_filename_component(filename ${src} NAME)
    set(dst ${SGL_OUTPUT_DIRECTORY}/${dst_dir}/${filename})

    # Add custom command that preserves symlinks
    if(IS_SYMLINK ${src})
        file(READ_SYMLINK ${src} link_target)
        add_custom_command(
            OUTPUT ${dst}
            DEPENDS ${src}
            COMMAND ${CMAKE_COMMAND} -E create_symlink ${link_target} ${dst}
            COMMENT "Creating symlink ${filename} -> ${link_target}"
        )
        set_property(GLOBAL APPEND PROPERTY SGL_BINARY_FILES_DST ${dst})

        # For install, we need to handle symlinks specially
        cmake_path(SET install_dir NORMALIZE "${CMAKE_INSTALL_BINDIR}/${dst_dir}")
        install(CODE "
            execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink
                \"${link_target}\"
                \"\${CMAKE_INSTALL_PREFIX}/${install_dir}/${filename}\")
        ")
    else()
        # Regular file, use normal copy
        sgl_copy_binary(${src} ${dst_dir})
    endif()
endfunction()

# Helper macro to copy a slang library and optionally its debug info
# lib_name_base: Base name like "slang-compiler", "slang-glslang", etc.
# lib_type: Either "LINKABLE" (for libraries that can be linked) or "MODULE" (for runtime modules)
#           or "UNVERSIONED" (for libraries with no version like llvm)
# Third parameter (optional): HAS_DEBUG_INFO to also copy debug info for this library
macro(sgl_copy_slang_library lib_name_base lib_type)
    # Handle optional third parameter
    set(copy_debug_info OFF)
    if(${ARGC} GREATER 2)
        if(${ARGV2} STREQUAL "HAS_DEBUG_INFO")
            set(copy_debug_info ON)
        endif()
    endif()

    # Validate lib_type
    if(NOT (lib_type STREQUAL "LINKABLE" OR lib_type STREQUAL "MODULE" OR lib_type STREQUAL "UNVERSIONED"))
        message(FATAL_ERROR "sgl_copy_slang_library: Invalid lib_type '${lib_type}'. Must be LINKABLE, MODULE, or UNVERSIONED")
    endif()

    # Build library suffix including extension, separator, and version
    if(SGL_WINDOWS)
        set(lib_dir "${SLANG_DIR}/bin")
        set(lib_suffix ".dll")
        set(debug_dir "${slang_debug_info_SOURCE_DIR}")
    elseif(SGL_MACOS)
        set(lib_dir "${SLANG_DIR}/lib")
        set(debug_dir "${slang_debug_info_SOURCE_DIR}/lib")
        if(SLANG_USE_NEW_NAMES)
            if(lib_type STREQUAL "LINKABLE")
                set(lib_suffix ".0.${SLANG_VERSION}.dylib")
            elseif(lib_type STREQUAL "MODULE")
                set(lib_suffix "-${SLANG_VERSION}.dylib")
            else()
                # Unversioned
                set(lib_suffix ".dylib")
            endif()
        else()
            # Old naming
            set(lib_suffix ".dylib")
        endif()
    else() # Linux
        set(lib_dir "${SLANG_DIR}/lib")
        set(debug_dir "${slang_debug_info_SOURCE_DIR}/lib")
        if(SLANG_USE_NEW_NAMES)
            if(lib_type STREQUAL "LINKABLE")
                set(lib_suffix ".so.0.${SLANG_VERSION}")
            elseif(lib_type STREQUAL "MODULE")
                set(lib_suffix "-${SLANG_VERSION}.so")
            else()
                # Unversioned
                set(lib_suffix ".so")
            endif()
        else()
            # Old naming
            set(lib_suffix ".so")
        endif()
    endif()

    # Find and copy all matching library files (preserving symlinks)
    if(SGL_WINDOWS)
        set(lib_pattern "${lib_dir}/${lib_name_base}${lib_suffix}")
    else()
        set(lib_pattern "${lib_dir}/lib${lib_name_base}*${lib_suffix}*")
    endif()

    file(GLOB lib_files "${lib_pattern}")
    if(lib_files)
        foreach(lib_file ${lib_files})
            # Skip debug info files (they'll be copied separately if requested)
            if(NOT lib_file MATCHES "\\.dwarf$")
                sgl_copy_file_preserve_symlink("${lib_file}" .)
            endif()
        endforeach()
    else()
        message(WARNING "Could not find ${lib_name_base} library matching ${lib_pattern}")
    endif()

    # Copy debug info if requested and available
    if(copy_debug_info AND NOT SGL_LOCAL_SLANG AND SGL_SLANG_DEBUG_INFO)
        if(SGL_WINDOWS)
            set(debug_pattern "${debug_dir}/${lib_name_base}.pdb")
        else()
            set(debug_pattern "${debug_dir}/lib${lib_name_base}*${lib_suffix}.dwarf")
        endif()

        file(GLOB debug_files "${debug_pattern}")
        if(debug_files)
            foreach(debug_file ${debug_files})
                sgl_copy_file_preserve_symlink("${debug_file}" .)
            endforeach()
        else()
            message(WARNING "Could not find ${lib_name_base} debug info matching ${debug_pattern}")
        endif()
    endif()
endmacro()

add_library(slang SHARED IMPORTED GLOBAL)

if(SGL_WINDOWS)
    # Windows: Link to slang-compiler.lib/dll or slang.lib/dll based on detection
    if(SLANG_USE_NEW_NAMES)
        set(SLANG_IMPLIB "${SLANG_DIR}/lib/slang-compiler.lib")
        set(SLANG_DLL "${SLANG_DIR}/bin/slang-compiler.dll")
    else()
        set(SLANG_IMPLIB "${SLANG_DIR}/lib/slang.lib")
        set(SLANG_DLL "${SLANG_DIR}/bin/slang.dll")
    endif()
    set_target_properties(slang PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES ${SLANG_INCLUDE_DIR}
        IMPORTED_IMPLIB ${SLANG_IMPLIB}
        IMPORTED_LOCATION ${SLANG_DLL}
    )
elseif(SGL_LINUX)
    # Linux: Use detected naming scheme
    # Note: If a symlink exists, it will resolve to the versioned file
    if(SLANG_USE_NEW_NAMES)
        set(SLANG_LIB "${SLANG_DIR}/lib/libslang-compiler.so")
    else()
        set(SLANG_LIB "${SLANG_DIR}/lib/libslang.so")
    endif()
    set_target_properties(slang PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES ${SLANG_INCLUDE_DIR}
        IMPORTED_LOCATION ${SLANG_LIB}
    )
elseif(SGL_MACOS)
    # macOS: Use detected naming scheme
    # Note: If a symlink exists, it will resolve to the versioned file
    if(SLANG_USE_NEW_NAMES)
        set(SLANG_LIB "${SLANG_DIR}/lib/libslang-compiler.dylib")
    else()
        set(SLANG_LIB "${SLANG_DIR}/lib/libslang.dylib")
    endif()
    set_target_properties(slang PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES ${SLANG_INCLUDE_DIR}
        IMPORTED_LOCATION ${SLANG_LIB}
    )
endif()

# Copy slang libraries and optionally their debug info (platform-agnostic)
sgl_copy_slang_library("slang-compiler" LINKABLE HAS_DEBUG_INFO)
sgl_copy_slang_library("slang-glslang" MODULE HAS_DEBUG_INFO)
sgl_copy_slang_library("slang-glsl-module" MODULE HAS_DEBUG_INFO)
sgl_copy_slang_library("slang-rt" LINKABLE HAS_DEBUG_INFO)
# slang-llvm is only distributed on Windows (no debug info available)
if(SGL_WINDOWS)
    sgl_copy_slang_library("slang-llvm" UNVERSIONED)
endif()

# Install slang header files
install(FILES ${SLANG_DIR}/include/slang.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(FILES ${SLANG_DIR}/include/slang-deprecated.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(FILES ${SLANG_DIR}/include/slang-image-format-defs.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# slang-rhi

set(SLANG_RHI_BUILD_TESTS OFF)
set(SLANG_RHI_BUILD_EXAMPLES OFF)
set(SLANG_RHI_FETCH_SLANG OFF)
set(SLANG_RHI_ENABLE_D3D11 OFF)
set(SLANG_RHI_ENABLE_WGPU OFF)
# TODO: We should rename SLANG_RHI_BUILD_FROM_SLANG_REPO to something more generic.
# We want slang-rhi to support using a `slang` target that is provided from the outside.
set(SLANG_RHI_BUILD_FROM_SLANG_REPO ON)
set(SLANG_RHI_FETCH_SLANG OFF)
set(SLANG_RHI_SLANG_INCLUDE_DIR ${SLANG_INCLUDE_DIR} CACHE STRING "" FORCE)
set(SLANG_RHI_SLANG_BINARY_DIR ${SLANG_DIR} CACHE STRING "" FORCE)
add_subdirectory(slang-rhi)

if(TARGET slang-rhi-nvapi)
    get_target_property(NVAPI_DIR slang-rhi-nvapi INTERFACE_INCLUDE_DIRECTORIES)
    sgl_copy_shader_file(${NVAPI_DIR}/nvHLSLExtns.h nvapi)
    sgl_copy_shader_file(${NVAPI_DIR}/nvHLSLExtnsInternal.h nvapi)
    sgl_copy_shader_file(${NVAPI_DIR}/nvShaderExtnEnums.h nvapi)
    set(SGL_HAS_NVAPI ON PARENT_SCOPE)
else()
    set(SGL_HAS_NVAPI OFF PARENT_SCOPE)
endif()

if(TARGET slang-rhi-agility-sdk)
    set(SGL_HAS_AGILITY_SDK ON PARENT_SCOPE)
else()
    set(SGL_HAS_AGILITY_SDK OFF PARENT_SCOPE)
endif()

# imgui

set(IMGUI_VERSION "1.90.1")
sgl_download_package(imgui "https://github.com/ocornut/imgui/archive/refs/tags/v${IMGUI_VERSION}.zip")
set(SGL_IMGUI_DIR ${imgui_SOURCE_DIR} PARENT_SCOPE)

# header only

add_library(header_only INTERFACE)
target_include_directories(header_only INTERFACE include)
