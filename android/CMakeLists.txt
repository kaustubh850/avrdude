cmake_minimum_required(VERSION 3.10)
project(libavrdude-android C)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_C_STANDARD 11)
set(BUILD_SHARED_LIBS ON)

# Android flags expected from command line:
# -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake
# -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-21

# Point this to your avrdude sources (adjust paths if you move files)
set(AVRDUDE_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/../)  # repo root
include_directories(${AVRDUDE_ROOT}/src ${AVRDUDE_ROOT}/include)

# Optionally use FetchContent to pull deps here, or assume you vendored under deps/
# Example: find libusb and hidapi built into deps/install/<abi>/lib
# We'll search for system libs first (NDK/lib) and then falling back to deps.

# Collect avrdude source files
file(GLOB AVRDUDE_SRC
  ${AVRDUDE_ROOT}/src/*.c
  ${AVRDUDE_ROOT}/src/libavrdude/*.c
  ${AVRDUDE_ROOT}/src/avrdude/*.c
)

add_library(avrdude SHARED ${AVRDUDE_SRC})

# Link standard Android libs
find_library(log-lib log)
target_link_libraries(avrdude ${log-lib} m dl z) 

# Try to link with libusb, hidapi, libftdi (you must provide or build them)
# Prefer libraries built by the NDK or provided in deps/
find_library(libusb-lib NAMES usb-1.0 usb PATHS ${ANDROID_NDK}/sources ${CMAKE_CURRENT_SOURCE_DIR}/deps INSTALL_CMAKE NO_DEFAULT_PATH)
if(libusb-lib)
  message(STATUS "Found libusb: ${libusb-lib}")
  target_link_libraries(avrdude ${libusb-lib})
else()
  message(STATUS "libusb not found in toolchain — ensure you build vendor libusb")
endif()

# Add your vendored libraries if present (deps/install/<abi>/lib)
# Example usage if you build deps into deps/install:
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/deps/install/${ANDROID_ABI}/lib/libhidapi-libusb.so)
  target_link_libraries(avrdude ${CMAKE_CURRENT_SOURCE_DIR}/deps/install/${ANDROID_ABI}/lib/libhidapi-libusb.so)
endif()

# If you need additional defines to disable readline, ncurses, elf:
target_compile_definitions(avrdude PRIVATE -DNO_READLINE -DNO_NCURSES -DNO_LIBELF)