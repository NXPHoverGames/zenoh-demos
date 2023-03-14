cmake_minimum_required(VERSION 3.19)

# This is your project statement. You should always list languages;
# Listing the version is nice here since it sets lots of useful variables
project(
  zenoh-ros2
  VERSION 1.0
  LANGUAGES C)

# Platform information for zenoh
if(CMAKE_SYSTEM_NAME MATCHES "Linux")
  add_definitions(-DZENOH_LINUX)
  set(JNI_PLATFORM_NAME "linux")
else()
  message(FATAL_ERROR "zenoh-pico is not yet available on ${CMAKE_SYSTEM_NAME} platform")
  return()
endif()



if(NOT DEFINED ROS_DISTRO)
  if(NOT DEFINED ENV{ROS_DISTRO})
    message(FATAL_ERROR "No ROS_DISTRO defined")
  else()
    set(ROS_DISTRO $ENV{ROS_DISTRO})
  endif()
endif()

message("-- Using ROS ${ROS_DISTRO}")

# ROS Paths
set(ROS_PATH "/opt/ros/${ROS_DISTRO}/share")
set(RCL_INTERFACES_PATH "${ROS_PATH}/rcl_interfaces")
set(BUILTIN_INTERFACES_PATH "${ROS_PATH}/builtin_interfaces")

# We assume that IDLC has been installed on build machine

find_program(IDLC idlc)
if(NOT IDLC)
  message(FATAL_ERROR "IDLC not found!")
endif()

# Runs the command to get the pluginval version
execute_process(COMMAND ${IDLC} -v
                OUTPUT_VARIABLE IDLC_VERSION_RAW_OUTPUT)

string(REPLACE "idlc (Eclipse Cyclone DDS) " "" IDLC_VERSION ${IDLC_VERSION_RAW_OUTPUT})

set(IDLC_MIN_REQ "0.11.0")

if("${IDLC_VERSION}" VERSION_LESS "${IDLC_MIN_REQ}")
  message(FATAL_ERROR "IDLC is version ${IDLC_VERSION}"
                      "Must be greater then ${IDLC_MIN_REQ}\n"
                      "Please compile and install CycloneDDS from source")
endif()

# CycloneDDS config

set(BUILD_IDLC NO)
set(BUILD_LIBCDR ON)

# CycloneDDS IDL
include(FetchContent)
FetchContent_Declare(cyclonedds
  GIT_REPOSITORY "https://github.com/NXPHoverGames/cyclonedds"
  GIT_TAG "origin/cdr_sharedlib_option"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/cyclonedds"
)
FetchContent_MakeAvailable(cyclonedds)

include("${RCL_INTERFACES_PATH}/cmake/rosidl_cmake-extras.cmake")
include("${BUILTIN_INTERFACES_PATH}/cmake/rosidl_cmake-extras.cmake")

foreach(_idl ${rcl_interfaces_IDL_FILES})
  list(APPEND IDL_FILES "${RCL_INTERFACES_PATH}/${_idl}")
endforeach()

foreach(_idl ${builtin_interfaces_IDL_FILES})
  list(APPEND IDL_FILES "${BUILTIN_INTERFACES_PATH}/${_idl}")
endforeach()

idlc_generate(TARGET rcl_interfaces_msgs FILES ${IDL_FILES} INCLUDES ${ROS_PATH} BASE_DIR ${ROS_PATH} WARNINGS no-implicit-extensibility)

set(CMAKE_INCLUDE_CURRENT_DIR ON)
include_directories (${CMAKE_BINARY_DIR})

# Zenoh-Pico
include(FetchContent)
FetchContent_Declare(zenoh-pico
  GIT_REPOSITORY "https://github.com/eclipse-zenoh/zenoh-pico"
  GIT_TAG "origin/master"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/zenoh-pico"
)
FetchContent_MakeAvailable(zenoh-pico)
set(ZENOHPICO_DIR ${CMAKE_CURRENT_BINARY_DIR}/zenoh-pico)
include_directories (${ZENOHPICO_DIR}/include)

# Adding something we can run - Output name matches target name
add_executable(z_pub_ros2
               z_pub_ros2.c
               rcl_interfaces/msg/Log.c
               builtin_interfaces/msg/Time.c)
add_dependencies(z_pub_ros2 zenohpico)

add_executable(z_sub_ros2
               z_sub_ros2.c
               rcl_interfaces/msg/Log.c
               builtin_interfaces/msg/Time.c)
add_dependencies(z_sub_ros2 zenohpico)

set(CMAKE_CXX_IMPLICIT_LINK_DIRECTORIES /usr/local/lib ${CMAKE_CXX_IMPLICIT_LINK_DIRECTORIES})
target_compile_definitions(z_sub_ros2 PRIVATE "DDS_LOG=0")
target_compile_definitions(z_pub_ros2 PRIVATE "DDS_LOG=0")
target_link_libraries(z_sub_ros2 zenohpico cdr)
target_link_libraries(z_pub_ros2 zenohpico cdr)
