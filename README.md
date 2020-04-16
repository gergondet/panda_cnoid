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
- [franka\_gazebo](https://github.com/mkrizmancic/franka_gazebo)

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
