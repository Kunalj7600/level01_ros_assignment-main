# testbed_navigation

Manual Nav2 setup for the Testbed-T1.0.0 robot (ROS 2 Humble, Gazebo Classic 11).
Instead of calling `nav2_bringup`, each part of the navigation workflow is launched
on its own, with its own launch file and parameter file:

| Step | Launch file | Parameters | Nodes |
|---|---|---|---|
| Load the map | `map_loader.launch.py` | (inline) | `map_server` |
| Localize | `localization.launch.py` | `config/amcl_params.yaml` | `amcl` |
| Navigate | `navigation.launch.py` | `config/nav2_params.yaml` | `controller_server`, `planner_server`, `behavior_server`, `bt_navigator` |

Each launch file starts its own `lifecycle_manager` (`lifecycle_manager_map`,
`lifecycle_manager_localization`, `lifecycle_manager_navigation`) with `autostart: true`,
so the components can be started, tested and restarted independently.

The bugs I fixed in the starter code are listed in `bugfixes.md` in the repository root.

## Package layout

```
testbed_navigation/
├── config/
│   ├── amcl_params.yaml     # AMCL parameters
│   └── nav2_params.yaml     # planner, controller, costmaps, behaviors, BT navigator
├── launch/
│   ├── map_loader.launch.py
│   ├── localization.launch.py
│   └── navigation.launch.py
├── CMakeLists.txt
├── package.xml
└── README.md
```

## Requirements

- Ubuntu 22.04, ROS 2 Humble, Gazebo Classic 11
- Nav2 packages: `sudo apt install ros-humble-navigation2 ros-humble-nav2-bringup`
  (only the plugins are used, `nav2_bringup`'s launch files are not)
- `teleop_twist_keyboard` (optional, for driving by hand)

## Build

```bash
cd ~/assignment_ws
colcon build --symlink-install
source install/setup.bash
```

Source `install/setup.bash` again in every terminal after a build that adds a package
or a new file.

## Run

Use one terminal per step, each with the workspace sourced. Wait for
"Managed nodes are active" in a terminal before starting the next one.

```bash
# 1. Gazebo + RViz + robot
ros2 launch testbed_bringup testbed_full_bringup.launch.py

# 2. Map
ros2 launch testbed_navigation map_loader.launch.py

# 3. Localization
ros2 launch testbed_navigation localization.launch.py

# 4. Navigation
ros2 launch testbed_navigation navigation.launch.py
```

Send a goal with **2D Goal Pose** in RViz, or from the command line:

```bash
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: map}, pose: {position: {x: 2.0, y: 3.0}, orientation: {w: 1.0}}}}"
```

In RViz (Fixed Frame `map`) useful displays are `/map` (Durability: Transient Local),
`/scan`, `/particle_cloud`, `/plan`, `/local_plan`, `/global_costmap/costmap` and
`/local_costmap/costmap`.

## How it works

```
 map_server ──/map──▶ amcl ──TF map→odom──▶ ┐
                       ▲                     │
   /scan, /odom ───────┘                     ▼
                              planner_server (global costmap)
                              controller_server (local costmap) ──/cmd_vel──▶ diff drive
                              behavior_server (spin, backup, wait, ...)
                              bt_navigator  ◀── NavigateToPose action
```

TF chain: `map → odom → base_footprint → base_link → ...`. The Gazebo diff-drive plugin
publishes `odom → base_footprint`, and AMCL publishes `map → odom`.

### Map loading
`map_server` loads `testbed_bringup/maps/testbed_world.yaml` (405 x 400 cells at
0.05 m/cell). The path is built with `get_package_share_directory`, so nothing is
hardcoded.

### Localization
AMCL uses the differential motion model (`robot_model_type: "nav2_amcl::DifferentialMotionModel"`),
`base_frame_id: base_footprint`, `odom_frame_id: odom`, `global_frame_id: map` and
`scan_topic: scan`. The initial pose is set to the spawn pose (0, 5, 0), so AMCL starts
correctly when the simulation is freshly started.

### Navigation
- **Global planner:** `nav2_navfn_planner/NavfnPlanner` on the global costmap
  (static, obstacle and inflation layers).
- **Local controller:** `dwb_core::DWBLocalPlanner` on a 3 x 3 m rolling local costmap
  (voxel and inflation layers), maximum speed 0.26 m/s.
- **Costmaps:** `robot_radius: 0.22`, inflation radius 0.55 m, both fed by `/scan`.
- **Goal tolerance:** 0.25 m in position and 0.25 rad in yaw.
- **Behaviors:** spin, backup, drive on heading, assisted teleop and wait.
- **BT navigator:** Nav2's default tree with replanning and recovery.

All nodes run with `use_sim_time: true`.

## How I tested it

- Map: `ros2 lifecycle get /map_server` reports `active` and `/map` shows the room in RViz.
- Localization: `ros2 run tf2_ros tf2_echo map odom`. In this simulation `/odom` reports the
  world pose, so a correct AMCL gives `map -> odom` close to (0, 0, 0 deg). After driving a
  short distance it converged to about (0.04 m, 0.03 m, under 1 deg).
- Navigation: sent a goal at (2.0, 3.0) with the command above. The planner produced a
  path, the robot drove to it and stopped inside the goal tolerance.

## Challenges and how I solved them

1. **`apt install` failed with 404.** My package list was stale, and Nav2 was already
   installed. `sudo apt update` fixes it, or the install can be skipped if
   `ros2 pkg list | grep nav2` shows the packages.
2. **AMCL failed to configure.** The error said the class `differential` does not exist and
   listed the declared types. The installed version expects
   `nav2_amcl::DifferentialMotionModel`. Lesson: when Nav2 prints "Declared types are...",
   use one of those strings verbatim.
3. **"Package 'testbed_navigation' not found".** The terminal was sourced before the
   package existed. Rebuilding and sourcing `install/setup.bash` again fixed it.
4. **Node names must match the yaml keys.** Nav2 reads parameters from the block whose key
   equals the node name. Using different names meant the parameter file was silently
   ignored. The names are now `controller_server`, `planner_server`, `behavior_server` and
   `bt_navigator`. Lifecycle managers also need unique names.
5. **Startup warnings about a transform timeout.** The global costmap logs
   "Timed out waiting for transform from base_footprint to map" for a second or two while it
   activates. It is a start-up race, and it stops once TF is available.
6. **AMCL lost the robot.** After a restart with the robot away from the spawn pose,
   AMCL locked onto a similar-looking corner about 21 m away. The scan dots still
   looked aligned with the walls because AMCL had found a good local fit. I detected it by
   comparing `/odom` with `ros2 run tf2_ros tf2_echo map base_footprint`, and fixed it by
   publishing the true pose on `/initialpose`:
   ```bash
   ros2 topic pub --once /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
   "{header: {frame_id: map}, pose: {pose: {position: {x: X, y: Y}, orientation: {z: QZ, w: QW}}, covariance: [0.25,0,0,0,0,0, 0,0.25,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0, 0,0,0,0,0,0.07]}}"
   ```
   Always restart localization with the robot at the spawn pose, or set the pose in RViz.
7. **Costmap looked misaligned with the scan.** The thick dark bands in the costmap are the
   inflated obstacle halo, so scan points sit on the edge of a band even when everything is
   aligned. Alignment should be judged against the raw `/map`, not the costmap.

## Known limitations

- Only basic navigation is covered: no waypoint following, velocity smoother or collision
  monitor.
- Wide inflation (0.55 m) can make narrow doorways look blocked. If the planner refuses
  a doorway, lower `inflation_radius` in both costmaps.
- The simulation runs below real time on my machine (about 0.7), which slows AMCL and
  the controller slightly.

## Demo

- Video: https://drive.google.com/file/d/1gYCdw22jFSbZ3DKL0tcZEM0j2f-MOHKg/view?usp=sharing
## Author

- Name: Kunal Singh
- E-Mail: kunalj6988@gmail.com
  
