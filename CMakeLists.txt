cmake_minimum_required(VERSION 3.10)
project(AcidDrop VERSION 1.0 LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Set prefix path if libmx2 is installed in custom location
list(APPEND CMAKE_PREFIX_PATH "/usr/local/mxvk")

# Find required packages
find_package(SDL2 REQUIRED)
find_package(SDL2_ttf REQUIRED)
find_package(SDL2_image REQUIRED)
find_package(SDL2_mixer REQUIRED)
find_package(Vulkan REQUIRED)

# Find installed libmx2
find_package(libmx2 REQUIRED)

# Set volk path
set(VOLK_DIR "${CMAKE_CURRENT_SOURCE_DIR}/volk" CACHE PATH "Path to volk directory")

# Check if we're on macOS with MoltenVK
if(APPLE)
    set(MOLTEN ON)
    set(MOLTEN_LINK_DIR "/usr/local/lib" CACHE PATH "MoltenVK library directory")
    set(MOLTEN_FRAMEWORKS "-framework Cocoa -framework IOKit -framework QuartzCore")
endif()

# Build or find volk (if not on macOS)
if(NOT MOLTEN)
    if(EXISTS "${VOLK_DIR}/volk.cpp")
        add_library(volk STATIC ${VOLK_DIR}/volk.cpp)
        target_include_directories(volk PUBLIC ${VOLK_DIR})
        target_link_libraries(volk PUBLIC ${CMAKE_DL_LIBS})
    else()
        message(WARNING "volk not found at ${VOLK_DIR}, will try to link against installed version")
        find_library(VOLK_LIB volk REQUIRED)
    endif()
endif()

# Create executable
add_executable(AcidDrop acid.drop.cpp vk.cpp)

# Set include directories
target_include_directories(AcidDrop PRIVATE 
    ${VOLK_DIR}
)

# Link libraries
if(MOLTEN)
    target_link_directories(AcidDrop PRIVATE ${MOLTEN_LINK_DIR})
    target_link_libraries(AcidDrop PRIVATE ${MOLTEN_FRAMEWORKS})
    target_link_libraries(AcidDrop PRIVATE 
        libmx2::mx
        SDL2::SDL2
        SDL2_ttf::SDL2_ttf
        SDL2_mixer::SDL2_mixer
        Vulkan::Vulkan
        ${CMAKE_DL_LIBS}
    )
else()
    target_link_libraries(AcidDrop PRIVATE 
        libmx2::mx
        volk
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

