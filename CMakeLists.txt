cmake_minimum_required(VERSION 3.1)

# Disable in-source builds
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "This project doesn't support in-source builds")
endif()

project(panda_cnoid)

find_package(mc_rtc REQUIRED)

find_package(choreonoid REQUIRED)

find_description_package(franka_gazebo REQUIRED)

find_package(PkgConfig REQUIRED)
pkg_check_modules(OPENHRP REQUIRED openhrp3.1)
pkg_check_modules(HRPSYS_BASE REQUIRED hrpsys-base)

find_program(XACRO xacro)
if("${XACRO}" STREQUAL "XACRO-NOTFOUND")
  message(FATAL_ERROR "Could not find xacro in your environment")
endif()

set(MODEL_DESTINATION "${OPENHRP_PREFIX}/share/OpenHRP-3.1/robot/panda")

# Install the VRML model
install(DIRECTORY model/panda/
  DESTINATION ${MODEL_DESTINATION}
  FILES_MATCHING PATTERN "*.wrl")

# Generate the URDF of panda
set(XACRO_IN "${FRANKA_GAZEBO_PATH}/robots/panda_arm_hand.urdf.xacro")
set(URDF_OUT "${CMAKE_BINARY_DIR}/urdf/panda_arm_hand.urdf")
add_custom_command(OUTPUT ${URDF_OUT}
  COMMAND ${XACRO} -i "${XACRO_IN}" -o "${URDF_OUT}")
add_custom_target(generate_urdf ALL DEPENDS "${URDF_OUT}")
set(URDF_DESTINATION "${CMAKE_INSTALL_PREFIX}/share/panda_cnoid/")
install(FILES "${URDF_OUT}" DESTINATION "${URDF_DESTINATION}")

# Create and install a YAML RobotModule
configure_file(module/panda.yaml "${CMAKE_BINARY_DIR}/module/panda.yaml")
install(FILES "${CMAKE_BINARY_DIR}/module/panda.yaml" DESTINATION "${URDF_DESTINATION}")

# Create and install an aliase
configure_file(module/aliase/panda.yaml "${CMAKE_BINARY_DIR}/module/aliase/panda.yaml")
install(FILES "${CMAKE_BINARY_DIR}/module/aliase/panda.yaml" DESTINATION "${MC_ROBOTS_ALIASES_DIRECTORY}")

# Configure and install a choreonoid project
install(FILES "cnoid/panda/Virtual-panda-RTC.conf" DESTINATION "${CHOREONOID_PLUGIN_DIR}/rtc/")
set(PROJECT_DESTINATION "${HRPSYS_BASE_PREFIX}/share/hrpsys/samples/panda")
set(USE_UDP False)
set(SIM_SCRIPT sim_mc.py)
set(PROJECT_FILES
  PDcontroller.conf.choreonoid
  PDgains_sim.dat
  robot.conf
  rtc.conf.choreonoid
  sim_mc.cnoid
  sim_mc.py
)
foreach(F ${PROJECT_FILES})
  configure_file(cnoid/panda/${F} "${CMAKE_BINARY_DIR}/cnoid/panda/${F}" @ONLY)
endforeach()
set(USE_UDP True)
set(SIM_SCRIPT sim_mc_udp.py)
configure_file(cnoid/panda/sim_mc.py "${CMAKE_BINARY_DIR}/cnoid/panda/${SIM_SCRIPT}" @ONLY)
configure_file(cnoid/panda/sim_mc.cnoid "${CMAKE_BINARY_DIR}/cnoid/panda/sim_mc_udp.cnoid" @ONLY)
install(DIRECTORY "${CMAKE_BINARY_DIR}/cnoid/panda/" DESTINATION "${PROJECT_DESTINATION}")
install(PROGRAMS cnoid/panda/clear-omninames.sh DESTINATION "${PROJECT_DESTINATION}")
