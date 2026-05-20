# 姝ラ2锛氭爲鑾撴淳鐑х郴缁?+ ROS2 瀹夎锛圖ay 3-4锛?
## 鐩爣
鏍戣帗娲?杩愯 Ubuntu 22.04 + ROS2 Humble锛岃兘璺戠涓€涓?ROS2 鑺傜偣銆?
---

## 2.1 鐑у綍 Ubuntu 22.04

### 鍑嗗
- 鐢佃剳涓嬭浇 [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
- 64GB TF鍗?+ 璇诲崱鍣?- 缃戠嚎锛堢涓€娆¤缃敤鏈夌嚎鏇寸ǔ锛?
### 鐑у綍姝ラ

1. 鎵撳紑 Raspberry Pi Imager
2. 閫夋嫨璁惧锛?*Raspberry Pi 5**
3. 閫夋嫨鎿嶄綔绯荤粺锛?*Other general-purpose OS 鈫?Ubuntu 鈫?Ubuntu 22.04 LTS (64-bit)**
4. 閫夋嫨瀛樺偍锛氫綘鐨?TF鍗?5. 鐐瑰嚮榻胯疆鍥炬爣 鈿欙笍 閰嶇疆锛?   - 鈽?寮€鍚?SSH锛堥€夊瘑鐮佽璇侊級
   - 璁剧疆鐢ㄦ埛鍚嶏細`pi`  瀵嗙爜锛歚robot123`
   - 閰嶇疆 WiFi锛氫綘鐨?WiFi鍚?+ 瀵嗙爜
   - 璇█璁剧疆锛歋hanghai
6. 鐐瑰嚮 **鐑у綍**锛岀瓑寰呭畬鎴?
### 棣栨鍚姩

```bash
# 1. TF鍗℃彃鍏ユ爲鑾撴淳
# 2. 鎺ヤ笂鐢垫簮锛堟爲鑾撴淳5鐢?USB-C PD锛?V/5A锛?# 3. 绛?鍒嗛挓鍚姩

# 4. 鍦ㄧ數鑴戜笂鏌ユ壘鏍戣帗娲綢P
# Windows PowerShell:
arp -a | findstr "dc-a6-32"   # 鏍戣帗娲綧AC鍓嶇紑
# 鎴栬€呰矾鐢卞櫒绠＄悊椤甸潰鏌ョ湅

# 5. SSH 杩炴帴
ssh pi@192.168.x.x
# 瀵嗙爜: robot123
```

---

## 2.2 鍒濆閰嶇疆

```bash
# 鏇存柊绯荤粺
sudo apt update && sudo apt upgrade -y

# 璁剧疆鏃跺尯
sudo timedatectl set-timezone Asia/Shanghai

# 璁剧疆 hostname
sudo hostnamectl set-hostname smartcar
echo "127.0.0.1 smartcar" | sudo tee -a /etc/hosts

# 鏂板缓鏈哄櫒浜虹敤鎴凤紙鍙€夛級
# sudo adduser robot

# 瀹夎甯哥敤宸ュ叿
sudo apt install -y git curl wget vim net-tools htop i2c-tools
```

---

## 2.3 瀹夎 ROS2 Humble

```bash
# 1. 纭繚 UTF-8 locale
sudo apt install -y locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# 2. 娣诲姞 ROS2 婧?sudo apt install -y software-properties-common
sudo add-apt-repository universe -y
sudo apt update && sudo apt install -y curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# 3. 瀹夎 ROS2 Humble 妗岄潰鐗?sudo apt update
sudo apt install -y ros-humble-desktop

# 4. 瀹夎寮€鍙戝伐鍏?sudo apt install -y ros-dev-tools python3-colcon-common-extensions

# 5. 璁剧疆鐜鍙橀噺锛堟瘡娆″惎鍔ㄨ嚜鍔ㄥ姞杞斤級
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc

# 6. 楠岃瘉瀹夎
ros2 --version
# 搴旇杈撳嚭绫讳技: humble 鎴?0.x.x
```

---

## 2.4 鍒涘缓宸ヤ綔绌洪棿

```bash
# 鍒涘缓宸ヤ綔绌洪棿鐩綍
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws

# 棣栨缂栬瘧锛堢┖鐨勫伐浣滅┖闂达級
colcon build --symlink-install

# 鍔犺浇宸ヤ綔绌洪棿
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc

# 楠岃瘉
echo $ROS_PACKAGE_PATH
# 搴旇鍖呭惈 ~/ros2_ws
```

---

## 2.5 绗竴涓?ROS2 鑺傜偣锛欻ello World

### 鍒涘缓鍔熻兘鍖?
```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python smartcar_base \
  --dependencies rclpy std_msgs
```

### 缂栧啓鍙戝竷鑰呰妭鐐?
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
        self.get_logger().info(f'鍙戦€? {msg.data}')
        self.count += 1

def main():
    rclpy.init()
    node = Talker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 缂栧啓鐩戝惉鑰呰妭鐐?
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
        self.get_logger().info(f'鏀跺埌: {msg.data}')

def main():
    rclpy.init()
    node = Listener()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

### 淇敼 setup.py

缂栬緫 `~/ros2_ws/src/smartcar_base/setup.py`锛屾壘鍒?`entry_points` 閮ㄥ垎鏇挎崲涓猴細

```python
entry_points={
    'console_scripts': [
        'talker = smartcar_base.talker:main',
        'listener = smartcar_base.listener:main',
    ],
},
```

### 缂栬瘧杩愯

```bash
cd ~/ros2_ws
colcon build --packages-select smartcar_base --symlink-install
source install/setup.bash

# 寮€涓や釜缁堢娴嬭瘯锛?
# 缁堢1锛氳繍琛屽彂甯冭€?ros2 run smartcar_base talker

# 缁堢2锛氳繍琛岀洃鍚€?ros2 run smartcar_base listener

# 浣犲簲璇ョ湅鍒帮細
# [INFO] 鏀跺埌: Hello from smartcar! [0]
# [INFO] 鏀跺埌: Hello from smartcar! [1]
# ...
```

---

## 2.6 瑁呭ソ蹇呭宸ュ叿鍖?
```bash
# Python 渚濊禆
pip install pyserial numpy opencv-python flask websockets

# ROS2 甯哥敤鍖?sudo apt install -y \
  ros-humble-cv-bridge \
  ros-humble-vision-msgs \
  ros-humble-nav2-bringup \
  ros-humble-slam-toolbox \
  ros-humble-robot-localization \
  ros-humble-teleop-twist-keyboard

# 涓插彛鏉冮檺
sudo usermod -a -G dialout $USER
# 閲嶆柊鐧诲綍鐢熸晥
```

---

## 鉁?Day 3-4 楠屾敹鏍囧噯

- [ ] 鏍戣帗娲捐兘 SSH 杩炴帴
- [ ] `ros2 --version` 姝ｅ父
- [ ] talker/listener 鑳戒簰鐩告敹鍙戞秷鎭?- [ ] 鑳?`ros2 topic list` 鐪嬪埌 chatter 璇濋
- [ ] 涓插彛鏉冮檺姝ｅ父 (`ls /dev/ttyUSB* /dev/ttyACM*`)

### 甯歌闂

| 闂 | 瑙ｅ喅 |
|------|------|
| SSH 杩炰笉涓?| 妫€鏌?IP銆佺‘璁?WiFi 宸茶繛銆佸皾璇曠綉绾?|
| ros2 鍛戒护涓嶅瓨鍦?| `source /opt/ros/humble/setup.bash` |
| 缂栬瘧鎶ラ敊 | `rosdep update && rosdep install -i --from-path src -y` |
| 鎵句笉鍒?ttyUSB | `sudo usermod -aG dialout $USER` 鐒跺悗閲嶅惎 |
| 鏍戣帗娲捐繃鐑?| 瑁呮暎鐑墖+椋庢墖锛宍vcgencmd measure_temp` |
