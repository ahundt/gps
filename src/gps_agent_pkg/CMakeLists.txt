cmake_minimum_required(VERSION 2.8.6)
project(gps_agent_pkg)

find_package(catkin REQUIRED COMPONENTS
    message_generation
    std_msgs
    sensor_msgs
    # pr2_controller_interface
    # pr2_mechanism_model
    pluginlib
    roscpp
    roslib
    rospy
    # pr2_controllers_msgs
    # control_toolbox
    # realtime_tools
    # orocos_kdl
    geometry_msgs
    tf
    # pr2_controller_manager
)

include_directories(include ${catkin_INCLUDE_DIRS})

find_package(orocos_kdl)
find_package(Eigen3 REQUIRED)
include_directories(${EIGEN3_INCLUDE_DIRS})

# Set the build type.  Options are:
#  Coverage       : w/ debug symbols, w/o optimization, w/ code-coverage
#  Debug          : w/ debug symbols, w/o optimization
#  Release        : w/o debug symbols, w/ optimization
#  RelWithDebInfo : w/ debug symbols, w/ optimization
#  MinSizeRel     : w/o debug symbols, w/ optimization, stripped binaries
set(ROS_BUILD_TYPE RelWithDebInfo)

#rosbuild_init()

#set the default path for built executables to the "bin" directory
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
#set the default path for built libraries to the "lib" directory
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

add_message_files(FILES
    CaffeParams.msg
    ControllerParams.msg
    DataRequest.msg
    DataType.msg
    LinGaussParams.msg
    PositionCommand.msg
    RelaxCommand.msg
    SampleResult.msg
    TfActionCommand.msg
    TfObsData.msg
    TfParams.msg
    TrialCommand.msg

)

#uncomment if you have defined messages
generate_messages(DEPENDENCIES std_msgs)
#rosbuild_genmsg()
#uncomment if you have defined services
#rosbuild_gensrv()

#common commands for building c++ executables and libraries
#rosbuild_add_library(${PROJECT_NAME} src/example.cpp)
#target_link_libraries(${PROJECT_NAME} another_library)
#rosbuild_add_executable(example examples/example.cpp)
#target_link_libraries(example ${PROJECT_NAME})

#Include protobuf
include_directories($ENV{GPS_ROOT_DIR}/build/gps)

#Include boost
#rosbuild_add_boost_directories()
find_package(Boost 1.46.0 COMPONENTS)
if(Boost_FOUND)
    include_directories(${Boost_INCLUDE_DIRS})
endif()

# Enable C++11 if available
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
        message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support, or our tests failed to detect it correctly, not enabling C++11.")
endif()

include_directories(include)

set(DDP_FILES src/robotplugin.cpp
              src/sample.cpp
              src/sensor.cpp
              src/neuralnetwork.cpp
              src/tfcontroller.cpp
              src/controller.cpp
              src/lingausscontroller.cpp
              src/camerasensor.cpp
              src/positioncontroller.cpp
              src/trialcontroller.cpp
              src/encodersensor.cpp
              src/encoderfilter.cpp
	      src/rostopicsensor.cpp
              src/util.cpp)


if(${roscpp_VERSION_MAJOR} EQUAL 1 AND ${roscpp_VERSION_MINOR} LESS 12)
   # pr2_controller_interface is not available to ROS KINETIC (1.12) but is in ROS JADE (1.11)
   # http://rosindex.github.io/p/pr2_controller_interface/#jade
   # http://rosindex.github.io/p/pr2_controller_interface/#kinetic
   #
   # if pr2_controller_interface is available
   # add pr2plugin to the list of DDP_FILES to compile
   list(APPEND DDP_FILES 
              src/pr2plugin.cpp
       )
endif()

#uncomment for test executable
option(BUILD_TEST "Build the test executables" OFF)

# Include Caffe
if (USE_CAFFE)
    # add definitions for the C++ code
    add_definitions(-DUSE_CAFFE)
    if (USE_CAFFE_GPU)
        add_definitions(-DCAFFE_GPU)
        find_package(CUDA 5.5 REQUIRED)
        include_directories(${CUDA_INCLUDE_DIRS})
    else (USE_CAFFE_GPU)
        add_definitions(-DCPU_ONLY)
    endif (USE_CAFFE_GPU)
    # add paths to CAFFE
    link_directories(${CAFFE_LIBRARY_PATH})  # Should be set to /path/to/caffe/build/lib
    include_directories(${CAFFE_INCLUDE_PATH})  # Should be set to /path/to/caffe/distribute/include
    # extra header paths
    include_directories(${CAFFE_EXTRA_PATH})
    # add neural network to DDP controller files
    set(DDP_FILES ${DDP_FILES} src/neuralnetworkcaffe.cpp src/caffenncontroller.cpp)
    # compile Caffe test
    if(BUILD_TEST)
        add_executable(caffe_test src/caffe_test.cpp src/neural_network_caffe.cpp)
        target_link_libraries(caffe_test caffe protobuf)

        # compile image processor node
        add_executable(caffe_img_processor src/img_processor.cpp src/neural_network_caffe.cpp)
        target_link_libraries(caffe_img_processor caffe protobuf)
    endif()
endif (USE_CAFFE)

add_library(gps_agent_lib ${DDP_FILES})
add_dependencies(gps_agent_lib  ${catkin_EXPORTED_TARGETS} ${${PROJECT_NAME}_EXPORTED_TARGETS})

if(BUILD_TEST)
    add_executable(main src/main.cpp)   
    add_executable(kinematic_baseline src/kinematic_baseline.cpp)   
    add_executable(controller_switcher src/controller_switcher.cpp) 
    add_executable(pointcloud_solver src/pointcloud_solver.cpp
                                     src/keypoint_detector.cpp) 
    add_executable(point_head src/point_head.cpp)
    add_executable(torso src/torso.cpp)
endif()

# Include Caffe in controller
if (USE_CAFFE)
    target_link_libraries(gps_agent_lib caffe protobuf)
endif (USE_CAFFE)

#rosbuild_link_boost(gps_agent_lib thread)# smart_ptr)
catkin_package(CATKIN_DEPENDS message_runtime std_msgs)