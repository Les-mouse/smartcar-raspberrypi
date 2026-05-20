# 步骤2：树莓派烧系统 + ROS2 安装（Day 3-4）

## 目标
树莓派5运行 Ubuntu 22.04 + ROS2 Humble，能跑第一个 ROS2 节点。

---

## 2.1 烧录 Ubuntu 22.04

### 准备
- 电脑下载 [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
- 64GB TF卡 + 读卡器
- 网线（第一次设置用有线更稳）

### 烧录步骤

1. 打开 Raspberry Pi Imager
2. 选择设备：**Raspberry Pi 5**
3. 选择操作系统：**Other general-purpose OS → Ubuntu → Ubuntu 22.04 LTS (64-bit)**
4. 选择存储：你的 TF卡
5. 点击齿轮图标 ⚙️ 配置：
   - ☑ 开启 SSH（选密码认证）
   - 设置用户名：`pi`  密码：`robot123`
   - 配置 WiFi：你的 WiFi名 + 密码
   - 语言设置：Shanghai
6. 点击 **烧录**，等待完成

### 首次启动

```bash
# 1. TF卡插入树莓派
# 2. 接上电源（树莓派5用 USB-C PD，5V/5A）
# 3. 等2分钟启动

# 4. 在电脑上查找树莓派IP
# Windows PowerShell:
arp -a | findstr "dc-a6-32"   # 树莓派MAC前缀
# 或者路由器管理页面查看

# 5. SSH 连接
ssh pi@192.168.x.x
# 密码: robot123
```

---

## 2.2 初始配置

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 设置时区
sudo timedatectl set-timezone Asia/Shanghai

# 设置 hostname
sudo hostnamectl set-hostname smartcar
echo "127.0.0.1 smartcar" | sudo tee -a /etc/hosts

# 新建机器人用户（可选）
# sudo adduser robot

# 安装常用工具
sudo apt install -y git curl wget vim net-tools htop i2c-tools
```

---

## 2.3 安装 ROS2 Humble

```bash
# 1. 确保 UTF-8 locale
sudo apt install -y locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# 2. 添加 ROS2 源
sudo apt install -y software-properties-common
sudo add-apt-repository universe -y
sudo apt update && sudo apt install -y curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# 3. 安装 ROS2 Humble 桌面版
sudo apt update
sudo apt install -y ros-humble-desktop

# 4. 安装开发工具
sudo apt install -y ros-dev-tools python3-colcon-common-extensions

# 5. 设置环境变量（每次启动自动加载）
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc

# 6. 验证安装
ros2 --version
# 应该输出类似: humble 或 0.x.x
```

---

## 2.4 创建工作空间

```bash
# 创建工作空间目录
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws

# 首次编译（空的工作空间）
colcon build --symlink-install

# 加载工作空间
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc

# 验证
echo $ROS_PACKAGE_PATH
# 应该包含 ~/ros2_ws
```

---

## 2.5 第一个 ROS2 节点：Hello World

### 创建功能包

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python smartcar_base \
  --dependencies rclpy std_msgs
```

### 编写发布者节点

```bash
mkdir -p ~/ros2_ws/src/smartcar_base/smartcar_base
```

```python
# ~/ros2_ws/src/smartcar_base/smartcar_base/talker.py

import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class Talker(Node):
    def __init__(self):
        super().__init__('talker')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(0.5, self.timer_callback)
        self.count = 0
        
    def timer_callback(self):
        msg = String()
        msg.data = f'Hello from smartcar! [{self.count}]'
        self.pub.publish(msg)
        self.get_logger().info(f'发送: {msg.data}')
        self.count += 1

def main():
    rclpy.init()
    node = Talker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 编写监听者节点

```python
# ~/ros2_ws/src/smartcar_base/smartcar_base/listener.py

import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class Listener(Node):
    def __init__(self):
        super().__init__('listener')
        self.sub = self.create_subscription(
            String, 'chatter', self.callback, 10)
        
    def callback(self, msg):
        self.get_logger().info(f'收到: {msg.data}')

def main():
    rclpy.init()
    node = Listener()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 修改 setup.py

编辑 `~/ros2_ws/src/smartcar_base/setup.py`，找到 `entry_points` 部分替换为：

```python
entry_points={
    'console_scripts': [
        'talker = smartcar_base.talker:main',
        'listener = smartcar_base.listener:main',
    ],
},
```

### 编译运行

```bash
cd ~/ros2_ws
colcon build --packages-select smartcar_base --symlink-install
source install/setup.bash

# 开两个终端测试：

# 终端1：运行发布者
ros2 run smartcar_base talker

# 终端2：运行监听者
ros2 run smartcar_base listener

# 你应该看到：
# [INFO] 收到: Hello from smartcar! [0]
# [INFO] 收到: Hello from smartcar! [1]
# ...
```

---

## 2.6 装好必备工具包

```bash
# Python 依赖
pip install pyserial numpy opencv-python flask websockets

# ROS2 常用包
sudo apt install -y \
  ros-humble-cv-bridge \
  ros-humble-vision-msgs \
  ros-humble-nav2-bringup \
  ros-humble-slam-toolbox \
  ros-humble-robot-localization \
  ros-humble-teleop-twist-keyboard

# 串口权限
sudo usermod -a -G dialout $USER
# 重新登录生效
```

---

## ✅ Day 3-4 验收标准

- [ ] 树莓派能 SSH 连接
- [ ] `ros2 --version` 正常
- [ ] talker/listener 能互相收发消息
- [ ] 能 `ros2 topic list` 看到 chatter 话题
- [ ] 串口权限正常 (`ls /dev/ttyUSB* /dev/ttyACM*`)

### 常见问题

| 问题 | 解决 |
|------|------|
| SSH 连不上 | 检查 IP、确认 WiFi 已连、尝试网线 |
| ros2 命令不存在 | `source /opt/ros/humble/setup.bash` |
| 编译报错 | `rosdep update && rosdep install -i --from-path src -y` |
| 找不到 ttyUSB | `sudo usermod -aG dialout $USER` 然后重启 |
| 树莓派过热 | 装散热片+风扇，`vcgencmd measure_temp` |
