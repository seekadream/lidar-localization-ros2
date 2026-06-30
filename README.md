<div align="center">

<h1>lidar_localization_ros2</h1>

<p><strong>Map-based 3D LiDAR localization for ROS 2, Nav2, and repeatable rosbag evaluation.</strong></p>

<p>
  <img alt="ROS 2 Humble" src="https://img.shields.io/badge/ROS%202-Humble-1f5b99">
  <img alt="ROS 2 Jazzy" src="https://img.shields.io/badge/ROS%202-Jazzy-2563eb">
  <img alt="Release v1.1.0" src="https://img.shields.io/badge/release-v1.1.0-2f855a">
  <img alt="Backend NDT OMP" src="https://img.shields.io/badge/default-NDT__OMP-4a5568">
</p>

<p><strong>⚠️ MID-360S 用户请直接跳到 <a href="#mid-360s-ndt-重定位实战">MID-360S 实战章节</a></strong></p>

</div>

---

## MID-360S NDT 重定位实战

这是本 fork 在 MID-360S 雷达上的实际部署方案。

### 硬件

- MID-360S 激光雷达（IP `192.168.0.126`）
- 树莓派 / Jetson 任意一台计算设备
- 主机网口 IP 设为 `192.168.0.5`
- 网线直连

### 工作空间结构

```text
~/
├── lidarloc_ws/          # NDT 重定位（本仓库）
│   └── src/
│       ├── lidar_localization_ros2/
│       └── ndt_omp_ros2/
├── mid360_slam_ws/       # 雷达驱动
│   └── src/
│       └── livox_ros_driver2/
└── mid360_map/           # 地图存放
    └── map.pcd
```

### 1. 克隆并编译

```bash
# 雷达驱动
mkdir -p ~/mid360_slam_ws/src
cd ~/mid360_slam_ws/src
git clone https://github.com/Livox-SDK/livox_ros_driver2.git
cd ~/mid360_slam_ws
source /opt/ros/humble/setup.bash
colcon build

# NDT 重定位
mkdir -p ~/lidarloc_ws/src
cd ~/lidarloc_ws/src
git clone https://github.com/seekadream/lidar-localization-ros2.git
cd lidar_localization_ros2
source scripts/setup_local_env.sh
cd ../../build_ws
colcon build --symlink-install --packages-up-to lidar_localization_ros2
source install/setup.bash
```

### 2. 准备地图

先用 SLAM 建图（推荐 [FAST-LIO-SAM](https://github.com/kahowang/FAST_LIO_SAM)），生成 PCD 文件：

```bash
mkdir -p ~/mid360_map
# 将生成的 PCD 文件复制到此处
cp your_map.pcd ~/mid360_map/map.pcd
```

### 3. 一键启动

```bash
# 需要 sudo 权限设置网口 IP
bash ~/lidarloc_ws/src/lidar_localization_ros2/scripts/start_relocalization.sh
```

或者分步手动启动：

```bash
# 终端1: 发布 odom -> base_link TF
source /opt/ros/humble/setup.bash
ros2 run tf2_ros static_transform_publisher --x 0 --y 0 --z 0 --roll 0 --pitch 0 --yaw 0 --frame-id odom --child-frame-id base_link &

# 终端2: 雷达驱动
source ~/mid360_slam_ws/install/setup.bash
ros2 launch livox_ros_driver2 msg_MID360s_pcl2_launch.py

# 终端3: NDT 重定位
source ~/lidarloc_ws/install/setup.bash
ros2 launch lidar_localization_ros2 mid360_legged_localization.launch.py \
    map_path:=/home/pi/mid360_map/map.pcd \
    cloud_topic:=/livox/lidar \
    imu_topic:=/livox/imu \
    use_imu_preintegration:=true \
    enable_map_odom_tf:=true

# 终端4: RVIZ2
rviz2 -d ~/lidarloc_ws/src/lidar_localization_ros2/rviz/mid360s_localization.rviz
```

### 4. 初始化位姿

1. 在 RVIZ2 工具栏点击 **"2D Pose Estimate"**
2. 在地图上点击你的大致位置，拖动箭头指向朝向
3. NDT 会从初始位姿开始匹配，收敛后自动跟踪

### 5. 实时坐标监控

```bash
python3 pose_monitor.py
# 实时输出 X, Y, Z 坐标和姿态四元数
```

---

## 通用文档（适用于所有雷达）

以下是上游仓库 `rsasaki0109/lidar_localization_ros2` 的使用说明，适用于 Velodyne、Ouster 等标准雷达。

## What This Is

`lidar_localization_ros2` 是一个 ROS 2 包，用于基于 3D 点云地图的定位。提供运行时定位器、Nav2 启动封装、基准测试工具和重定位实验运行器。

默认后端: **NDT_OMP**

## Quick Start（通用）

```text
lidarloc_ws/
  src/
    lidar_localization_ros2/
    ndt_omp_ros2/
```

```bash
mkdir -p ~/lidarloc_ws/src
cd ~/lidarloc_ws/src
git clone https://github.com/rsasaki0109/lidar_localization_ros2.git
cd lidar_localization_ros2
scripts/bootstrap_colcon_workspace.sh --build
source ~/lidarloc_ws/install/setup.bash
```

或手动：

```bash
git clone --branch humble https://github.com/rsasaki0109/ndt_omp_ros2.git src/ndt_omp_ros2
cd ~/lidarloc_ws
source /opt/ros/humble/setup.bash
colcon build --symlink-install --packages-up-to lidar_localization_ros2
```

## Choose A Workflow

| Goal | Command |
|---|---|
| LiDAR localization only | `ros2 launch lidar_localization_ros2 nav2_lidar_localization.launch.py` |
| Full Nav2 wrapper | `ros2 launch lidar_localization_ros2 nav2_navigation.launch.py map_yaml:=/path/to/map.yaml` |
| **MID-360 NDT 重定位** | `ros2 launch lidar_localization_ros2 mid360_legged_localization.launch.py map_path:=/path/to/map.pcd` |
| 30-minute public demo | `source scripts/setup_local_env.sh && scripts/run_public_demo.sh` |

## Generate A Config

```bash
ros2 run lidar_localization_ros2 create_lidar_localization_config.py \
  --profile standalone \
  --map-path /absolute/path/to/map.pcd \
  --output /tmp/lidar_localization.yaml
```

## Frames And TF

默认坐标系: `map` (全局), `odom`, `base_link`

- `/initialpose` 使用 `header.frame_id = map`
- 默认 TF 模式发布 `map -> base_link`
- Nav2 模式使用 `enable_map_odom_tf: true`，需要外部发布 `odom -> base_link`

## Topics

**输入**:
- `/cloud`: `sensor_msgs/PointCloud2`
- `/map` 或 `map_path`: 点云地图
- `/initialpose`: `geometry_msgs/PoseWithCovarianceStamped`
- `/imu`: `sensor_msgs/Imu`（可选）

**输出**:
- `/pcl_pose`: `geometry_msgs/PoseStamped`
- `/localization/pose_with_covariance`: `geometry_msgs/PoseWithCovarianceStamped`
- `/path`: `nav_msgs/Path`
- `/alignment_status`: `diagnostic_msgs/DiagnosticArray`

## Dependencies

- ROS 2 Humble 或 Jazzy
- [ndt_omp_ros2](https://github.com/rsasaki0109/ndt_omp_ros2.git) (humble 分支)
- 可选: [small_gicp](https://github.com/koide3/small_gicp)

## 致谢

原始仓库: [rsasaki0109/lidar_localization_ros2](https://github.com/rsasaki0109/lidar_localization_ros2)
