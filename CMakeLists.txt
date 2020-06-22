cmake_minimum_required(VERSION 3.1)

# Disable in-source builds
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "This project doesn't support in-source builds")
endif()

project(panda_cnoid)

find_package(mc_rtc REQUIRED)

find_package(choreonoid QUIET)

find_description_package(franka_gazebo REQUIRED)

if(choreonoid_FOUND)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(OPENHRP REQUIRED openhrp3.1)
  pkg_check_modules(HRPSYS_BASE REQUIRED hrpsys-base)
endif()

find_program(XACRO xacro)
if("${XACRO}" STREQUAL "XACRO-NOTFOUND")
  message(FATAL_ERROR "Could not find xacro in your environment")
endif()

set(MODEL_DESTINATION "${OPENHRP_PREFIX}/share/OpenHRP-3.1/robot/panda")

if(choreonoid_FOUND)
  # Install the VRML model
  install(DIRECTORY model/panda/
    DESTINATION ${MODEL_DESTINATION}
    FILES_MATCHING PATTERN "*.wrl")
endif()

set(URDF_DESTINATION "${CMAKE_INSTALL_PREFIX}/share/panda_cnoid/")
set(CONVEX_DESTINATION "${URDF_DESTINATION}/convex")
set(MESHES_DESTINATION "${URDF_DESTINATION}/meshes/")

macro(GENERATE_ROBOT XACRO_IN ROBOT_IN)
  # Generate a xacro file and the associated urdf
  get_filename_component(XACRO_OUT "${XACRO_IN}" NAME)
  set(XACRO_OUT "${CMAKE_BINARY_DIR}/xacro/${XACRO_OUT}")
  configure_file("${XACRO_IN}" "${XACRO_OUT}" @ONLY)
  get_filename_component(NAME "${XACRO_OUT}" NAME_WE)
  set(URDF_OUT "${CMAKE_BINARY_DIR}/urdf/${NAME}.urdf")
  set(URDF_DEPENDS "${XACRO_OUT}")
  set(EXTRA_DEPENDS "${ARGN}")
  foreach(EXTRA IN LISTS EXTRA_DEPENDS)
    list(APPEND URDF_DEPENDS "${EXTRA}")
  endforeach()
  add_custom_command(OUTPUT "${URDF_OUT}" COMMAND ${XACRO} -i "${XACRO_OUT}" -o "${URDF_OUT}" DEPENDS ${URDF_DEPENDS})
  add_custom_target(generate_${NAME}_urdf ALL DEPENDS ${URDF_OUT})
  install(FILES "${URDF_OUT}" DESTINATION "${URDF_DESTINATION}")
  # Create and install a YAML RobotModule
  set(MODULE_IN "module/${ROBOT_IN}.yaml")
  set(MODULE_OUT "${CMAKE_BINARY_DIR}/${MODULE_IN}")
  configure_file(${MODULE_IN} "${MODULE_OUT}")
  install(FILES "${MODULE_OUT}" DESTINATION "${URDF_DESTINATION}")
  # Create and install an aliase
  set(ALIASE_IN "module/aliase/${ROBOT_IN}.yaml")
  set(ALIASE_OUT "${CMAKE_BINARY_DIR}/${ALIASE_IN}")
  configure_file(${ALIASE_IN} "${ALIASE_OUT}")
  install(FILES "${ALIASE_OUT}" DESTINATION "${MC_ROBOTS_ALIASES_DIRECTORY}")
endmacro()

configure_file("xacro/main.xacro" "${CMAKE_BINARY_DIR}/xacro/main.xacro" @ONLY)
configure_file("xacro/main.gazebo.xacro" "${CMAKE_BINARY_DIR}/xacro/main.gazebo.xacro" @ONLY)
configure_file("xacro/pandaA_arm.xacro" "${CMAKE_BINARY_DIR}/xacro/pandaA_arm.xacro" @ONLY)
configure_file("xacro/pandaA_arm.gazebo.xacro" "${CMAKE_BINARY_DIR}/xacro/pandaA_arm.gazebo.xacro" @ONLY)
configure_file("xacro/pandaB_arm.xacro" "${CMAKE_BINARY_DIR}/xacro/pandaB_arm.xacro" @ONLY)
configure_file("xacro/pandaB_arm.gazebo.xacro" "${CMAKE_BINARY_DIR}/xacro/pandaB_arm.gazebo.xacro" @ONLY)
configure_file("xacro/foot.xacro" "${CMAKE_BINARY_DIR}/xacro/foot.xacro" @ONLY)
install(FILES xacro/meshes/foot.stl DESTINATION "${MESHES_DESTINATION}")

# panda robot without any end-effector
#GENERATE_ROBOT("${FRANKA_GAZEBO_PATH}/robots/panda_arm.urdf.xacro" "panda_default")
GENERATE_ROBOT("xacro/panda_arm_default.urdf.xacro" "panda_default")
# panda robot with a two-finger gripper "hand" as end-effector
#GENERATE_ROBOT("${FRANKA_GAZEBO_PATH}/robots/panda_arm_hand.urdf.xacro" "panda_hand")
GENERATE_ROBOT("xacro/panda_arm_hand.urdf.xacro" "panda_hand")
# panda robot with a flat "foot" as end-effector
GENERATE_ROBOT("xacro/panda_arm_foot.urdf.xacro" "panda_foot") # "${CMAKE_BINARY_DIR}/xacro/foot.xacro"
# panda robot with a suction cup "pump" as end-effector
# GENERATE_ROBOT("xacro/panda_arm_pump.urdf.xacro" "panda_pump") 
# two panda robots without any end-effector
GENERATE_ROBOT("xacro/doublepanda_arm_default.urdf.xacro" "doublepanda_default")

# Install convex

install(DIRECTORY convex DESTINATION "${URDF_DESTINATION}" FILES_MATCHING PATTERN "*-ch.txt")

# Install rsdf

install(DIRECTORY rsdf DESTINATION "${URDF_DESTINATION}" FILES_MATCHING PATTERN "*.rsdf")

macro(GENERATE_CNOID_PROJECT PROJECT_FOLDER WRL_NAME)
  set(PROJECT_DESTINATION "${HRPSYS_BASE_PREFIX}/share/hrpsys/samples/${PROJECT_FOLDER}")
  set(MODEL_OUT "${MODEL_DESTINATION}/${WRL_NAME}.wrl")
  file(READ cnoid/${PROJECT_FOLDER}/joint_positions.cnoid JOINT_POSITIONS)
  string(STRIP "${JOINT_POSITIONS}" JOINT_POSITIONS)
  set(USE_UDP False)
  set(SIM_SCRIPT sim_mc.py)
  set(ROBOT_NAME "${PROJECT_FOLDER}")
  set(SENSOR_PORT_A "LeftHandForceSensor")
  set(SENSOR_PORT_B "RightHandForceSensor")
  set(CONF_FILE_NAME "Virtual-${PROJECT_FOLDER}-RTC.conf")
  install(FILES "cnoid/${PROJECT_FOLDER}/Virtual-${PROJECT_FOLDER}-RTC.conf" DESTINATION "${CHOREONOID_PLUGIN_DIR}/rtc/")
  set(PROJECT_FILES
    cnoid/common/clear-omninames.sh
    cnoid/common/robot.conf
    cnoid/common/rtc.conf.choreonoid
    cnoid/common/sim_mc.cnoid
    cnoid/${PROJECT_FOLDER}/sim_mc.py
    cnoid/${PROJECT_FOLDER}/PDcontroller.conf.choreonoid
    cnoid/${PROJECT_FOLDER}/PDgains_sim.dat
  )
  foreach(F ${PROJECT_FILES})
    get_filename_component(F_NAME "${F}" NAME)
    configure_file(${F} "${CMAKE_BINARY_DIR}/cnoid/${PROJECT_FOLDER}/${F_NAME}" @ONLY)
  endforeach()
  set(USE_UDP True)
  set(SIM_SCRIPT sim_mc_udp.py)
  set(SENSOR_PORT "lhsensor")
  configure_file(cnoid/${PROJECT_FOLDER}/sim_mc.py "${CMAKE_BINARY_DIR}/cnoid/${PROJECT_FOLDER}/${SIM_SCRIPT}" @ONLY)
  configure_file(cnoid/common/sim_mc.cnoid "${CMAKE_BINARY_DIR}/cnoid/${PROJECT_FOLDER}/sim_mc_udp.cnoid" @ONLY)
  install(DIRECTORY "${CMAKE_BINARY_DIR}/cnoid/${PROJECT_FOLDER}/" DESTINATION "${PROJECT_DESTINATION}")
  install(PROGRAMS cnoid/common/clear-omninames.sh DESTINATION "${PROJECT_DESTINATION}")
endmacro()

if(choreonoid_FOUND)
  GENERATE_CNOID_PROJECT(panda_default panda_arm_default)
  GENERATE_CNOID_PROJECT(panda_hand panda_arm_hand)
  GENERATE_CNOID_PROJECT(panda_foot panda_arm_foot)
  #GENERATE_CNOID_PROJECT(panda_pump panda_arm_pump)
  GENERATE_CNOID_PROJECT(doublepanda_default doublepanda_arm_default)
endif()
