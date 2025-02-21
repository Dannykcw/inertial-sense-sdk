CMAKE_MINIMUM_REQUIRED(VERSION 3.10.0)

project(InertialSenseSDK)

# Add libusb as a subdirectory so it gets built first
add_subdirectory(src/libusb)

# Now you can use libusb in the InertialSenseSDK build
file(GLOB SOURCES_SDK
    "src/*.c"
    "src/*.cpp"
    "src/*.h"
    "src/protocol/*.cpp"
    "src/protocol/*.h"
    "src/util/*.cpp"
    "src/util/*.h"
    "hw-libs/bootloader/bootloaderShared.c"
    "tests/runtime/*.cpp"
    "tests/runtime/*.h"
    "src/yaml-cpp/*.cpp"
    "src/yaml-cpp/*.h"
    "src/yaml-cpp/contrib/*.cpp"
    "src/yaml-cpp/contrib/*.h"
)

# # Compile libusb from source
# if (WIN32)
#     file(GLOB SOURCES_LIB_USB
#         "src/libusb/libusb/libusb/*.h"
#         "src/libusb/libusb/libusb/*.c"
#         "src/libusb/libusb/libusb/os/*windows*"
#     )
# elseif (APPLE)
#     file(GLOB SOURCES_LIB_USB
#     "src/libusb/libusb/libusb/*.h"
#     "src/libusb/libusb/libusb/*.c"
#     "src/libusb/libusb/libusb/os/*darwin*"
#     "src/libusb/libusb/libusb/os/*posix*"
#     )
# else ()
#     file(GLOB SOURCES_LIB_USB
#         "src/libusb/libusb/libusb/*.h"
#         "src/libusb/libusb/libusb/*.c"
#         "src/libusb/libusb/libusb/os/*linux*"
#         "src/libusb/libusb/libusb/os/*posix*"
#     )
# endif ()

# Ignore bootloader files
# list(FILTER SOURCES_SDK EXCLUDE REGEX "ISBootloader.*")
list(FILTER SOURCES_SDK EXCLUDE REGEX "ISBootloaderSony.*")
list(FILTER SOURCES_SDK EXCLUDE REGEX "ISBootloaderSTM32.*")

# list(APPEND SOURCES_SDK ${SOURCES_LIB_USB})

# find_package(udev)
find_library(UDEV_LIB udev)

add_library(${PROJECT_NAME} STATIC ${SOURCES_SDK})

# Include paths
target_include_directories(${PROJECT_NAME} PUBLIC
    src
    src/util
    src/protocol
    src/libusb
    src/libusb/libusb
    src/libusb/libusb/libusb
    src/yaml-cpp
    tests/runtime
    external
)

# Find libusb-1.0 and link it
if (UNIX)
    # Find libusb-1.0 on Linux and macOS
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(LIBUSB REQUIRED libusb-1.0)

    target_link_libraries(${PROJECT_NAME} ${LIBUSB_LIBRARIES})
    target_include_directories(${PROJECT_NAME} PUBLIC ${LIBUSB_INCLUDE_DIRS})

elseif (WIN32)
    # Windows-specific configuration for libusb-1.0
    set(LIBUSB_INCLUDE_DIR "C:/path/to/libusb/include")
    set(LIBUSB_LIBRARIES "C:/path/to/libusb/lib/libusb-1.0.a")

    target_include_directories(${PROJECT_NAME} PUBLIC ${LIBUSB_INCLUDE_DIR})
    target_link_libraries(${PROJECT_NAME} ${LIBUSB_LIBRARIES})
endif()

# Build using C11 and CXX20
set_property(TARGET ${PROJECT_NAME} PROPERTY C_STANDARD 11)
set_property(TARGET ${PROJECT_NAME} PROPERTY CXX_STANDARD 20)
set_property(TARGET ${PROJECT_NAME} PROPERTY POSITION_INDEPENDENT_CODE ON)

# Other settings
if (WIN32)
    # yamlcpp needs to know it's a static lib.
    add_definitions(-DYAML_CPP_STATIC_DEFINE)

    # Windows specific include dir
    target_include_directories(${PROJECT_NAME} PUBLIC
        ${CMAKE_CURRENT_LIST_DIR}/src/libusb/libusb/msvc
    )

    # We need to ensure windows static libs are found
    target_link_libraries(${PROJECT_NAME} Ws2_32.lib)
elseif (APPLE)
    # macOS-specific include directory
    target_include_directories(${PROJECT_NAME} PUBLIC
        ${CMAKE_CURRENT_LIST_DIR}/src/libusb/libusb/macos
    )

    # macOS typically requires linking against CoreFoundation and IOKit for USB access
    find_library(COREFOUNDATION_FRAMEWORK CoreFoundation)
    find_library(IOKIT_FRAMEWORK IOKit)
    find_library(SECURITY Security)

    # Link against macOS frameworks needed for USB support
    target_link_libraries(${PROJECT_NAME} 
        ${COREFOUNDATION_FRAMEWORK} 
        ${IOKIT_FRAMEWORK}
        ${SECURITY}
    )
else ()
    # Linux specific include dir
    target_include_directories(${PROJECT_NAME} PUBLIC
        ${CMAKE_CURRENT_LIST_DIR}/src/libusb/libusb/linux
    )

    # Linux compiler flags
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wno-unused-function -Wno-unused-parameter -Wno-missing-field-initializers")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wno-unused-function -Wno-unused-parameter -Wno-missing-field-initializers")

    # Set Linux compiler linker flag
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")

    # Link in Linux specific packages
    target_link_libraries(${PROJECT_NAME} udev m)
endif ()
