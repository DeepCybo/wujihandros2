cmake_minimum_required(VERSION 3.8)
project(wujihand_driver)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_compile_options(-Wall -Wextra -Wpedantic)
endif()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(wujihand_msgs REQUIRED)
find_package(Threads REQUIRED)

# Find wujihandcpp. PrimeU pins the official source next to wujihandros2 so a
# release can be rebuilt without first mutating /usr on every controller.
# WUJIHANDPY_SOURCE_DIR remains supported for existing developer machines.
if(DEFINED ENV{WUJIHANDPY_SOURCE_DIR})
    set(WUJIHANDPY_DIR "$ENV{WUJIHANDPY_SOURCE_DIR}")
    set(WUJIHANDCPP_INCLUDE_DIR "${WUJIHANDPY_DIR}/wujihandcpp/include")
    set(WUJIHANDCPP_LIBRARY "${WUJIHANDPY_DIR}/build/wujihandcpp/libwujihandcpp.a")
    set(WUJIHANDCPP_SPDLOG_LIBRARY "${WUJIHANDPY_DIR}/build/_deps/spdlog-build/libspdlog.a")
    message(STATUS "Using wujihandcpp from source: ${WUJIHANDPY_DIR}")
elseif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../../wujihandpy/wujihandcpp/CMakeLists.txt")
    set(WUJIHANDCPP_SOURCE_DIR
        "${CMAKE_CURRENT_SOURCE_DIR}/../../wujihandpy/wujihandcpp")

    # Compile the pinned official sources directly. This keeps the SDK
    # submodule at its published tag while avoiding its network-only
    # FetchContent path. Ubuntu's system spdlog uses external fmt, so generate
    # the one compatibility source in the build tree rather than patching the
    # vendor checkout.
    file(GLOB_RECURSE WUJIHANDCPP_SOURCES CONFIGURE_DEPENDS
        "${WUJIHANDCPP_SOURCE_DIR}/src/*.cpp"
        "${WUJIHANDCPP_SOURCE_DIR}/src/*.c")
    set(WUJIHANDCPP_LOGGING_SOURCE
        "${WUJIHANDCPP_SOURCE_DIR}/src/logging/logging.cpp")
    list(REMOVE_ITEM WUJIHANDCPP_SOURCES "${WUJIHANDCPP_LOGGING_SOURCE}")
    file(READ "${WUJIHANDCPP_LOGGING_SOURCE}" WUJIHANDCPP_LOGGING_CONTENT)
    string(REPLACE
        "#include <spdlog/fmt/bundled/base.h>"
        "#include <fmt/core.h>"
        WUJIHANDCPP_LOGGING_CONTENT
        "${WUJIHANDCPP_LOGGING_CONTENT}")
    set(WUJIHANDCPP_GENERATED_DIR
        "${CMAKE_CURRENT_BINARY_DIR}/generated_wujihandcpp")
    file(MAKE_DIRECTORY "${WUJIHANDCPP_GENERATED_DIR}")
    set(WUJIHANDCPP_GENERATED_LOGGING
        "${WUJIHANDCPP_GENERATED_DIR}/logging.cpp")
    file(WRITE "${WUJIHANDCPP_GENERATED_LOGGING}"
        "${WUJIHANDCPP_LOGGING_CONTENT}")
    list(APPEND WUJIHANDCPP_SOURCES "${WUJIHANDCPP_GENERATED_LOGGING}")

    find_package(spdlog CONFIG REQUIRED)
    add_library(primeu_wujihandcpp STATIC ${WUJIHANDCPP_SOURCES})
    target_compile_features(primeu_wujihandcpp PUBLIC cxx_std_20)
    target_compile_definitions(primeu_wujihandcpp
        PUBLIC STATIC_LINKING_WUJIHANDCPP WUJIHANDCPP_VERSION="1.5.1"
        PRIVATE BUILDING_WUJIHANDCPP)
    target_include_directories(primeu_wujihandcpp
        PUBLIC "${WUJIHANDCPP_SOURCE_DIR}/include"
        PRIVATE "${WUJIHANDCPP_SOURCE_DIR}/src"
        SYSTEM PRIVATE /usr/include/libusb-1.0)
    set_target_properties(primeu_wujihandcpp PROPERTIES
        POSITION_INDEPENDENT_CODE ON)
    target_link_libraries(primeu_wujihandcpp
        PUBLIC spdlog::spdlog usb-1.0 Threads::Threads)

    set(WUJIHANDCPP_INCLUDE_DIR "${WUJIHANDCPP_SOURCE_DIR}/include")
    set(WUJIHANDCPP_LIBRARY primeu_wujihandcpp)
    set(WUJIHANDCPP_SPDLOG_LIBRARY "")
    message(STATUS "Building pinned PrimeU wujihandcpp source: ${WUJIHANDCPP_SOURCE_DIR}")
else()
    find_library(WUJIHANDCPP_LIBRARY NAMES wujihandcpp PATHS /usr/lib)
    find_path(WUJIHANDCPP_INCLUDE_DIR NAMES wujihandcpp/device/hand.hpp PATHS /usr/include)
    find_library(WUJIHANDCPP_SPDLOG_LIBRARY NAMES spdlog PATHS /usr/lib)
    message(STATUS "Using system-installed wujihandcpp")
endif()

if(NOT WUJIHANDCPP_LIBRARY OR NOT WUJIHANDCPP_INCLUDE_DIR)
    message(FATAL_ERROR "wujihandcpp not found. Please install the wujihandcpp deb package or build wujihandpy first.")
endif()

if(NOT WUJIHANDCPP_SPDLOG_LIBRARY AND NOT TARGET ${WUJIHANDCPP_LIBRARY})
    message(FATAL_ERROR "spdlog not found. Please install libspdlog-dev or build wujihandpy first.")
endif()

message(STATUS "Found wujihandcpp: ${WUJIHANDCPP_LIBRARY}")
message(STATUS "Found wujihandcpp include: ${WUJIHANDCPP_INCLUDE_DIR}")

# Build executable
add_executable(wujihand_driver_node
  src/wujihand_driver_node.cpp
  src/main.cpp
)

target_include_directories(wujihand_driver_node PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  ${WUJIHANDCPP_INCLUDE_DIR}
)

ament_target_dependencies(wujihand_driver_node
  rclcpp
  std_msgs
  sensor_msgs
  wujihand_msgs
)

target_link_libraries(wujihand_driver_node
  ${WUJIHANDCPP_LIBRARY}
  ${WUJIHANDCPP_SPDLOG_LIBRARY}
  usb-1.0
  Threads::Threads
)

# Install
install(TARGETS wujihand_driver_node
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY include/
  DESTINATION include
)

ament_package()
