cmake_minimum_required(VERSION 3.8)
set(CMAKE_CXX_STANDARD 17)
project(fast_lio)

set(CMAKE_BUILD_TYPE Release)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_definitions(-DROOT_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/\")


message("Current CPU archtecture: ${CMAKE_SYSTEM_PROCESSOR}")

if (CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
    include(ProcessorCount)
    ProcessorCount(N)
    message("Processer number:  ${N}")

    if (N GREATER 4)
        add_definitions(-DMP_EN)
        add_definitions(-DMP_PROC_NUM=3)
        message("core for MP: 3")
    elseif (N GREATER 3)
        add_definitions(-DMP_EN)
        add_definitions(-DMP_PROC_NUM=2)
        message("core for MP: 2")
    else ()
        add_definitions(-DMP_PROC_NUM=1)
    endif ()
else ()
    add_definitions(-DMP_PROC_NUM=1)
endif ()

find_package(OpenMP QUIET)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}   ${OpenMP_C_FLAGS}")

# ROS dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(std_msgs REQUIRED)
find_package(std_srvs REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(pcl_ros REQUIRED)
find_package(pcl_conversions REQUIRED)

set(dependencies
        rclcpp
        geometry_msgs
        nav_msgs
        sensor_msgs
        std_msgs
        std_srvs
        visualization_msgs
        pcl_ros
        pcl_conversions
)

# Thirdparty libraries
find_package(Eigen3 REQUIRED)
find_package(PCL REQUIRED COMPONENTS common io)

#message(Eigen: ${EIGEN3_INCLUDE_DIR})
#message(STATUS "PCL: ${PCL_INCLUDE_DIRS}")

find_package(livox_ros_driver2 QUIET)
if (livox_ros_driver2_FOUND)
    message( "livox_ros_driver2 is found")
    list(APPEND dependencies livox_ros_driver2)
    add_definitions(-DUSE_LIVOX)
else ()
    message("livox_ros_driver2 is not found")
endif ()


find_package(rosidl_default_generators REQUIRED)
rosidl_generate_interfaces(${PROJECT_NAME}_msgs
        "msg/Pose6D.msg"
)

# fast_lio library
add_library(${PROJECT_NAME}
        src/ikd_Tree.cpp
        src/preprocess.cpp
)
target_include_directories(${PROJECT_NAME}
        PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include/${PROJECT_NAME}>"
        SYSTEM
        ${EIGEN3_INCLUDE_DIR}
        ${PCL_INCLUDE_DIRS}
)
ament_target_dependencies(${PROJECT_NAME} SYSTEM
        ${dependencies}
)
rosidl_get_typesupport_target(cpp_typesupport_target
        ${PROJECT_NAME}_msgs "rosidl_typesupport_cpp")
target_link_libraries(${PROJECT_NAME} ${cpp_typesupport_target})

install(
        TARGETS ${PROJECT_NAME}
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION lib/${PROJECT_NAME}
)

# laserMapping node
add_executable(fastlio_mapping src/laserMapping.cpp)
target_link_libraries(fastlio_mapping ${PROJECT_NAME})
install(
        TARGETS
        fastlio_mapping
        DESTINATION lib/${PROJECT_NAME}
)

install(
        DIRECTORY config launch rviz
        DESTINATION share/${PROJECT_NAME}
)

ament_package()