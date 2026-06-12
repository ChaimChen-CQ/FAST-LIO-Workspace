# Livox Mid-360 + FAST_LIO_ROS2 Humble 通用配置手册

[English](README.md)

这是一份通用复现手册，用于在 **Ubuntu 22.04 + ROS 2 Humble** 环境下启动 Livox Mid-360，并运行 `livox_ros_driver2` 与 `FAST_LIO_ROS2`。

## 1. 系统关系

整体关系可以理解为：

```text
Ubuntu 22.04
-> ROS 2 Humble
-> livox_ros_driver2
-> FAST_LIO_ROS2
-> RViz
```

Ubuntu 负责网卡、文件系统、系统 Python、C++ 编译工具、动态库等底层环境。

ROS 2 是机器人中间件，负责 topic 通信、launch 启动、package 管理、`ros2` 命令和 `colcon` 编译。

Conda 不是必须的。配置和编译 ROS 项目时，建议先退出 Conda：

```bash
conda deactivate 2>/dev/null || true
```

## 2. 推荐版本

| 组件 | 推荐版本 |
| --- | --- |
| 操作系统 | Ubuntu 22.04 LTS |
| ROS | ROS 2 Humble |
| Python | 系统 Python，通常是 `/usr/bin/python3` |
| 雷达 | Livox Mid-360 |
| 雷达驱动 | `livox_ros_driver2` |
| 建图算法 | `FAST_LIO_ROS2` |

注意：Ubuntu 22.04 推荐搭配 ROS 2 Humble，不要混用 ROS 1 Noetic 的安装流程。

## 3. 安装 ROS 2 Humble

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

加载 ROS 环境：

```bash
source /opt/ros/humble/setup.bash
echo $ROS_DISTRO
```

期望输出：

```text
humble
```

## 4. 推荐目录结构

建议把 SDK、雷达驱动和 FAST_LIO 放在同一个总目录下：

```text
FAST-LIO/
├── Livox-SDK2/
├── ros2_ws/
│   └── src/livox_ros_driver2/
└── fastlio_ws/
    └── src/FAST_LIO_ROS2/
```

下面的 `<FAST_LIO_ROOT>` 代表你自己的项目根目录。

## 5. 编译 Livox-SDK2

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

检查动态库：

```bash
ldconfig -p | grep livox
```

如果后面运行驱动时提示找不到 Livox 动态库，通常重新执行下面命令即可：

```bash
sudo ldconfig
```

## 6. 编译 livox_ros_driver2

```bash
mkdir -p <FAST_LIO_ROOT>/ros2_ws/src
cd <FAST_LIO_ROOT>/ros2_ws/src
git clone https://github.com/Livox-SDK/livox_ros_driver2.git

conda deactivate 2>/dev/null || true
source /opt/ros/humble/setup.bash

cd <FAST_LIO_ROOT>/ros2_ws/src/livox_ros_driver2
./build.sh humble
```

加载驱动工作区：

```bash
source <FAST_LIO_ROOT>/ros2_ws/install/setup.bash
```

## 7. 编译 FAST_LIO_ROS2

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

加载 FAST_LIO 工作区：

```bash
source <FAST_LIO_ROOT>/fastlio_ws/install/setup.bash
```

## 8. 网络配置

Mid-360 建议通过网线直连电脑。

常见 IP 规划：

| 设备 | 示例 IP |
| --- | --- |
| 电脑有线网卡 | `192.168.1.50/24` |
| Mid-360 | `192.168.1.161` |

查看网卡：

```bash
ip -4 addr show
nmcli device status
```

使用 NetworkManager 设置有线网卡。先查看连接名：

```bash
nmcli connection show
```

假设有线连接名为 `Wired connection 1`：

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual ipv4.addresses 192.168.1.50/24
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

扫描雷达 IP：

```bash
sudo apt install nmap -y
nmap -sn 192.168.1.0/24
```

如果扫描到的 Mid-360 IP 不是 `192.168.1.161`，后面的配置文件也要同步修改。

## 9. 配置 MID360_config.json

编辑：

```text
<FAST_LIO_ROOT>/ros2_ws/src/livox_ros_driver2/config/MID360_config.json
```

把 `host_net_info` 里的 IP 改成电脑有线网卡 IP，例如：

```json
"cmd_data_ip": "192.168.1.50",
"push_msg_ip": "192.168.1.50",
"point_data_ip": "192.168.1.50",
"imu_data_ip": "192.168.1.50",
"log_data_ip": "192.168.1.50"
```

把 `lidar_configs` 里的 IP 改成实际雷达 IP，例如：

```json
"ip": "192.168.1.161"
```

## 10. 启动雷达驱动

终端 A：

```bash
conda deactivate 2>/dev/null || true
source /opt/ros/humble/setup.bash
source <FAST_LIO_ROOT>/ros2_ws/install/setup.bash

ros2 launch livox_ros_driver2 msg_MID360_launch.py
```

健康日志通常包含：

```text
successfully change work mode
successfully enable Livox Lidar imu
livox/imu publish use imu format
livox/lidar publish use livox custom format
```

验证 topic：

```bash
ros2 topic list
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

典型频率：

```text
/livox/lidar  约 10 Hz
/livox/imu    约 200 Hz
```

## 11. 启动 FAST_LIO

终端 B：

```bash
conda deactivate 2>/dev/null || true
source /opt/ros/humble/setup.bash
source <FAST_LIO_ROOT>/ros2_ws/install/setup.bash
source <FAST_LIO_ROOT>/fastlio_ws/install/setup.bash

ros2 launch fast_lio mapping.launch.py
```

健康日志通常包含：

```text
Node init finished.
IMU Initial Done
Initialize the map kdtree
```

RViz 里出现：

```text
Stereo is NOT SUPPORTED
```

通常不是错误，只是当前显示环境不支持立体渲染。普通 RViz 可视化不需要这个功能，可以忽略。

## 12. RViz 正常状态

比较健康的 RViz 状态：

- `Global Status` 为 OK。
- `Fixed Frame` 正常，常见为 `camera_init`。
- `TF`、`Odometry`、`Path`、`CloudMap` 没有红色错误。
- 能看到点云地图，并且随着移动持续累积。

## 13. 常见问题

### 13.1 ROS 没有 source

现象：

```text
ros2: command not found
```

处理：

```bash
source /opt/ros/humble/setup.bash
```

### 13.2 Conda 干扰编译

现象：编译时找不到 `empy`、`catkin_pkg`、`ament_package` 等 Python 模块。

处理：

```bash
conda deactivate 2>/dev/null || true
colcon build --symlink-install --cmake-args -DPython3_EXECUTABLE=/usr/bin/python3
```

### 13.3 找不到 Livox 动态库

处理：

```bash
sudo ldconfig
ldconfig -p | grep livox
```

### 13.4 Bind Failed 或没有雷达数据

检查：

```bash
ip -4 addr show
ping <LIDAR_IP>
nmap -sn 192.168.1.0/24
```

确认：

- 电脑有线网卡 IP 和 `host_net_info` 一致。
- 雷达 IP 和 `lidar_configs` 一致。
- 网线连接正常，雷达已供电。
- 防火墙没有阻断通信。

### 13.5 FAST_LIO 提示 `No point, skip this scan`

启动初期偶尔出现通常没问题。如果一直重复，需要检查点云是否真的发布：

```bash
ros2 topic hz /livox/lidar
ros2 topic echo /livox/lidar --once
```

## 14. 成功标准

满足下面条件即可认为链路跑通：

- `livox_ros_driver2` 成功连接 Mid-360。
- `/livox/lidar` 约 10 Hz。
- `/livox/imu` 约 200 Hz。
- FAST_LIO 输出 `IMU Initial Done`。
- RViz 中能看到 odometry、path 和持续累积的点云地图。
