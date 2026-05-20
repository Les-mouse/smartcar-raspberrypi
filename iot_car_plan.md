# 🚗 智能小车项目总控 — 对标 7 步教程

> 架构：微信 → OpenClaw(AI大脑) → 树莓派5(ROS2) → Arduino Mega(底层)
> 预算：¥2895 | 周期：14天

## 📊 项目总览

```
微信对话 ──→ OpenClaw ──→ 树莓派5 ──→ Arduino Mega ──→ 电机/传感器
   ↑              ↑            ↑              ↑
 说人话        AI理解      ROS2协调      实时控制(50Hz)
```

## 🗺️ 7 步骤进度

| 步骤 | 天数 | 主题 | 核心产出 | 状态 |
|------|------|------|----------|------|
| 1 | Day 1-2 | 底盘组装 | 电机转起来，编码器读数 | ⚪ |
| 2 | Day 3-4 | 树莓派+ROS2 | talker/listener互通 | ⚪ |
| 3 | Day 5-6 | Arduino通信 | 键盘遥控跑起来 | ⚪ |
| 4 | Day 7-8 | YOLO视觉 | 实时目标检测 | ⚪ |
| 5 | Day 9-10 | 避障导航 | 自主绕障+IMU航向 | ⚪ |
| 6 | Day11-12 | OpenClaw桥接 | 微信控制小车⭐ | ⚪ |
| 7 | Day13-14 | 场景开发 | 巡逻/找东西/配送 | ⚪ |

## 📦 采购清单（¥2895）

### 必买 ¥2445
| 硬件 | 型号 | 单价 | 用途 |
|------|------|------|------|
| 树莓派5 | 8GB | ¥550 | ROS2主控 |
| Arduino Mega | 2560 | ¥55 | 底层控制 |
| 4WD底盘 | 铝合金+编码器 | ¥420 | 车体 |
| TB6612×2 | 双路驱动 | ¥25 | 驱动4电机 |
| USB摄像头 | 1080P 120° | ¥95 | YOLO视觉 |
| 舵机云台×2 | MG996R | ¥70 | 摄像头转动 |
| 超声波×4 | HC-SR04 | ¥8 | 避障 |
| GY-85 | 9轴IMU | ¥45 | 姿态航向 |
| 12V锂电池 | 6800mAh | ¥180 | 动力 |
| LM2596×3 | 降压模块 | ¥10 | 供电 |
| OLED | SSD1306 | ¥25 | 状态显示 |
| 其他 | TF卡/杜邦线/铜柱等 | ¥247 | 配件 |

### 选配 ¥455
| 硬件 | 用途 |
|------|------|
| 红外避障×4 | 防跌落 |
| DHT22 | 温湿度 |
| 3-DOF机械臂 | 简易抓取 |
| WS2812B灯带 | 氛围灯 |
| USB无线网卡 | 增强WiFi |

## 🔧 引脚分配

### Arduino Mega
```
电机:
  前左: AIN1=22 AIN2=23 PWM=6
  前右: BIN1=24 BIN2=25 PWM=7
  后左: AIN1=26 AIN2=27 PWM=8
  后右: BIN1=28 BIN2=29 PWM=9

编码器:
  前左: A=18 前右: A=20
  后左: A=2  后右: A=4

超声波:
  前: trig=30 echo=31
  后: trig=32 echo=33
  左: trig=34 echo=35
  右: trig=36 echo=37

OLED: I2C (SDA=20 SCL=21)
IMU:  I2C (同上)
```

### 树莓派 ↔ Arduino
```
树莓派 GPIO  →  Arduino
  GND(pin6)  →  GND
  TX(pin8)   →  RX1(pin19)
  RX(pin10)  →  TX1(pin18)
```

## 📂 文件结构
```
smartcar/
├── docs/
│   ├── README.md          ← 完整教程
│   ├── 步骤1_底盘组装.md
│   ├── 步骤2_树莓派ROS2.md
│   ├── 步骤3_Arduino控制.md
│   ├── 步骤4_YOLO视觉.md
│   ├── 步骤5_避障导航.md
│   ├── 步骤6_OpenClaw桥接.md
│   ├── 步骤7_场景开发.md
│   └── 采购清单.md
├── firmware/
│   └── car_firmware.ino   ← Arduino固件
├── ros2_ws/src/
│   └── smartcar_base/     ← ROS2功能包
├── robot_bridge/
│   ├── bridge_server.py   ← OpenClaw桥接服务
│   └── scenes/            ← 场景脚本
│       └── patrol_scene.py
└── openclaw_skill/
    └── smartcar_skill.py  ← OpenClaw技能定义
```

## 🎯 最终效果

```
你 → 微信:
  "去巡逻一圈"
  → 小车自动走路线 → YOLO检测异常 → 拍照回微信

  "客厅多少度"
  → DHT22读数 → "客厅26°C，湿度55%"

  "找我的钥匙"
  → 全屋巡逻 → YOLO扫描 → "在茶几上" + 照片

  "把这杯水送到书房"
  → 导航到书房 → 到站播报
```
