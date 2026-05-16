cmake_minimum_required(VERSION 3.16)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
  message(FATAL_ERROR
    "In-source builds are disabled.\n"
    "Use scripts/build.sh or configure with -B build."
  )
endif()

project(core LANGUAGES C CXX)

set(ONNXRUNTIME_VERSION "1.23.0")
option(BUILD_CORE "Build the core runtime and ROS2 transport" ON)
option(BUILD_SIMULATION "Build the bundled simulation targets" ON)
set(VENDORS_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/vendors")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

function(require_path label path)
  if(NOT EXISTS "${path}")
    message(FATAL_ERROR
      "Missing ${label}: ${path}\n"
      "Run scripts/build.sh before configuring this project."
    )
  endif()
endfunction()

function(find_vendor_library out_var label)
  cmake_parse_arguments(ARG "" "" "NAMES;PATHS" ${ARGN})
  set(_result_var "_${out_var}_RESULT")
  unset(${_result_var})
  unset(${_result_var} CACHE)
  find_library(${_result_var}
    NAMES ${ARG_NAMES}
    PATHS ${ARG_PATHS}
    NO_DEFAULT_PATH
  )
  if(NOT ${_result_var})
    message(FATAL_ERROR
      "Could not find ${label} library.\n"
      "Checked paths: ${ARG_PATHS}\n"
      "Run scripts/build.sh to refresh vendors."
    )
  endif()
  set(${out_var} "${${_result_var}}" PARENT_SCOPE)
endfunction()

function(configure_runtime_paths target)
  cmake_parse_arguments(ARG "" "" "LIBRARY_DIRS" ${ARGN})
  if(NOT TARGET "${target}")
    message(FATAL_ERROR "configure_runtime_paths: target '${target}' does not exist")
  endif()

  set(_library_dirs "")
  foreach(_dir IN LISTS ARG_LIBRARY_DIRS)
    if(_dir AND IS_DIRECTORY "${_dir}")
      list(APPEND _library_dirs "${_dir}")
    endif()
  endforeach()
  list(REMOVE_DUPLICATES _library_dirs)
  if(NOT _library_dirs)
    return()
  endif()

  set_property(TARGET "${target}" APPEND PROPERTY BUILD_RPATH "${_library_dirs}")
  set_property(TARGET "${target}" APPEND PROPERTY INSTALL_RPATH "${_library_dirs}")

  if(UNIX AND NOT APPLE)
    foreach(_dir IN LISTS _library_dirs)
      target_link_options("${target}" PRIVATE "-Wl,-rpath-link,${_dir}")
    endforeach()
  endif()
endfunction()

function(collect_prefix_library_dirs out_var)
  set(_library_dirs "")
  set(_prefixes ${CMAKE_PREFIX_PATH})
  foreach(_env_var AMENT_PREFIX_PATH CMAKE_PREFIX_PATH COLCON_PREFIX_PATH)
    if(DEFINED ENV{${_env_var}} AND NOT "$ENV{${_env_var}}" STREQUAL "")
      string(REPLACE ":" ";" _env_prefixes "$ENV{${_env_var}}")
      list(APPEND _prefixes ${_env_prefixes})
    endif()
  endforeach()

  foreach(_prefix IN LISTS _prefixes)
    if(IS_DIRECTORY "${_prefix}/lib")
      list(APPEND _library_dirs "${_prefix}/lib")
    endif()
  endforeach()
  list(REMOVE_DUPLICATES _library_dirs)
  set(${out_var} "${_library_dirs}" PARENT_SCOPE)
endfunction()

find_package(Threads REQUIRED)
find_package(yaml-cpp REQUIRED)
if(NOT BUILD_CORE AND NOT BUILD_SIMULATION)
  message(FATAL_ERROR "At least one of BUILD_CORE or BUILD_SIMULATION must be ON.")
endif()

if(BUILD_CORE)
  find_package(ament_cmake REQUIRED)
  find_package(rclcpp REQUIRED)
  find_package(CycloneDDS REQUIRED)
  find_package(nav_msgs REQUIRED)
  find_package(rosidl_cmake REQUIRED)
  find_package(rosidl_generator_c REQUIRED)
  find_package(rosidl_generator_cpp REQUIRED)
  collect_prefix_library_dirs(PREFIX_LIBRARY_DIRS)

  set(ONNXRUNTIME_ROOT "${VENDORS_ROOT}/onnxruntime")
  require_path("ONNX Runtime ${ONNXRUNTIME_VERSION} vendor" "${ONNXRUNTIME_ROOT}")
  find_path(ONNXRUNTIME_INCLUDE_DIR
    NAMES onnxruntime_cxx_api.h
    PATHS "${ONNXRUNTIME_ROOT}/include"
    NO_DEFAULT_PATH
    REQUIRED
  )
  find_vendor_library(ONNXRUNTIME_LIBRARY "ONNX Runtime"
    NAMES onnxruntime libonnxruntime.so.${ONNXRUNTIME_VERSION} libonnxruntime.so.1 libonnxruntime.so
    PATHS "${ONNXRUNTIME_ROOT}/lib" "${ONNXRUNTIME_ROOT}/bin"
  )
  get_filename_component(ONNXRUNTIME_LIBRARY_DIR "${ONNXRUNTIME_LIBRARY}" DIRECTORY)

  find_path(FX_CLIENT_INCLUDE_DIR
    NAMES fx_client.h
    PATHS
      "${CMAKE_CURRENT_SOURCE_DIR}/vendors/fx_client_sim/include"
      "${CMAKE_CURRENT_SOURCE_DIR}/vendors/fx_client_sim"
      "${CMAKE_CURRENT_SOURCE_DIR}/fx_client_sim/include"
      "${CMAKE_CURRENT_SOURCE_DIR}/fx_client_sim"
      "${CMAKE_CURRENT_SOURCE_DIR}/fx_client/include"
      "${CMAKE_CURRENT_SOURCE_DIR}/fx_client"
    NO_DEFAULT_PATH
  )
  if(NOT FX_CLIENT_INCLUDE_DIR)
    message(FATAL_ERROR
      "Could not find fx_client.h under fx_client_sim or fx_client."
    )
  endif()

  set(FX_CLIENT_SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/vendors/fx_client_sim/src/fx_client.cpp")
  if(NOT EXISTS "${FX_CLIENT_SOURCE}")
    message(FATAL_ERROR
      "Could not find fx_client implementation: ${FX_CLIENT_SOURCE}"
    )
  endif()

  add_library(fx_client STATIC "${FX_CLIENT_SOURCE}")
  target_include_directories(fx_client
    PUBLIC
      ${FX_CLIENT_INCLUDE_DIR}
  )
  target_link_libraries(fx_client PUBLIC Threads::Threads)

  set(F4_CORE_MESSAGE_FILES
    "include/comm/ros2/msg/EventCore.msg"
    "include/comm/ros2/msg/EventUser.msg"
    "include/comm/ros2/msg/CommandCore.msg"
    "include/comm/ros2/msg/CommandUser.msg"
    "include/comm/ros2/msg/RobotReport.msg"
    "include/comm/ros2/msg/CommandFilter.msg"
    "include/comm/ros2/msg/JointSafety.msg"
    "include/comm/ros2/srv/DynamicParam.srv"
  )

  rosidl_generate_interfaces(${PROJECT_NAME}
    ${F4_CORE_MESSAGE_FILES}
    DEPENDENCIES nav_msgs
  )

  set(F4_CORE_DDS_IDL_FILES
    "${CMAKE_CURRENT_SOURCE_DIR}/include/comm/dds/idl/HeightMap.idl"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/comm/dds/idl/UserBus.idl"
  )

  idlc_generate(
    TARGET f4_core_dds_types
    FILES ${F4_CORE_DDS_IDL_FILES}
  )

  set(F4_CORE_SOURCES
    src/comm/dds/api.cpp
    src/comm/ros2/api.cpp
    src/controller/custom_drive_controller.cpp
    src/controller/flat_drive_controller.cpp
    src/controller/freeze_controller.cpp
    src/controller/idle_controller.cpp
    src/controller/lie_controller.cpp
    src/controller/ready_controller.cpp
    src/controller/rough_drive_controller.cpp
    src/controller/sit_controller.cpp
    src/controller/stand_controller.cpp
    src/controller/user_drive_controller.cpp
    src/core/command_iface.cpp
    src/core/fsm.cpp
    src/core/runner.cpp
    src/core/safety.cpp
    src/core/user_iface.cpp
    src/hardware/joystick.cpp
    src/hardware/mcu.cpp
    src/hardware/robot.cpp
    src/param/param_manager.cpp
    src/param/param_checker.cpp
    src/utils/logger.cpp
    src/utils/onnxpolicy.cpp
    src/utils/rl.cpp 
    src/utils/timer.cpp
  )

  add_executable(f4_core_lib
    src/main.cpp
    ${F4_CORE_SOURCES}
  )

  target_include_directories(f4_core_lib
    PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
      $<INSTALL_INTERFACE:include>
      ${ONNXRUNTIME_INCLUDE_DIR}
  )
  target_link_libraries(f4_core_lib
    fx_client
    yaml-cpp
    ${ONNXRUNTIME_LIBRARY}
    Threads::Threads
    f4_core_dds_types
    CycloneDDS::ddsc
  )
  target_link_directories(f4_core_lib PRIVATE ${PREFIX_LIBRARY_DIRS} ${CMAKE_CURRENT_BINARY_DIR})
  configure_runtime_paths(
    f4_core_lib
    LIBRARY_DIRS
      ${ONNXRUNTIME_LIBRARY_DIR}
      ${CMAKE_CURRENT_BINARY_DIR}
  )
  ament_target_dependencies(f4_core_lib rclcpp nav_msgs)

  rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")
  target_link_libraries(f4_core_lib "${cpp_typesupport_target}")

  install(TARGETS f4_core_lib
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
  )

  ament_export_dependencies(rosidl_default_runtime nav_msgs)
  ament_package()
endif()

if(BUILD_SIMULATION)
  add_subdirectory(vendors/simulation)

  add_custom_target(
    simulation_link ALL
    COMMAND ${CMAKE_COMMAND} -E create_symlink
      $<TARGET_FILE:simulation>
      ${CMAKE_BINARY_DIR}/simulation
    COMMAND ${CMAKE_COMMAND} -E create_symlink
      $<TARGET_FILE:simulation_runtime>
      ${CMAKE_BINARY_DIR}/simulation_runtime
    DEPENDS simulation simulation_runtime
  )
endif()
