# Antropi Robotics Assignment: TurtleBot3 Autonomous Navigation

TurtleBot3 (waffle) navigating autonomously around obstacles in Gazebo using the ROS 2 Nav2 stack.
No custom navigation algorithms: everything uses the standard TurtleBot3 and Nav2 packages.

**Environment:** Ubuntu 24.04 · ROS 2 Jazzy · Gazebo Harmonic

**Demo video:** <PASTE LINK HERE>

---

## 1. Workspace layout

```
antropi_ws/
└── src/tb3_navigation/
    ├── launch/    turtlebot3_world, robot_state_publisher, spawn_turtlebot3, navigation2
    ├── urdf/      turtlebot3_waffle.urdf
    ├── worlds/    turtlebot3_world.world
    ├── models/    turtlebot3_waffle, turtlebot3_common, turtlebot3_world
    ├── meshes/    robot meshes used by the URDF
    ├── bridge/    ROS-Gazebo bridge config
    ├── param/     Nav2 parameters (waffle.yaml)
    ├── map/       pre-built map of turtlebot3_world
    └── rviz/      RViz config
```

`tb3_navigation` is one package assembled from the waffle-specific files of `turtlebot3_description`,
`turtlebot3_gazebo` and `turtlebot3_navigation2` (ROBOTIS, Apache-2.0), with package references
rewritten to point at this package.

## 2. Install

```bash
sudo apt install -y ros-jazzy-navigation2 ros-jazzy-nav2-bringup ros-jazzy-ros-gz \
  ros-jazzy-robot-state-publisher ros-jazzy-rviz2 python3-colcon-common-extensions

git clone https://github.com/hetos10/antropi_ws.git
cd antropi_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install
source install/setup.bash
```

Optional, so every new terminal is ready:

```bash
echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
echo 'source ~/antropi_ws/install/setup.bash' >> ~/.bashrc
```

`TURTLEBOT3_MODEL` defaults to `waffle` in the launch files, so no export is needed.

## 3. Run

**Terminal 1: simulation** (press the Play button in Gazebo if it starts paused)

```bash
ros2 launch tb3_navigation turtlebot3_world.launch.py
```

**Terminal 2: localization + navigation + RViz**

```bash
ros2 launch tb3_navigation navigation2.launch.py use_sim_time:=True
```

**In RViz**

1. Click **2D Pose Estimate**, then click and drag at the robot's position (about x=-2.0, y=-0.5, facing +x). The laser scan should line up with the map walls.
2. Wait about 15 seconds for the costmaps to appear.
3. Click **Nav2 Goal** and choose a point in free space. The robot plans and drives there.
4. **Obstacle avoidance demo:** insert a box in front of the robot from the Gazebo toolbar. The local costmap picks it up and the robot replans around it.

## 4. How it works

```
LiDAR /scan + /odom -> AMCL (map->odom) -> global + local costmaps
                    -> global planner -> local controller -> /cmd_vel -> robot
```

Transform chain: `map -> odom -> base_footprint -> base_link`.

## 5. Debug checks

```bash
ros2 topic echo /clock --once            # sim time running (press Play in Gazebo if empty)
ros2 topic hz /scan                      # LiDAR publishing
ros2 run tf2_ros tf2_echo map odom       # exists only after initial pose is set
ros2 lifecycle get /bt_navigator         # must be: active [3]
```

Kill stale processes before relaunching:

```bash
pkill -9 -f "gz sim"; pkill -9 -f ruby; pkill -9 -f ros_gz; pkill -9 -f parameter_bridge
pkill -9 -f rviz2; pkill -9 -f component_container; pkill -9 -f robot_state_publisher
ros2 daemon stop
```

## 6. Problems solved

| Problem | Cause | Fix |
|---|---|---|
| `bt_navigator: Action server is inactive` | Nav2 nodes inactive: no initial pose, incomplete TF | Set initial pose, check frames with `tf2_echo`, wait for costmaps |
| Gazebo world empty, spawner looping on `Requesting list of world names` | Model path set after Gazebo started; duplicate `gz sim` running | Set `GZ_SIM_RESOURCE_PATH` before the server in the launch file; kill stale processes |
| `Invalid frame ID "base_link"` | URDF is xacro with `${namespace}` placeholders read as plain text | Removed the placeholders (single robot) |
| `Transform data too old` / collision monitor ignoring scan | Nav2 on wall-clock time, Gazebo on sim time | `use_sim_time` defaults to `true` |
| `KeyError: 'TURTLEBOT3_MODEL'` | Variable not set | Launch files default to `waffle` |
| `Could not parse config ... bridge.yaml` | Config in the wrong folder | Moved to `bridge/` and installed it in `CMakeLists.txt` |

## 7. Credits

TurtleBot3 packages by ROBOTIS (Apache-2.0). Navigation by the Nav2 project.
