cmake_minimum_required(VERSION 2.8.3)
project(rotors_gazebo_plugins_wind)

set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
message(STATUS "CMAKE_BINARY_DIR = ${CMAKE_BINARY_DIR}")

# Initialise a list which will keep track of all targets
# that need to be installed.
set(targets_to_install "")

# Specify C++11 standard
add_definitions(-std=c++11)

# To enable assertions when compiled in release mode.
add_definitions(-DROS_ASSERT_ENABLED)

find_package(catkin REQUIRED COMPONENTS
  gazebo_plugins
  geometry_msgs
  mav_msgs
  roscpp
  std_srvs
  tf
)

find_package(Eigen REQUIRED)
find_package(gazebo_dev QUIET)
if (NOT GAZEBO_DEV_FOUND)
  find_package(gazebo REQUIRED)
endif()

if(${gazebo_VERSION_MAJOR} LESS 9)
  if(${gazebo_VERSION_MAJOR} LESS 7)
    message(WARN "GAZEBO Version ${gazebo_VERSION_MAJOR}.${gazebo_VERSION_MINOR}.${gazebo_VERSION_PATCH}")
    message(WARN "GAZEBO older than v9 detected. Autobackporting source files")

    execute_process(COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/autobackport/commandpre7.sh" "" WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" RESULT_VARIABLE git_result)

  else()
    message(WARN "GAZEBO Version ${gazebo_VERSION_MAJOR}.${gazebo_VERSION_MINOR}.${gazebo_VERSION_PATCH}")
    message(WARN "GAZEBO older than v9 detected. Autobackporting source files")

    execute_process(COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/autobackport/command.sh" "" WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" RESULT_VARIABLE git_result)
  endif()

endif()

link_directories(${GAZEBO_LIBRARY_DIRS})
include_directories(${GAZEBO_INCLUDE_DIRS})
include_directories(${CMAKE_CURRENT_BINARY_DIR})


# ============================================== #
# =================== YAML_CPP ================= #
# ============================================== #

find_package(yaml_cpp_catkin QUIET)
if(${yaml_cpp_catkin_FOUND})
  message(STATUS "Found yaml_cpp_catkin, using instead of system library.")
  set(YamlCpp_LIBRARIES ${yaml_cpp_catkin_LIBRARIES})
  set(YamlCpp_INCLUDE_DIRS ${yaml_cpp_catkin_INCLUDE_DIRS})
else()
  message(STATUS "No yaml_cpp_catkin, using yaml-cpp system library instead.")
  pkg_check_modules(YamlCpp REQUIRED yaml-cpp>=0.5)
endif()
include_directories(${YamlCpp_INCLUDE_DIR})

# ============================================== #
# =================== PROTOBUF ================= #
# ============================================== #

set(PROTOBUF_IMPORT_DIRS "")
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(PROTOBUF_IMPORT_DIRS "${ITR}/gazebo/msgs/proto")
  endif()
endforeach()
message(STATUS "PROTOBUF_IMPORT_DIRS = " ${PROTOBUF_IMPORT_DIRS})

# protobuf required for publishing/subscribing to Gazebo
# messages
# WARNING: THIS MUST BE RUN AFTER PROTOBUF_IMPORT_DIRS IS SETUP
find_package(Protobuf REQUIRED)
pkg_check_modules(PROTOBUF protobuf)

# Protobuf version check
if ("${PROTOBUF_VERSION}" VERSION_LESS "2.5.0")
  message(FATAL_ERROR "protobuf version: ${PROTOBUF_VERSION} not compatible, must be >= 2.5.0")
endif()

# After running this block of code,
# we should have something similar to
# GAZEBO_MSG_INCLUDE_DIRS = /usr/include/gazebo-7/gazebo/msgs
set(GAZEBO_MSG_INCLUDE_DIRS)
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(GAZEBO_MSG_INCLUDE_DIRS "${ITR}/gazebo/msgs")
  endif()
endforeach()

# Get lists of all .proto files in the msgs directory
file(GLOB msgs msgs/*.proto)

# Finally, generate the .cpp files from the .proto files
PROTOBUF_GENERATE_CPP(PROTO_SRCS PROTO_HDRS ${msgs})

# Create a shared library of protobuf messages (.so extension on Linux platforms)
add_library(mav_msgs SHARED ${PROTO_SRCS})
target_link_libraries(mav_msgs ${PROTOBUF_LIBRARY} gazebo_msgs)

# This causes mav_msgs to be linked with every created library in this file from this
# point forward.
# NOTE: This is deprecated, should be using target_link_libraries instead
link_libraries(mav_msgs)

# ============================================== #
# ==================== CATKIN ================== #
# ============================================== #

if(${gazebo_dev_FOUND})
  catkin_package(
    #INCLUDE_DIRS include ${Eigen_INCLUDE_DIRS}
    #LIBRARIES rotors_gazebo_motor_model rotors_gazebo_controller_interface
    #CATKIN_DEPENDS cv_bridge geometry_msgs mav_msgs octomap_msgs octomap_ros rosbag roscpp gazebo_plugins rotors_comm rotors_control std_srvs tf
    #DEPENDS Eigen octomap OpenCV
    #CFG_EXTRAS rotors_gazebo_plugins.cmake
  )
  set(target_linking_LIBRARIES ${catkin_LIBRARIES} ${gazebo_dev_LIBRARIES})
else()
  catkin_package(
    #INCLUDE_DIRS include ${Eigen_INCLUDE_DIRS}
    #LIBRARIES rotors_gazebo_motor_model rotors_gazebo_controller_interface
    #CATKIN_DEPENDS cv_bridge geometry_msgs mav_msgs octomap_msgs octomap_ros rosbag roscpp gazebo_plugins rotors_comm rotors_control std_srvs tf
    #DEPENDS Eigen octomap OpenCV
    ##CFG_EXTRAS rotors_gazebo_plugins.cmake
  )
  set(target_linking_LIBRARIES ${catkin_LIBRARIES} ${GAZEBO_LIBRARIES})
endif()


# Including GAZEBO_MSG_INCLUDE_DIRS here allows the .cpp files generated by custom .proto files to find
# the .cpp/hpp files generated by the built-in Gazebo .proto files
include_directories(include ${catkin_INCLUDE_DIRS} ${GAZEBO_MSG_INCLUDE_DIRS})
include_directories(${Eigen_INCLUDE_DIRS})


# =============================================================================================== #
# ========================================= USER LIBRARIES ====================================== #
# =============================================================================================== #

#========================================= WIND PLUGIN ==========================================//
add_library(rotors_gazebo_wind_plugin SHARED src/gazebo_wind_plugin.cpp)
target_link_libraries(rotors_gazebo_wind_plugin ${target_linking_LIBRARIES} )
if (NOT NO_ROS)
  add_dependencies(rotors_gazebo_wind_plugin ${catkin_EXPORTED_TARGETS})
endif()
list(APPEND targets_to_install rotors_gazebo_wind_plugin)

set(BIN_DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})
set(LIB_DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION})

# Install all targets that were defined above (some conditionally) and added to the
# variable targets_to_install
install(
  TARGETS
    ${targets_to_install}
  DESTINATION ${BIN_DESTINATION}
  LIBRARY DESTINATION ${LIB_DESTINATION}
)
