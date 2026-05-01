# The Odyssey — Autonomous Coffee Delivery

[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/7KaPTW5f)

**Maintainer:** odyssey <odyssey@uca.edu>
**License:** Apache-2.0
**ROS 2 Distribution:** Jazzy
**Map:** `lsc_odyssey_1.pgm` / `lsc_odyssey_1.yaml` (LSC 159 → LSC 171)

This package drives a HomeR rover autonomously from Dr. Chen's lab (LSC 159) to the PAE department front desk (LSC 171), delivering a 12 oz cup of coffee on top of the chassis.

---

## 1. Coffee Transportation Solution

### Mechanical Design

The cup carrier is a flat platform mounted to the top deck of the HomeR chassis, centered over the drive axle to keep the payload above the rover's pivot point and minimize tip-over moment during turns.

**Key components**
- **Base plate** — laser-cut acrylic disc bolted to the four M3 standoffs on the top deck of the HomeR chassis.
- **Cup retainer** — a circular ring sized to the bottom of a standard 12 oz coffee cup, holding it against lateral motion without compressing the sleeve.
- **Anti-slip pad** — silicone disc inside the retainer to prevent the cup from sliding vertically or rotating during acceleration / braking.
- **Center of mass** — the cup's CoM sits directly over the wheel axle to minimize the pitching moment when the controller commands acceleration changes.

**Critical dimensions** (matched to the standard 12 oz cup):

| Dimension | Value |
|-----------|-------|
| Cup top diameter | 90 mm |
| Cup bottom diameter | 60 mm |
| Cup height | 110 mm |
| Retainer inner diameter | 62 mm (loose slip fit on the cup base) |
| Base plate diameter | 140 mm |
| Mount hole pattern | 4× M3 on a 60 mm bolt circle (matches HomeR top deck) |

> **TODO:** drop your sketch / techdraw here. Recommended path: `drawings/cup_carrier.png` (or `.pdf`). Reference it as
> `![Cup Carrier](drawings/cup_carrier.png)`

### Hardware Installation Guide

1. **Power down the rover** and remove the existing top deck plate (4× M3 screws).
2. **Place the cup carrier** over the four M3 standoffs on the HomeR top deck. Confirm orientation: the retainer ring is centered on the chassis, with no overhang past the bumpers.
3. **Bolt down** with four M3×8 mm screws and lock washers. Torque finger-tight, then a quarter-turn with a 2.5 mm hex key.
4. **Insert the silicone anti-slip pad** into the bottom of the retainer ring.
5. **Verify clearance**: the LiDAR's 360° scan plane must remain unobstructed by the cup. Looking from above, the cup must not protrude past the edge of the LiDAR housing.
6. **Load the cup** with the lid sealed and place it inside the retainer ring just before launch.

> **TODO:** add a 2–4 photo install sequence (or annotated diagram) here for the bonus 5%. Save photos to `drawings/install_step_1.jpg` … `install_step_4.jpg`.

---

## 2. Software Usage Instructions

The system runs across two machines:

- **Raspberry Pi 5** on the rover — drives motors, publishes `/scan`, `/odom`, `/imu`, broadcasts TF.
- **Server computer** (laptop) — runs `slam_toolbox`, Nav2, RViz, and the `navigate` action client.

Both machines must be on the **same ROS_DOMAIN_ID** and the same Wi-Fi network.

### 2.1 Raspberry Pi setup (fresh ROS 2 Jazzy)

```bash
# ROS 2 dependencies
sudo apt update && sudo apt install -y \
    ros-jazzy-rplidar-ros \
    ros-jazzy-tf2-ros \
    ros-jazzy-robot-localization \
    ros-jazzy-teleop-twist-joy \
    ros-jazzy-teleop-twist-keyboard \
    python3-pip
pip install pyserial

# This package
mkdir -p ~/robotics_2/src && cd ~/robotics_2/src
git clone <this-repo-url> .
cd ~/robotics_2
colcon build --packages-select odyssey_controller
echo "source ~/robotics_2/install/setup.bash" >> ~/.bashrc
echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
source ~/.bashrc

# Pico firmware (one-time, via Thonny → MicroPython device)
#   Upload src/odyssey_controller/picoscripts/main.py            → /main.py
#   Upload src/odyssey_controller/picoscripts/drivetrain/        → /drivetrain/
#   Upload src/odyssey_controller/picoscripts/imu.py             → /imu.py
```

### 2.2 Server computer setup (fresh ROS 2 Jazzy)

```bash
sudo apt update && sudo apt install -y \
    ros-jazzy-slam-toolbox \
    ros-jazzy-navigation2 \
    ros-jazzy-nav2-bringup \
    ros-jazzy-rviz2 \
    ros-jazzy-tf2-tools

# This package (for the action client + map + Nav2 params)
mkdir -p ~/robotics_2/src && cd ~/robotics_2/src
git clone <this-repo-url> .
cd ~/robotics_2
colcon build --packages-select odyssey_controller
echo "source ~/robotics_2/install/setup.bash" >> ~/.bashrc
echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
source ~/.bashrc
```

### 2.3 Create a new map

**Pi (terminal 1)** — bring up the rover:
```bash
ros2 launch odyssey_controller bringup.launch.py
```

**Server (terminal 2)** — start SLAM and RViz:
```bash
ros2 launch slam_toolbox online_async_launch.py
ros2 run rviz2 rviz2 -d $(ros2 pkg prefix odyssey_controller)/share/odyssey_controller/config/mapping.rviz
```

**Server (terminal 3)** — drive the rover with the keyboard so it can build the map:
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Drive slowly down the LSC hallway from 159 to 171, doubling back over the same ground a couple times to close loops cleanly.

### 2.4 Save the map

Once the map looks complete in RViz, save it from the **server** with the `nav2_map_server` CLI (bonus 1%):

```bash
ros2 run nav2_map_server map_saver_cli -f lsc_odyssey_1
```

This produces `lsc_odyssey_1.pgm` and `lsc_odyssey_1.yaml` in the current directory. Move both into the repo root and commit them.

### 2.5 Run autonomous navigation with the saved map

**Pi (terminal 1)** — rover bringup:
```bash
ros2 launch odyssey_controller bringup.launch.py
```

**Server (terminal 2)** — Nav2 + map server + AMCL, pre-loaded with `lsc_odyssey_1`:
```bash
ros2 launch odyssey_controller navigate.launch.py \
    map:=$HOME/robotics_2/lsc_odyssey_1.yaml
```

**Server (terminal 3)** — the navigation action client that sends the goal pose for the LSC 171 front desk:
```bash
ros2 run odyssey_controller navigate
```

In RViz, use **2D Pose Estimate** to set the rover's initial pose at the LSC 159 start line before launching the action client. The rover will then drive itself to the goal and indicate arrival via the LED.

---

## 3. Navigation Node

### Files

| Path | Purpose |
|------|---------|
| `lsc_odyssey_1.pgm`, `lsc_odyssey_1.yaml` | Occupancy grid map of the LSC 159 → 171 corridor |
| `src/odyssey_controller/odyssey_controller/navigation_node.py` | Sends the goal pose via Nav2 `NavigateToPose` action |
| `src/odyssey_controller/launch/navigate.launch.py` | Brings up `map_server`, `amcl`, Nav2, and the action client |
| `src/odyssey_controller/config/` | Nav2, EKF, and joystick parameters |

### Behavior

1. Wait for the Nav2 action server (`navigate_to_pose`) to become available.
2. Send a `NavigateToPose` goal at the LSC 171 front-desk pose (in the `map` frame).
3. Stream feedback to the LED indicator topic.
4. On success, latch a fast red strobe on the LED and shut down cleanly.
5. On failure / cancellation, latch blue and exit non-zero.

### Goal pose

The destination pose is hard-coded for the LSC 171 front desk. To re-run from a different start, only the **2D Pose Estimate** in RViz needs to change — the goal is fixed.

| Frame | x (m) | y (m) | yaw (deg) |
|-------|-------|-------|-----------|
| `map` | *fill in from your saved map* | *fill in* | *fill in* |

Read the goal from the map by clicking with the **2D Goal Pose** tool in RViz once, then copy the values into `navigation_node.py`.

### Package metadata

Maintainer, email, description, and license are all set in `src/odyssey_controller/package.xml` and `setup.py`. The package builds cleanly on a fresh ROS 2 Jazzy install with:

```bash
cd ~/robotics_2
colcon build --packages-select odyssey_controller
```

---

