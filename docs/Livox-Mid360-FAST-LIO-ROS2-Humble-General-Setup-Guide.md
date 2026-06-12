# Livox Mid-360 + FAST_LIO_ROS2 General Guide

This guide is a general checklist for bringing up a Livox Mid-360 with `livox_ros_driver2` and `FAST_LIO_ROS2` on Ubuntu 22.04 + ROS 2 Humble.

## 1. System Relationship

```text
Ubuntu 22.04
-> ROS 2 Humble
-> livox_ros_driver2
-> FAST_LIO_ROS2
-> RViz
```

Ubuntu handles hardware, networking, system Python, C++ build tools, and shared libraries.

ROS 2 provides the robot middleware: topics, launch files, package discovery, `ros2` commands, and `colcon` builds.

Conda is not required for this workflow. If Conda is installed, deactivate it before building or launching ROS packages:

```bash
conda deactivate 2>/dev/null || true
```

## 2. Recommended Versions

| Component | Recommended |
| --- | --- |
| OS | Ubuntu 22.04 LTS |
| ROS | ROS 2 Humble |
| Python | System Python, usually `/usr/bin/python3` |
| LiDAR | Livox Mid-360 |
| Driver | `livox_ros_driver2` |
| SLAM | `FAST_LIO_ROS2` |

Avoid mixing Ubuntu 22.04 with ROS 1 Noetic instructions.

## 3. Install ROS 2 Humble

```bash
sudo apt update
sudo apt install software-properties-common curl -y
sudo add-apt-repository universe

sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt install ros-humble-desktop -y
sudo apt install python3-colcon-common-extensions python3-empy python3-catkin-pkg -y
sudo apt install ros-humble-pcl-ros ros-humble-pcl-conversions libeigen3-dev -y
```

Source ROS:

```bash
source /opt/ros/humble/setup.bash
echo $ROS_DISTRO
```

Expected:

```text
humble
```

## 4. Workspace Layout

Use one project folder with three main parts:

```text
FAST-LIO/
├── Livox-SDK2/
├── ros2_ws/
│   └── src/livox_ros_driver2/
└── fastlio_ws/
    └── src/FAST_LIO_ROS2/
```

Replace `<FAST_LIO_ROOT>` below with your actual project path.

## 5. Build Livox-SDK2

```bash
cd <FAST_LIO_ROOT>
git clone https://github.com/Livox-SDK/Livox-SDK2.git

cd Livox-SDK2
mkdir -p build
cd build
cmake ..
make -j$(nproc)
sudo make install
sudo ldconfig
```

Verify:

```bash
ldconfig -p | grep livox
```

## 6. Build livox_ros_driver2

```bash
mkdir -p <FAST_LIO_ROOT>/ros2_ws/src
cd <FAST_LIO_ROOT>/ros2_ws/src
git clone https://github.com/Livox-SDK/livox_ros_driver2.git

conda deactivate 2>/dev/null || true
source /opt/ros/humble/setup.bash

cd <FAST_LIO_ROOT>/ros2_ws/src/livox_ros_driver2
./build.sh humble
```

Source the driver workspace:

```bash
source <FAST_LIO_ROOT>/ros2_ws/install/setup.bash
```

## 7. Build FAST_LIO_ROS2

```bash
mkdir -p <FAST_LIO_ROOT>/fastlio_ws/src
cd <FAST_LIO_ROOT>/fastlio_ws/src
git clone https://github.com/Ericsii/FAST_LIO_ROS2.git --recursive

conda deactivate 2>/dev/null || true
source /opt/ros/humble/setup.bash
source <FAST_LIO_ROOT>/ros2_ws/install/setup.bash

cd <FAST_LIO_ROOT>/fastlio_ws
colcon build --symlink-install --cmake-args -DPython3_EXECUTABLE=/usr/bin/python3
```

Source the FAST_LIO workspace:

```bash
source <FAST_LIO_ROOT>/fastlio_ws/install/setup.bash
```

## 8. Network Setup

Connect the Mid-360 to the computer with Ethernet.

Typical static IP plan:

| Device | Example IP |
| --- | --- |
| Computer Ethernet | `192.168.1.50/24` |
| Mid-360 | `192.168.1.161` |

Check network interfaces:

```bash
ip -4 addr show
nmcli device status
```

Set the wired interface manually using NetworkManager. Replace the connection name with your actual wired connection:

```bash
nmcli connection show
sudo nmcli connection modify "Wired connection 1" ipv4.method manual ipv4.addresses 192.168.1.50/24
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

Scan for the LiDAR:

```bash
sudo apt install nmap -y
nmap -sn 192.168.1.0/24
```

## 9. Configure MID360_config.json

Edit:

```text
<FAST_LIO_ROOT>/ros2_ws/src/livox_ros_driver2/config/MID360_config.json
```

Set all `host_net_info` IP fields to the computer Ethernet IP:

```json
"cmd_data_ip": "192.168.1.50",
"push_msg_ip": "192.168.1.50",
"point_data_ip": "192.168.1.50",
"imu_data_ip": "192.168.1.50",
"log_data_ip": "192.168.1.50"
```

Set `lidar_configs` to the actual Mid-360 IP:

```json
"ip": "192.168.1.161"
```

## 10. Start the LiDAR Driver

Terminal A:

```bash
conda deactivate 2>/dev/null || true
source /opt/ros/humble/setup.bash
source <FAST_LIO_ROOT>/ros2_ws/install/setup.bash

ros2 launch livox_ros_driver2 msg_MID360_launch.py
```

Expected healthy signs:

```text
successfully change work mode
successfully enable Livox Lidar imu
livox/imu publish use imu format
livox/lidar publish use livox custom format
```

Verify topics:

```bash
ros2 topic list
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

Typical rates:

```text
/livox/lidar  about 10 Hz
/livox/imu    about 200 Hz
```

## 11. Start FAST_LIO

Terminal B:

```bash
conda deactivate 2>/dev/null || true
source /opt/ros/humble/setup.bash
source <FAST_LIO_ROOT>/ros2_ws/install/setup.bash
source <FAST_LIO_ROOT>/fastlio_ws/install/setup.bash

ros2 launch fast_lio mapping.launch.py
```

Expected healthy logs:

```text
Node init finished.
IMU Initial Done
Initialize the map kdtree
```

`Stereo is NOT SUPPORTED` from RViz is usually harmless. It only means stereo rendering is unavailable in the current display environment.

## 12. RViz Healthy State

A good RViz state usually looks like:

- `Global Status` is OK.
- `Fixed Frame` is OK, often `camera_init`.
- `TF`, `Odometry`, `Path`, and `CloudMap` have no red errors.
- A point cloud map appears and grows as the LiDAR moves.

## 13. Common Issues

### ROS is not sourced

Symptom:

```text
ros2: command not found
```

Fix:

```bash
source /opt/ros/humble/setup.bash
```

### Conda breaks build

Symptom: Python module errors such as `empy`, `catkin_pkg`, or `ament_package`.

Fix:

```bash
conda deactivate 2>/dev/null || true
colcon build --symlink-install --cmake-args -DPython3_EXECUTABLE=/usr/bin/python3
```

### Livox shared library not found

Fix:

```bash
sudo ldconfig
ldconfig -p | grep livox
```

### Bind failed or no LiDAR data

Check:

```bash
ip -4 addr show
ping <LIDAR_IP>
nmap -sn 192.168.1.0/24
```

Confirm:

- Computer Ethernet IP matches `host_net_info`.
- LiDAR IP matches `lidar_configs`.
- Ethernet cable and LiDAR power are good.
- Firewall is not blocking traffic.

### FAST_LIO says `No point, skip this scan`

This can happen briefly during startup. It is usually fine if point cloud data appears afterward.

If it repeats forever, check:

```bash
ros2 topic hz /livox/lidar
ros2 topic echo /livox/lidar --once
```

## 14. Success Criteria

The setup is considered successful when:

- `livox_ros_driver2` connects to the Mid-360.
- `/livox/lidar` publishes around 10 Hz.
- `/livox/imu` publishes around 200 Hz.
- FAST_LIO logs `IMU Initial Done`.
- RViz shows odometry, path, and an accumulating point cloud map.
