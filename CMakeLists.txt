cmake_minimum_required(VERSION 3.10)
project(AcidDrop VERSION 1.0 LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(MINGW)
    set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
    set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
    add_compile_options(-Wno-narrowing)
    add_link_options(-mwindows)
endif()

# Set prefix path if libmx2 is installed in custom location
list(APPEND CMAKE_PREFIX_PATH "/usr/local")

# ---------------------------------------------------------------------------
# Find required packages
# ---------------------------------------------------------------------------
find_package(SDL2 REQUIRED)
find_package(SDL2_ttf REQUIRED)
find_package(SDL2_mixer REQUIRED)
if(NOT MINGW)
    find_package(Vulkan REQUIRED)
endif()

# Find installed libmx2
find_package(libmx2 REQUIRED)

# ---------------------------------------------------------------------------
# Verify ALL required headers and libraries are reachable
# ---------------------------------------------------------------------------
message(STATUS "")
message(STATUS "=========== Dependency / Header Check ===========")

set(_mx2_inc "${CMAKE_INSTALL_PREFIX}/include/mx2")
set(_check_pass TRUE)

# ---- libmx2 core headers (directly included by acid.drop.cpp) ----
set(_mx2_direct_headers vk.hpp input.hpp argz.hpp loadpng.hpp)
message(STATUS "")
message(STATUS "[libmx2] Checking direct headers in ${_mx2_inc} ...")
foreach(_hdr ${_mx2_direct_headers})
    if(EXISTS "${_mx2_inc}/${_hdr}")
        message(STATUS "  [OK]   ${_hdr}")
    else()
        message(WARNING "  [MISS] ${_hdr}")
        set(_check_pass FALSE)
    endif()
endforeach()

# ---- libmx2 transitive headers (pulled in by the above) ----
set(_mx2_transitive_headers
    config.h mx.hpp exception.hpp util.hpp
    vk_text.hpp vk_sprite.hpp joystick.hpp
    object.hpp font.hpp tee_stream.hpp texture.hpp sound.hpp
)
message(STATUS "")
message(STATUS "[libmx2] Checking transitive headers ...")
foreach(_hdr ${_mx2_transitive_headers})
    if(EXISTS "${_mx2_inc}/${_hdr}")
        message(STATUS "  [OK]   ${_hdr}")
    else()
        message(WARNING "  [MISS] ${_hdr}")
        set(_check_pass FALSE)
    endif()
endforeach()

# ---- volk header (used by vk.hpp) ----
message(STATUS "")
message(STATUS "[volk] Checking volk header ...")
if(EXISTS "${_mx2_inc}/volk/volk.h")
    message(STATUS "  [OK]   volk/volk.h")
else()
    message(WARNING "  [MISS] volk/volk.h")
    set(_check_pass FALSE)
endif()

# ---- SDL2 headers ----
message(STATUS "")
message(STATUS "[SDL2] Checking SDL2 headers ...")
find_path(SDL2_INCLUDE_CHECK SDL.h
    HINTS ${SDL2_INCLUDE_DIRS}
    PATH_SUFFIXES SDL2
)
if(SDL2_INCLUDE_CHECK)
    message(STATUS "  [OK]   SDL.h  (${SDL2_INCLUDE_CHECK})")
else()
    message(WARNING "  [MISS] SDL.h")
    set(_check_pass FALSE)
endif()

set(_sdl2_headers SDL_vulkan.h SDL_ttf.h SDL_mixer.h)
foreach(_hdr ${_sdl2_headers})
    find_path(_sdl2_${_hdr}_dir ${_hdr}
        HINTS ${SDL2_INCLUDE_DIRS}
        PATH_SUFFIXES SDL2
    )
    if(_sdl2_${_hdr}_dir)
        message(STATUS "  [OK]   ${_hdr}")
    else()
        message(WARNING "  [MISS] ${_hdr}")
        set(_check_pass FALSE)
    endif()
    unset(_sdl2_${_hdr}_dir CACHE)
endforeach()

# ---- GLM headers (used by vk.hpp) ----
message(STATUS "")
message(STATUS "[GLM] Checking GLM headers ...")
find_path(_glm_dir glm/glm.hpp)
if(_glm_dir)
    message(STATUS "  [OK]   glm/glm.hpp  (${_glm_dir})")
else()
    message(WARNING "  [MISS] glm/glm.hpp")
    set(_check_pass FALSE)
endif()
unset(_glm_dir CACHE)

find_path(_glm_transform_dir glm/gtc/matrix_transform.hpp)
if(_glm_transform_dir)
    message(STATUS "  [OK]   glm/gtc/matrix_transform.hpp")
else()
    message(WARNING "  [MISS] glm/gtc/matrix_transform.hpp")
    set(_check_pass FALSE)
endif()
unset(_glm_transform_dir CACHE)

# ---- Vulkan library ----
message(STATUS "")
message(STATUS "[Vulkan] Checking Vulkan library ...")
if(NOT MINGW)
    if(Vulkan_FOUND)
        message(STATUS "  [OK]   Vulkan ${Vulkan_VERSION}  (${Vulkan_LIBRARIES})")
    else()
        message(WARNING "  [MISS] Vulkan SDK not found")
        set(_check_pass FALSE)
    endif()
else()
    message(STATUS "  [SKIP] MINGW build – Vulkan loaded at runtime via volk")
endif()

# ---- libmx2 library files ----
message(STATUS "")
message(STATUS "[libmx2] Checking library files ...")
set(_mx2_libs
    "${CMAKE_INSTALL_PREFIX}/lib/libmx.so"
    "${CMAKE_INSTALL_PREFIX}/lib/libmxvk.a"
    "${CMAKE_INSTALL_PREFIX}/lib/libvolk.a"
)
foreach(_lib ${_mx2_libs})
    if(EXISTS "${_lib}")
        get_filename_component(_name "${_lib}" NAME)
        message(STATUS "  [OK]   ${_name}  (${_lib})")
    else()
        message(WARNING "  [MISS] ${_lib}")
        set(_check_pass FALSE)
    endif()
endforeach()

# ---- libmx2 CMake config ----
message(STATUS "")
message(STATUS "[libmx2] Checking CMake package config ...")
if(EXISTS "${CMAKE_INSTALL_PREFIX}/lib/cmake/libmx2/libmx2Config.cmake")
    message(STATUS "  [OK]   libmx2Config.cmake")
else()
    message(WARNING "  [MISS] libmx2Config.cmake")
    set(_check_pass FALSE)
endif()

# ---- SDL2 libraries ----
message(STATUS "")
message(STATUS "[SDL2] Checking SDL2 libraries ...")
foreach(_tgt SDL2::SDL2 SDL2_ttf::SDL2_ttf SDL2_mixer::SDL2_mixer)
    if(TARGET ${_tgt})
        message(STATUS "  [OK]   ${_tgt}")
    else()
        message(WARNING "  [MISS] ${_tgt}")
        set(_check_pass FALSE)
    endif()
endforeach()

# ---- libmx2 imported targets ----
message(STATUS "")
message(STATUS "[libmx2] Checking imported targets ...")
foreach(_tgt libmx2::mx libmx2::mxvk libmx2::volk)
    if(TARGET ${_tgt})
        message(STATUS "  [OK]   ${_tgt}")
    else()
        message(WARNING "  [MISS] ${_tgt}")
        set(_check_pass FALSE)
    endif()
endforeach()

# ---- Source file ----
message(STATUS "")
message(STATUS "[Source] Checking source files ...")
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/acid.drop.cpp")
    message(STATUS "  [OK]   acid.drop.cpp")
else()
    message(WARNING "  [MISS] acid.drop.cpp")
    set(_check_pass FALSE)
endif()

# ---- Data directory ----
message(STATUS "")
message(STATUS "[Assets] Checking data directory ...")
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/data")
    file(GLOB _data_files "${CMAKE_CURRENT_SOURCE_DIR}/data/*")
    list(LENGTH _data_files _data_count)
    message(STATUS "  [OK]   data/  (${_data_count} files)")
else()
    message(WARNING "  [MISS] data/ directory not found")
endif()

# ---- Final verdict ----
message(STATUS "")
if(_check_pass)
    message(STATUS "=========== All checks PASSED ===================")
else()
    message(FATAL_ERROR "One or more required headers/libraries are missing. See above.")
endif()
message(STATUS "")

# ---------------------------------------------------------------------------
# macOS / MoltenVK
# ---------------------------------------------------------------------------
if(APPLE)
    set(MOLTEN ON)
    set(MOLTEN_LINK_DIR "/usr/local/lib" CACHE PATH "MoltenVK library directory")
    set(MOLTEN_FRAMEWORKS "-framework Cocoa -framework IOKit -framework QuartzCore")
endif()

# ---------------------------------------------------------------------------
# Create executable
# ---------------------------------------------------------------------------
if(MINGW)
    set(WIN_RC "${CMAKE_CURRENT_SOURCE_DIR}/icon.rc")
    add_executable(AcidDrop WIN32 acid.drop.cpp ${WIN_RC})
else()
    add_executable(AcidDrop acid.drop.cpp)
endif()

# libmx2 imported targets already propagate their include directories via
# INTERFACE_INCLUDE_DIRECTORIES, so an explicit target_include_directories
# call is not required.

# ---------------------------------------------------------------------------
# Link libraries
# ---------------------------------------------------------------------------
if(MINGW)
    target_link_libraries(AcidDrop PRIVATE
        SDL2::SDL2main
        libmx2::mx
        libmx2::mxvk
        SDL2::SDL2
        SDL2_ttf::SDL2_ttf
        SDL2_mixer::SDL2_mixer
    )
elseif(MOLTEN)
    target_link_directories(AcidDrop PRIVATE ${MOLTEN_LINK_DIR})
    target_link_libraries(AcidDrop PRIVATE
        ${MOLTEN_FRAMEWORKS}
        libmx2::mx
        libmx2::mxvk
        SDL2::SDL2
        SDL2_ttf::SDL2_ttf
        SDL2_mixer::SDL2_mixer
        Vulkan::Vulkan
        ${CMAKE_DL_LIBS}
    )
else()
    target_link_libraries(AcidDrop PRIVATE
        libmx2::mx
        libmx2::mxvk
        SDL2::SDL2
        SDL2_ttf::SDL2_ttf
        SDL2_mixer::SDL2_mixer
        Vulkan::Vulkan
        ${CMAKE_DL_LIBS}
    )
endif()

# ---------------------------------------------------------------------------
# Output directory & copy runtime data
# ---------------------------------------------------------------------------
set_target_properties(AcidDrop PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
)

# Copy data assets next to the executable so the game finds them at runtime
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/data")
    add_custom_command(TARGET AcidDrop POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory
            "${CMAKE_CURRENT_SOURCE_DIR}/data"
            "$<TARGET_FILE_DIR:AcidDrop>/data"
        COMMENT "Copying data directory to build output"
    )
endif()

# ---------------------------------------------------------------------------
# Summary
# ---------------------------------------------------------------------------
message(STATUS "----------- AcidDrop build summary -----------")
message(STATUS "  C++ standard : C++${CMAKE_CXX_STANDARD}")
message(STATUS "  Compiler     : ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "  SDL2         : ${SDL2_VERSION}")
message(STATUS "  SDL2_ttf     : found")
message(STATUS "  SDL2_mixer   : found")
if(NOT MINGW)
    message(STATUS "  Vulkan       : ${Vulkan_VERSION}")
endif()
message(STATUS "  libmx2       : found (${CMAKE_INSTALL_PREFIX}/lib)")
message(STATUS "  Build type   : ${CMAKE_BUILD_TYPE}")
message(STATUS "----------------------------------------------")

