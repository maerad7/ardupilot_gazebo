cmake_minimum_required(VERSION 2.8.3 FATAL_ERROR)
cmake_policy(SET CMP0048 NEW)
cmake_policy(SET CMP0054 NEW)

if (NOT CMAKE_INSTALL_PREFIX)
  set(CMAKE_INSTALL_PREFIX "/usr" CACHE STRING "install prefix" FORCE)
endif()

message(STATUS "install-prefix: ${CMAKE_INSTALL_PREFIX}")

project(ardupilot_sitl_gazebo VERSION 1.0.0)
include(GNUInstallDirs)

#######################
## Find Dependencies ##
#######################

# Add search directory for CMake on OS X
list(APPEND CMAKE_MODULE_PATH /usr/local/share/cmake/Modules)

## System dependencies are found with CMake's conventions
find_package(PkgConfig REQUIRED)
find_package(gazebo REQUIRED)
find_package(OpenCV REQUIRED)
find_package(Boost REQUIRED COMPONENTS system thread timer)

# see if catkin was invoked to build this
if (CATKIN_DEVEL_PREFIX)
  message(STATUS "catkin ENABLED")
  find_package(catkin REQUIRED)
  if (catkin_FOUND)
    catkin_package()
  else()
    message(FATAL_ERROR "catkin not found")
  endif()
else()
  message(STATUS "catkin DISABLED")
endif()

# XXX this approach is extremely error prone
# it would be preferable to either depend on the
# compiled headers from Gazebo directly
# or to have something entirely independent.
#

set(PROTOBUF_IMPORT_DIRS "")
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(PROTOBUF_IMPORT_DIRS "${ITR}/gazebo/msgs/proto")
  endif()
endforeach()

# PROTOBUF_IMPORT_DIRS has to be set before
# find_package is called
find_package(Protobuf REQUIRED)
pkg_check_modules(PROTOBUF protobuf)

if ("${PROTOBUF_VERSION}" VERSION_LESS "2.5.0")
  message(FATAL_ERROR "protobuf version: ${PROTOBUF_VERSION} not compatible, must be >= 2.5.0")
endif()

if("${GAZEBO_VERSION}" VERSION_LESS "6.0")
  message(FATAL_ERROR "You need at least Gazebo 6.0. Your version: ${GAZEBO_VERSION}")
else()
  message("Gazebo version: ${GAZEBO_VERSION}")
endif()

find_package(Eigen3 QUIET)
if(NOT EIGEN3_FOUND)
  # Fallback to cmake_modules
  find_package(Eigen QUIET)
  if(NOT EIGEN_FOUND)
    pkg_check_modules(EIGEN3 REQUIRED eigen3)
  else()
    set(EIGEN3_INCLUDE_DIRS ${EIGEN_INCLUDE_DIRS})
    set(EIGEN3_LIBRARIES ${EIGEN_LIBRARIES})
  endif()
else()
  set(EIGEN3_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR})
endif()

find_package(Boost 1.40 COMPONENTS system thread timer REQUIRED )

###########
## Build ##
###########

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}  -std=c99 -Wno-deprecated-declarations")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${GAZEBO_CXX_FLAGS} -std=c++11 -Wno-deprecated-declarations")
# -Wno-deprecated-declarations: Shut up warnings about std::binder1st, std::binder2nd.

set(GAZEBO_MSG_INCLUDE_DIRS)
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(GAZEBO_MSG_INCLUDE_DIRS "${ITR}/gazebo/msgs")
  endif()
endforeach()

include_directories(
  include
  ${PROJECT_SOURCE_DIR}
  ${GAZEBO_INCLUDE_DIRS}
  ${GAZEBO_MSG_INCLUDE_DIRS}
  ${EIGEN3_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
  ${CMAKE_CURRENT_BINARY_DIR}
  ${Boost_INCLUDE_DIR}
  # Workaround for Eigen3
  ${Boost_INCLUDE_DIR}/eigen3
  ${EIGEN3_INCLUDE_DIRS}/eigen3
  # Workaround for OGRE include dirs on Mac OS
  /usr/local/include/OGRE
  /usr/local/include/OGRE/Paging
  )

link_libraries(
  ${PROTOBUF_LIBRARY}
  ${GAZEBO_LIBRARIES}
  ${OpenCV_LIBRARIES}
  ${Boost_SYSTEM_LIBRARY_RELEASE}
  ${Boost_THREAD_LIBRARY_RELEASE}
  ${Boost_TIMER_LIBRARY_RELEASE}
  )

link_directories(
  ${GAZEBO_LIBRARY_DIRS}
  ${CMAKE_CURRENT_BINARY_DIR}
  )



add_library(ArduCopterIRLockPlugin SHARED src/ArduCopterIRLockPlugin.cc)
target_link_libraries(ArduCopterIRLockPlugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${Boost_SYSTEM_LIBRARY_RELEASE} ${Boost_THREAD_LIBRARY_RELEASE})

add_library(ArduPilotPlugin SHARED src/ArduPilotPlugin.cc)
target_link_libraries(ArduPilotPlugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${Boost_SYSTEM_LIBRARY_RELEASE} ${Boost_THREAD_LIBRARY_RELEASE})

add_library(GimbalSmall2dPlugin SHARED src/GimbalSmall2dPlugin.cc)
target_link_libraries(GimbalSmall2dPlugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${Boost_SYSTEM_LIBRARY_RELEASE} ${Boost_THREAD_LIBRARY_RELEASE})


set(plugins
    ArduPilotPlugin
    ArduCopterIRLockPlugin
    GimbalSmall2dPlugin
  )


#############
## Install ##
#############

set(PLUGIN_PATH ${CMAKE_INSTALL_LIBDIR}/${PROJECT_NAME}/plugins)
set(MODEL_PATH ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/models)
set(RESOURCE_PATH ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME})

file(REMOVE_RECURSE ${PROJECT_SOURCE_DIR}/models/.DS_Store)
file(GLOB models_list LIST_DIRECTORIES true ${PROJECT_SOURCE_DIR}/models/*)

file(REMOVE_RECURSE ${PROJECT_SOURCE_DIR}/worlds/.DS_Store)
file(GLOB worlds_list LIST_DIRECTORIES true ${PROJECT_SOURCE_DIR}/worlds/*)

install(TARGETS ${plugins} DESTINATION ${PLUGIN_PATH})
install(DIRECTORY ${models_list} DESTINATION ${MODEL_PATH})
install(FILES ${worlds_list} DESTINATION ${RESOURCE_PATH}/worlds)

configure_file(src/setup.sh.in "${CMAKE_CURRENT_BINARY_DIR}/setup.sh" @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/setup.sh DESTINATION ${RESOURCE_PATH})


#############
## Testing ##
#############

###############
## Packaging ##
###############

set(CPACK_PACKAGE_NAME ${PROJECT_NAME}-${GAZEBO_MAJOR_VERSION})
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
set(CPACK_PACKAGE_CONTACT pxusers@googlegroups.com)
set(DEBIAN_PACKAGE_DEPENDS "")
set(RPM_PACKAGE_DEPENDS "")

set(CPACK_DEBIAN_PACKAGE_DEPENDS ${DEBIAN_PACKAGE_DEPENDS})
set(CPACK_DEBIAN_PACKAGE_SECTION "devel")
set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "gazebo plugins for ardupilot sitl.")

set(CPACK_RPM_PACKAGE_REQUIRES "${DEBIAN_PACKAGE_DEPENDS}")
set(CPACK_RPM_PACKAGE_DESCRIPTION "Gazebo plugins for ardupilot sitl.")
set(CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}-${GAZEBO_MAJOR_VERSION}-${PROJECT_VERSION}")
set(CPACK_SOURCE_PACKAGE_FILE_NAME "${PROJECT_NAME}-${GAZEBO_MAJOR_VERSION}-${PROJECT_VERSION}")

include(CPack)
# vim: set et fenc= ff=unix sts=0 sw=2 ts=2 :

