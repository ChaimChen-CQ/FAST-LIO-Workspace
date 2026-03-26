# Livox Mid-360 + FAST_LIO_ROS2 部署手册

本说明面向 **Ubuntu 22.04 + ROS 2 Humble** 环境，目标是在电脑上完成 Livox Mid-360 雷达接入、驱动编译、FAST_LIO_ROS2 运行。（基于https://efficient-noodle-c93.notion.site/Livox-Mid-360-FAST_LIO-3287dda0610a803a9d58e6748f7f7f2b）

## 1. 环境要求

* 操作系统：Ubuntu 22.04 LTS
* ROS：ROS 2 Humble Desktop
* Python：系统自带 `/usr/bin/python3`
* 电脑网卡：手动静态 IP
* 雷达：Livox Mid-360
* 建议：编译前先退出 Conda

## 2. 网络配置

将电脑有线网卡设置为手动模式：

* IP Address：`192.168.1.50`
* Netmask：`255.255.255.0`
* Gateway：`192.168.1.1`（如不需要可留空）

说明：Mid-360 的实际雷达 IP 以扫描结果为准，本次实测为 `192.168.1.161`。

## 3. 安装 ROS 2 Humble

```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository universe
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt install ros-humble-desktop -y
sudo apt install python3-colcon-common-extensions python3-empy python3-catkin-pkg -y
sudo apt install ros-humble-pcl-ros ros-humble-pcl-conversions libeigen3-dev -y
```

把 ROS 环境写入 shell 配置：

```bash
echo "source /opt/ros/humble/setup.zsh" >> ~/.zshrc
source ~/.zshrc
```

## 4. 安装 Livox SDK2

```bash
cd <本地路径>/FAST-LIO
git clone https://github.com/Livox-SDK/Livox-SDK2.git
cd Livox-SDK2
mkdir -p build && cd build
cmake ..
make -j$(nproc)
sudo make install
sudo ldconfig
```

`sudo ldconfig` 很重要，不然可能出现动态库加载失败。

## 5. 编译 livox_ros_driver2

```bash
cd <本地路径>/FAST-LIO
mkdir -p ros2_ws/src
cd ros2_ws/src
git clone https://github.com/Livox-SDK/livox_ros_driver2.git

conda deactivate
cd <本地路径>/FAST-LIO/ros2_ws/src/livox_ros_driver2
./build.sh humble
```

加入环境变量：

```bash
echo "source <本地路径>/FAST-LIO/ros2_ws/install/setup.zsh" >> ~/.zshrc
source ~/.zshrc
```

## 6. 编译 FAST_LIO_ROS2

```bash
cd <本地路径>/FAST-LIO
mkdir -p fastlio_ws/src
cd fastlio_ws/src
git clone https://github.com/Ericsii/FAST_LIO_ROS2.git --recursive

conda deactivate
cd <本地路径>/FAST-LIO/fastlio_ws
rm -rf build install log
colcon build --symlink-install --cmake-args -DPython3_EXECUTABLE=/usr/bin/python3
```

加入环境变量：

```bash
echo "source <本地路径>/FAST-LIO/fastlio_ws/install/setup.zsh" >> ~/.zshrc
source ~/.zshrc
```

## 7. 配置文件

编辑：

```bash
<本地路径>/FAST-LIO/ros2_ws/src/livox_ros_driver2/config/MID360_config.json
```

确认以下信息一致：

* `host_net_info` 中的 IP 设为电脑 IP：`192.168.1.50`
* `lidar_configs` 中的 IP 设为雷达 IP：`192.168.1.161`

## 8. 启动顺序

### 终端 A：启动雷达驱动

```bash
conda deactivate
ros2 launch livox_ros_driver2 msg_MID360_launch.py
```

### 终端 B：启动 FAST_LIO

```bash
conda deactivate
ros2 launch fast_lio mapping.launch.py
```

## 9. 常见问题

### 9.1 编译报错找不到 Python 模块

原因通常是 Conda 干扰了系统 Python。解决方法：

* 编译前执行 `conda deactivate`
* 必要时显式指定 `-DPython3_EXECUTABLE=/usr/bin/python3`

### 9.2 动态库找不到

安装 SDK2 后执行：

```bash
sudo ldconfig
```

### 9.3 驱动启动失败 / Bind Failed

检查以下几项：

* 电脑 IP 是否为 `192.168.1.50`
* 雷达 IP 是否为 `192.168.1.161`
* 配置文件中的 IP 是否与真实网络一致
* 网线是否直连正常

### 9.4 版本不匹配

本方案基于：

* Ubuntu 22.04
* ROS 2 Humble

不要按 ROS 1 Noetic 的流程混用。

## 10. 目录建议

```text
<本地路径>/FAST-LIO/
├── Livox-SDK2/
├── ros2_ws/
└── fastlio_ws/
```

## 11. 局域网通信指南（TODO）

> 这一部分先保留方向，后续补成完整联调说明。

* TODO：梳理雷达、电脑、机器人之间的局域网拓扑
* TODO：说明各设备的 IP 分配原则和网段约束
* TODO：补充中间件通信方式与数据流向
* TODO：补充与机器人控制端的消息接口对接方式
* TODO：补充联调时的连通性检查步骤
* TODO：补充最终部署时的安全注意事项

## 12. 备注

本 README 适合直接作为复现文档使用。后续如需扩展，可继续补充：

* 点云保存为 `.pcd` 地图
* 坐标系与机器人模型对齐
* 局域网联调和多机通信细节
