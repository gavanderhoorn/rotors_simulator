Fork of RotorS
==============

This is a fork of [RotorS](https://github.com/ethz-asl/rotors_simulator) which retains just the wind Gazebo plugin.

Only build-tested on Kinetic and Melodic.

To build in a Catkin workspace:

```bash
# check missing dependencies
rosdep update
rosdep check --from-paths /path/to/your/catkin_ws/src -i

# if something is missing:
rosdep install --from-paths /path/to/your/catkin_ws/src -i

# now build
cd /path/to/your/catkin_ws
catkin build

# activate
source /path/to/your/catkin_ws/devel/setup.bash
```
