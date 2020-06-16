panda\_cnoid
==

This repository contains the necessary VRML and cnoid project file to run panda simulation with Choreonoid and mc\_openrtm.

The project takes care of:
- installing the VRML model of the panda robot
- generate an URDF file from the panda xacro file
- install a JSON RobotModule for mc\_rtc
- install a robot alias for mc\_rtc
- install a Choreonoid project suitable for mc\_openrtm control

The URDF model for Panda was taken from [franka\_gazebo](https://github.com/mkrizmancic/franka_gazebo) which is a dependency of this project. The VRML model was converted using [simtrans](https://github.com/fkanehiro/simtrans) and restoring joint axis to the URDF ones.

Dependencies
--

- [mc\_openrtm](https://gite.lirmm.fr/multi-contact/mc_openrtm)
- [franka_gazebo]

Install
--

```bash
mkdir build
cd build
cmake ../
make
sudo make install
```

Usage
--

Edit your mc\_rtc configuration so that `MainRobot` is `panda`

Adding a new panda variant
--

1. Create a new xacro file for your panda extension

An example is given in [xacro/panda_arm_foot.urdf.xacro](xacro/panda_arm_foot.urdf.xacro). In that particular case we are putting a specific end-effector on the panda arm so the xacro file itself includes the panda arm xacro file for [franka_gazebo].

Things to note:
- that particular file uses an extra xacro file ([xacro/foot.xacro](xacro/foot.xacro)), we will see how to handle this in the next step;
- we use CMake specific variables (e.g. `@CMAKE_BINARY_DIR@` or `@MESHES_DESTINATION@`) in those file because we are not building a catkin package.

2. Create an mc\_rtc YAML RobotModule and aliase

Simply copy the existing models in the [repository](model/).

3. Append the relevant data to `CMakeLists.txt`

This will ensure that the URDF model is generated from your xacro file and that the robot model and aliases are generated.

Here is the relevant snippet for `panda_arm_foot`

```cmake
configure_file("xacro/foot.xacro" "${CMAKE_BINARY_DIR}/xacro/foot.xacro" @ONLY)
install(FILES xacro/meshes/foot.stl DESTINATION "${MESHES_DESTINATION}")
GENERATE_ROBOT("xacro/panda_arm_foot.urdf.xacro" "panda_foot" "${CMAKE_BINARY_DIR}/xacro/foot.xacro")
```

The first two lines take care of generating the intermediary xacro file we need and to install the extra mesh we have added.

The last line is generating the robot URDF and the relevant files for mc\_rtc. The first two arguments are mandatory, they provide the main xacro file used to generate the urdf and the robot name. Extra arguments are provided to specify extra dependencies on the URDF generation.

4. Generate a VRML model

This requires [simtrans] and a generated URDF file.

Go into the `model/panda` folder and run the following:

```bash
simtrans -i /path/to/your/panda.urdf -o panda_arm_myvariant.wrl
```

You then need to edit `panda_arm_myvariant.wrl` to:

- modify `jointId` entries so that panda\_joint1 has id 0, panda\_joint2 has id 1 and so-on (up-to panda\_joint7 or more if your own model has extra joints)
- modify `jointAxis` to `0.0 0.0 1.0` for every joint from panda\_joint1 to panda\_joint7 as simtrans seems to get those wrong (if your model includes extra joints you might need to check whether the axis are correctly defined by yourself)

You can check that your robot is displayed correctly by running `choreonoid panda_arm_myvariant_project.cnoid`. You can remove this file and `panda_arm_myvariant_project.xml` afterwards as those files are not required.

5. Generate a cnoid project

First copy `cnoid/panda_foot` into `cnoid/panda_myvariant`. If you robot does not have extra joints that's probably enough.

If needed, here are the three files:
- `cnoid/panda_default/joint_positions.cnoid`: the initial joint configuration for your robot; *DO NOT CHANGE THE INDENT IN THIS FILE*
- `cnoid/panda_default/PDcontroller.conf.choreonoid`: configuration file for the PDcontroller component, you only need to edit `pdcontrol_tlimitratio` to match your number of joints (including mimics)
- `cnoid/panda_default/PDgains_sim.dat`: PD gains for the PDcontroller component; there should be one entry per joint

Once these files are correct, you can append the following to `CMakeLists.txt`:

```cmake
GENERATE_CNOID_PROJECT(panda_myvariant panda_arm_myvariant)
```

[franka_gazebo]: https://github.com/mkrizmancic/franka_gazebo
[simtrans]: https://github.com/fkanehiro/simtrans
