cmake_minimum_required(VERSION 3.16)
project(mujoco_ros2_control)

# Set default C standards
if(NOT CMAKE_C_STANDARD)
    set(CMAKE_C_STANDARD 99)
endif()

# Set default C++ standards
if(NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 17)
endif()

# Enable debug build
set(CMAKE_BUILD_TYPE Release)

# Export compile commands for IDEs
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

# Set OpenGL preferences
set(OpenGL_GL_PREFERENCE LEGACY)

# Package dependencies
set(THIS_PACKAGE_INCLUDE_DEPENDS
    ament_cmake
    rclcpp
    std_msgs
    geometry_msgs
    rosgraph_msgs
    controller_manager
    hardware_interface
    urdf
    ament_lint_auto
    realtime_tools
    OpenCV
    PCL
    pcl_conversions
    cv_bridge
    generate_parameter_library
    Eigen3
    tf2_ros
    glfw3
)

foreach(Dependency IN ITEMS ${THIS_PACKAGE_INCLUDE_DEPENDS})
    find_package(${Dependency} REQUIRED)
endforeach()

# Set Eigen3
set(EIGEN3_FOUND TRUE)
set(EIGEN3_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR})

# Common variables to reduce repetition (conservative approach)
set(COMMON_TARGET_PROPERTIES POSITION_INDEPENDENT_CODE ON)
set(COMMON_PROJECT_INCLUDE_DIR $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>)

##################################
## MUJOCO ########################
##################################
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

find_package(mujoco QUIET)
if(mujoco_FOUND)
  message(STATUS "Mujoco build from source has been found")
  set(MUJOCO_LIB mujoco::mujoco)
  set(MUJOCO_INCLUDE_DIR ${MUJOCO_INCLUDE_DIR})
  set(MUJOCO_SIMULATE_DIR ${MUJOCO_INCLUDE_DIR}/../simulate)
elseif(DEFINED ENV{MUJOCO_DIR})
  message(STATUS "Mujoco build from source has not been found. Attempting to find the binary in $ENV{MUJOCO_DIR} instead.")
  find_library(MUJOCO_LIB mujoco HINTS $ENV{MUJOCO_DIR}/lib)
  if(NOT MUJOCO_LIB)
    message(FATAL_ERROR "Failed to find binary in $ENV{MUJOCO_DIR}")
  endif()
  set(MUJOCO_INCLUDE_DIR $ENV{MUJOCO_DIR}/include)
  set(MUJOCO_SIMULATE_DIR $ENV{MUJOCO_DIR}/simulate)
else()
  message(FATAL_ERROR "Failed to find mujoco with find_package.
  Either build and install mujoco from source or set the MUJOCO_DIR environment variable to tell CMake where to find the binary install. ")
endif()

# Fetch lodepng dependency.
if(NOT TARGET lodepng)
  include(FetchContent)
  FetchContent_Declare(
    lodepng
    GIT_REPOSITORY https://github.com/lvandeve/lodepng.git
    GIT_TAG ${MUJOCO_DEP_VERSION_lodepng}
  )

  FetchContent_GetProperties(lodepng)
  if(NOT lodepng_POPULATED)
    FetchContent_Populate(lodepng)
    # This is not a CMake project.
    set(LODEPNG_SRCS ${lodepng_SOURCE_DIR}/lodepng.cpp)
    set(LODEPNG_HEADERS ${lodepng_SOURCE_DIR}/lodepng.h)
    add_library(lodepng STATIC ${LODEPNG_HEADERS} ${LODEPNG_SRCS})
    set_target_properties(lodepng PROPERTIES POSITION_INDEPENDENT_CODE ON)
    target_compile_options(lodepng PRIVATE ${MUJOCO_MACOS_COMPILE_OPTIONS})
    target_link_options(lodepng PRIVATE ${MUJOCO_MACOS_LINK_OPTIONS})
    target_include_directories(lodepng PUBLIC ${lodepng_SOURCE_DIR})
  endif()
endif()

# We don't care about warnings from the external sources
set(MUJOCO_SILENCE_COMPILER_WARNINGS
  -Wno-missing-field-initializers
  -Wno-unused-parameter
  -Wno-sign-compare
  -Wno-psabi
)

# Build Mujoco's simulate application and make it available for linking
add_library(platform_ui_adapter OBJECT)
set_target_properties(platform_ui_adapter PROPERTIES ${COMMON_TARGET_PROPERTIES})
target_sources(
  platform_ui_adapter
  PUBLIC ${MUJOCO_SIMULATE_DIR}/glfw_adapter.h ${MUJOCO_SIMULATE_DIR}/glfw_dispatch.h ${MUJOCO_SIMULATE_DIR}/platform_ui_adapter.h
  PRIVATE ${MUJOCO_SIMULATE_DIR}/glfw_adapter.cc ${MUJOCO_SIMULATE_DIR}/glfw_dispatch.cc ${MUJOCO_SIMULATE_DIR}/platform_ui_adapter.cc
)
target_include_directories(
  platform_ui_adapter PUBLIC ${MUJOCO_SIMULATE_DIR} ${MUJOCO_INCLUDE_DIR}
                             $<TARGET_PROPERTY:glfw,INTERFACE_INCLUDE_DIRECTORIES>
)
target_link_libraries(platform_ui_adapter PUBLIC ${MUJOCO_LIB})
target_compile_options(platform_ui_adapter PRIVATE ${MUJOCO_SILENCE_COMPILER_WARNINGS})
add_library(mujoco::platform_ui_adapter ALIAS platform_ui_adapter)

add_library(libsimulate SHARED $<TARGET_OBJECTS:platform_ui_adapter>)
add_library(mujoco::libsimulate ALIAS libsimulate)
set_target_properties(libsimulate PROPERTIES
  OUTPUT_NAME simulate
  ${COMMON_TARGET_PROPERTIES}
)
target_sources(
  libsimulate
  PUBLIC ${MUJOCO_SIMULATE_DIR}/simulate.h
  PRIVATE ${MUJOCO_SIMULATE_DIR}/simulate.cc ${MUJOCO_SIMULATE_DIR}/array_safety.h
)
target_include_directories(libsimulate PUBLIC ${MUJOCO_SIMULATE_DIR} ${MUJOCO_INCLUDE_DIR})
target_link_libraries(libsimulate PUBLIC lodepng mujoco::platform_ui_adapter ${MUJOCO_LIB})
target_compile_options(libsimulate PRIVATE ${MUJOCO_SILENCE_COMPILER_WARNINGS})
#################################################


# Generate parameter libraries
generate_parameter_library(${PROJECT_NAME}_parameters
    src/${PROJECT_NAME}_parameters.yaml
)

generate_parameter_library(mujoco_rgbd_camera_parameters
    src/mujoco_rgbd_camera_parameters.yaml
)

# Add system plugin (before main executable)
add_library(mujoco_system_plugins SHARED src/mujoco_system.cpp)
ament_target_dependencies(mujoco_system_plugins
    hardware_interface
    rclcpp
    urdf
)
target_include_directories(mujoco_system_plugins PUBLIC ${COMMON_PROJECT_INCLUDE_DIR})
# Mark these as system packages to prevent compiler warnings
target_include_directories(mujoco_system_plugins SYSTEM PUBLIC
  ${MUJOCO_INCLUDE_DIR}
)

# Add GUI simulation library (before main executable)
add_library(simulate_gui SHARED src/simulate_gui.cpp)
target_link_libraries(simulate_gui PUBLIC glfw)
target_link_libraries(simulate_gui PRIVATE
    ${MUJOCO_LIB}
    libsimulate
    lodepng
)
# Mark these as system packages to prevent compiler warnings
target_include_directories(simulate_gui SYSTEM PUBLIC
  ${MUJOCO_SIMULATE_DIR}
)
target_include_directories(simulate_gui PUBLIC ${COMMON_PROJECT_INCLUDE_DIR})

# Add executable for the main ROS2 control node
add_executable(${PROJECT_NAME}
    src/mujoco_ros2_control_plugin.cpp
    src/mujoco_depth_camera.cpp
    src/imu_sensor.cpp
    src/pose_sensor.cpp
    src/wrench_sensor.cpp
    src/mujoco_ros2_sensors.cpp
)

ament_target_dependencies(${PROJECT_NAME}
    controller_manager
    hardware_interface
    rclcpp
    urdf
    realtime_tools
    cv_bridge
    PCL
    pcl_conversions
    tf2_ros
)

# Link libraries to the main executable
target_link_libraries(${PROJECT_NAME}
    ${MUJOCO_LIB}
    glfw
    ${PCL_LIBRARIES}
    ${OpenCV_LIBS}
    ${PROJECT_NAME}_parameters
    mujoco_rgbd_camera_parameters
)
# Mark these as system packages to prevent compiler warnings
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC
  ${MUJOCO_INCLUDE_DIR}
  ${MUJOCO_SIMULATE_DIR}
)
# Add compile definition to enable USE_LIBSIMULATE in the main executable
target_compile_definitions(${PROJECT_NAME} PRIVATE USE_LIBSIMULATE)

# Link simulate_gui to the main project
target_link_libraries(${PROJECT_NAME} simulate_gui)

# Install targets (consolidated)
install(TARGETS
    simulate_gui
    mujoco_system_plugins
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

install(TARGETS ${PROJECT_NAME} DESTINATION lib/${PROJECT_NAME})

# Install MJCF models
install(DIRECTORY mjcf DESTINATION share/${PROJECT_NAME}/)

# Install Python scripts
ament_python_install_package(scripts)
install(PROGRAMS
    scripts/xacro2mjcf.py
    scripts/urdf2mjcf.py
    DESTINATION lib/${PROJECT_NAME}
)

# Export dependencies
ament_export_include_directories(include)
ament_export_dependencies(ament_cmake)
ament_export_dependencies(rclcpp)
ament_export_libraries(
    ${PROJECT_NAME}
    mujoco_system_plugins
)

pluginlib_export_plugin_description_file(mujoco_ros2_control mujoco_system_plugins.xml)

# Finalize package
ament_package()

#############
## Testing ##
#############
if(BUILD_TESTING)
    ament_lint_auto_find_test_dependencies()
    find_package(ament_cmake_ros REQUIRED)
    find_package(launch_testing_ament_cmake REQUIRED)
    install(DIRECTORY test/data/ DESTINATION share/${PROJECT_NAME}/test_data)

    function(add_ros_isolated_launch_test path)
        set(RUNNER "${ament_cmake_ros_DIR}/run_test_isolated.py")
        add_launch_test("${path}" RUNNER "${RUNNER}" ${ARGN})
    endfunction()

    add_ros_isolated_launch_test(test/simple_launch.test.py)
endif()

#############
## Doxygen ##
#############
find_package(Doxygen)
if(BUILD_DOC AND DOXYGEN_FOUND)
    set(doxyfile_in ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in)
    set(doxyfile ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)
    configure_file(${doxyfile_in} ${doxyfile} @ONLY)
    message("Doxygen build started.")
    add_custom_target(doc ALL
        COMMAND ${DOXYGEN_EXECUTABLE} ${doxyfile}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Generating API documentation with Doxygen"
        VERBATIM)
    install(DIRECTORY doc/images DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/doc)
else()
    if(NOT DOXYGEN_FOUND)
        message("Doxygen needs to be installed to generate the documentation")
    endif()
endif()
