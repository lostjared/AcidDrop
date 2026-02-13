cmake_minimum_required(VERSION 3.10)
project(AcidDrop VERSION 1.0 LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

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

# Find required packages
find_package(SDL2 REQUIRED)
find_package(SDL2_ttf REQUIRED)
#find_package(SDL2_image REQUIRED)
find_package(SDL2_mixer REQUIRED)
if(NOT MINGW)
    find_package(Vulkan REQUIRED)
endif()

# Find installed libmx2
find_package(libmx2 REQUIRED)

# Check if we're on macOS with MoltenVK
if(APPLE)
    set(MOLTEN ON)
    set(MOLTEN_LINK_DIR "/usr/local/lib" CACHE PATH "MoltenVK library directory")
    set(MOLTEN_FRAMEWORKS "-framework Cocoa -framework IOKit -framework QuartzCore")
endif()

# Create executable
if(MINGW)
    # Windows icon resource
    set(WIN_RC "${CMAKE_CURRENT_SOURCE_DIR}/icon.rc")
    add_executable(AcidDrop WIN32 acid.drop.cpp ${WIN_RC})
else()
    add_executable(AcidDrop acid.drop.cpp)
endif()

# Set include directories (use installed libmx2 headers)
target_include_directories(AcidDrop PRIVATE
    ${CMAKE_INSTALL_PREFIX}/include/mx2
)

# Link libraries
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
    target_link_libraries(AcidDrop PRIVATE ${MOLTEN_FRAMEWORKS})
    target_link_libraries(AcidDrop PRIVATE 
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

# Set output directory
set_target_properties(AcidDrop PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
)

