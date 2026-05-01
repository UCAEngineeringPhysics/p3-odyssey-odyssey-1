# The Odyssey — Autonomous Coffee Delivery

[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/7KaPTW5f)

**Maintainer:** odyssey <odyssey@uca.edu> · **License:** Apache-2.0 · **ROS 2:** Jazzy
**Map:** [`lsc_odyssey_1.pgm`](lsc_odyssey_1.pgm) / [`lsc_odyssey_1.yaml`](lsc_odyssey_1.yaml)

A HomeR-based rover that drives itself from LSC 159 to the front desk in LSC 171, carrying a 12-oz coffee on a trailer at the back. In our design, the trailer has hazard lights using a simple 555 timer IC circuit. We also added a circuit on the front that plays the Galaga theme with headlights on during navigation!

>![Hazards_schematic](hazards_schamatic.png)

>![galaga_circuit](galaga_circuit.jpg)

>![galaga_schematic](galaga_schematic.jpg)

---

## 1. Coffee Transportation

### Mechanical Design

>![New Base](NewBase-1.png)

>![Trailer](Trailer-1.png)

>![Bearing Parts](BearingsParts-1.png)

>![Odyssey Build](OdysseyBuild.jpg) 
### Installation

> Utilizing the lables on the technical drawings and the image provided, slot the 12mm bearings into the appropriate places on the New base and Trailer, secue them with set screws

> Then using the same size bearings slot them into the bearing brackets and thread a long screw though the bearing to use as a stud to attach the same kind of wheels used on the front of the bot.
> Now mount the bracket to the trailer with the flat side (the side where the bearing was inserted) against the trailer and affix using 2.5mm screws
---

## 2. Software Setup

Two machines, same Wi-Fi, same `ROS_DOMAIN_ID`:

- **Raspberry Pi** (on the rover) — motors, LiDAR, IMU, TF.
- **Laptop** — SLAM, Nav2, RViz.

### Raspberry Pi (ROS 2 Jazzy)

```bash
sudo apt install -y ros-jazzy-rplidar-ros ros-jazzy-robot-localization \
    ros-jazzy-teleop-twist-keyboard python3-pip
pip install pyserial

mkdir -p ~/robotics_2/src && cd ~/robotics_2/src
git clone <this-repo-url> .
cd ~/robotics_2 && colcon build --packages-select odyssey_controller
source install/setup.bash
```

Flash the Pico firmware once via Thonny (upload the contents of `src/odyssey_controller/picoscripts/` to the device).

### Laptop (ROS 2 Jazzy)

```bash
sudo apt install -y ros-jazzy-slam-toolbox ros-jazzy-navigation2 \
    ros-jazzy-nav2-bringup ros-jazzy-rviz2

mkdir -p ~/homer_ws/src && cd ~/homer_ws/src
git clone https://github.com/linzhangUCA/homer_navigation.git
cd ~/homer_ws && colcon build
source install/setup.bash
```

---

## 3. Running It

### Build a map

**Pi:**
```bash
cd robotics2_ws && source install/setup.bash
ros2 launch odyssey_controller bringup.launch.py
```

**Laptop:**
```bash
cd homer_ws && source install/setup.bash
ros2 launch homer_navigation create_map.launch.py
```

Rviz will open up upon launch. Use this to see what the robot maps and debug in the case of any discrepancies. Drive with the keyboard/controller until the LSC 159 → 171 hallway is fully scanned in RViz. Take your time with this because if the wifi signal is weak between the Pi and the laptop, the map can begin to read inaccurately.

### Save the map

From the laptop, in `~/homer_ws/src/homer_navigation/maps/`:

```bash
cd robotics2_ws && source install/setup.bash
ros2 run nav2_map_server map_saver_cli -f lsc_odyssey_1
```

This writes `lsc_odyssey_1.pgm` and `lsc_odyssey_1.yaml`. The navigate launch picks them up automatically.

### Navigate

**Pi:**
```bash
cd robotics2_ws && source install/setup.bash
ros2 launch odyssey_controller bringup.launch.py
```

**Laptop:**
```bash
cd homer_ws && source install/setup.bash
ros2 launch homer_navigation navigate.launch.py
```

Set the initial pose in RViz with **2D Pose Estimate**, then send the LSC 171 goal with **2D Goal Pose**. The rover drives itself there and indicates arrival via the LED.
