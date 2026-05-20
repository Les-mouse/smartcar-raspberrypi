# 步骤6：OpenClaw AI大脑 + 微信控制 ⭐（Day 11-12）

## 目标
你在微信里跟小车说话，小车能听懂、执行任务、回复结果。

---

## 6.1 整体通信架构

```
┌────────────────┐
│   你的微信       │
└──────┬─────────┘
       ↓ (微信API)
┌────────────────┐
│  你的电脑       │  ← 已经跑着 OpenClaw
│  OpenClaw      │
│  ├─ 理解意图   │
│  ├─ 任务规划   │
│  └─ 执行Skill  │
└──────┬─────────┘
       ↓ WiFi / WebSocket
┌────────────────┐
│  树莓派5        │
│  openclaw_bridge│  ← 接收指令，返回结果
│       ↓         │
│   ROS2 网络     │
│       ↓         │
│  Arduino Mega   │  ← 实际控制
└────────────────┘
```

**核心思路**：OpenClaw 通过 Skill 调用一个 WebSocket API → 树莓派上跑 bridge → 转成 ROS2 话题/服务 → 控制小车。

---

## 6.2 树莓派端：WebSocket Bridge Server

```bash
# 在树莓派上安装依赖
pip install websockets asyncio
```

```python
# ~/robot_bridge/bridge_server.py
"""
OpenClaw Bridge Server
监听 WebSocket，接收OpenClaw发来的指令，控制ROS2机器人
"""

import asyncio
import websockets
import json
import subprocess
import threading
import time
import http.server
import socketserver

class RobotBridge:
    def __init__(self):
        self.mode = 'idle'
        self.last_image = None
        self.last_detections = []
        self.pose = {'x': 0, 'y': 0, 'theta': 0}
        self.battery = 100
        
        # 启动 ROS2 监听线程
        self.ros_thread = threading.Thread(target=self.ros_listener, daemon=True)
        self.ros_thread.start()
    
    def ros_listener(self):
        """监听 ROS2 话题，更新状态"""
        import rclpy
        from rclpy.node import Node
        from std_msgs.msg import String
        
        class Listener(Node):
            def __init__(self, bridge):
                super().__init__('bridge_listener')
                self.bridge = bridge
                self.sub = self.create_subscription(
                    String, '/camera/detections', self.det_callback, 10)
            
            def det_callback(self, msg):
                try:
                    self.bridge.last_detections = json.loads(msg.data)
                except:
                    pass
        
        rclpy.init()
        node = Listener(self)
        rclpy.spin(node)
    
    def execute_command(self, action, params):
        """执行ROS2命令"""
        cmd_map = {
            'move': self.cmd_move,
            'rotate': self.cmd_rotate,
            'stop': self.cmd_stop,
            'snap': self.cmd_snap,
            'scan': self.cmd_scan,
            'patrol': self.cmd_patrol,
            'track': self.cmd_track,
            'say': self.cmd_say,
        }
        
        func = cmd_map.get(action)
        if func:
            return func(params)
        return {'error': f'未知动作: {action}'}
    
    def cmd_move(self, params):
        distance = params.get('distance', 0.5)  # 米
        speed = params.get('speed', 0.2)
        duration = distance / speed
        cmd = f'ros2 topic pub /cmd_vel geometry_msgs/msg/Twist ' \
              f'"{{linear: {{x: {speed}}}, angular: {{z: 0.0}}}}" -1'
        subprocess.run(cmd, shell=True, timeout=1)
        time.sleep(duration)
        self.cmd_stop({})
        return {'action': 'move', 'distance': distance, 'status': 'done'}
    
    def cmd_rotate(self, params):
        angle = params.get('angle', 90)  # 度
        speed = params.get('speed', 0.5)
        duration = abs(angle) / 180.0 / speed
        direction = 1 if angle > 0 else -1
        cmd = f'ros2 topic pub /cmd_vel geometry_msgs/msg/Twist ' \
              f'"{{linear: {{x: 0.0}}, angular: {{z: {direction * speed}}}}}" -1'
        subprocess.run(cmd, shell=True, timeout=1)
        time.sleep(duration)
        self.cmd_stop({})
        return {'action': 'rotate', 'angle': angle, 'status': 'done'}
    
    def cmd_stop(self, params):
        subprocess.run(
            'ros2 topic pub /cmd_vel geometry_msgs/msg/Twist '
            '"{linear: {x: 0.0}, angular: {z: 0.0}}" -1',
            shell=True, timeout=1)
        return {'action': 'stop', 'status': 'stopped'}
    
    def cmd_snap(self, params):
        """拍照并返回图片URL"""
        import cv2
        cap = cv2.VideoCapture(0)
        ret, frame = cap.read()
        if ret:
            cv2.imwrite('/tmp/snap.jpg', frame)
            cap.release()
            return {
                'action': 'snap',
                'image_url': f'http://{get_ip()}:8080/snap',
                'detections': self.last_detections
            }
        cap.release()
        return {'error': '拍照失败'}
    
    def cmd_scan(self, params):
        """扫描视野中的物体"""
        return {
            'action': 'scan',
            'objects': self.last_detections,
            'count': len(self.last_detections)
        }
    
    def cmd_patrol(self, params):
        """启动自动巡逻"""
        waypoints = params.get('waypoints', [])
        subprocess.run(
            'ros2 run smartcar_nav patrol &',
            shell=True, timeout=1)
        self.mode = 'patrol'
        return {'action': 'patrol', 'status': 'started', 'waypoints': waypoints}
    
    def cmd_track(self, params):
        """跟踪指定物体"""
        target = params.get('target', 'person')
        # 通过参数服务器设置目标
        subprocess.run(
            f'ros2 param set /avoidance target_object "{target}"',
            shell=True, timeout=1)
        self.mode = 'track'
        return {'action': 'track', 'target': target, 'status': 'tracking'}
    
    def cmd_say(self, params):
        """TTS 语音播报"""
        text = params.get('text', '你好')
        subprocess.run(f'espeak-ng "{text}" 2>/dev/null || '
                       f'echo "{text}" | festival --tts 2>/dev/null || '
                       f'echo "TTS not available"',
                       shell=True, timeout=5)
        return {'action': 'say', 'text': text}

def get_ip():
    import socket
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        s.connect(('8.8.8.8', 80))
        ip = s.getsockname()[0]
    except:
        ip = '127.0.0.1'
    finally:
        s.close()
    return ip

# ===== WebSocket 服务 =====

bridge = RobotBridge()

async def handle_client(websocket):
    """处理 OpenClaw 发来的 WebSocket 连接"""
    client_ip = websocket.remote_address
    print(f"OpenClaw 已连接: {client_ip}")
    
    # 发送欢迎消息
    await websocket.send(json.dumps({
        'type': 'ready',
        'robot': 'SmartCar',
        'status': bridge.mode,
        'ip': get_ip()
    }, ensure_ascii=False))
    
    try:
        async for message in websocket:
            try:
                data = json.loads(message)
                action = data.get('action', '')
                params = data.get('params', {})
                
                print(f"收到指令: {action} {params}")
                
                # 执行命令
                result = bridge.execute_command(action, params)
                
                # 返回结果
                response = {
                    'type': 'result',
                    'request_id': data.get('request_id', ''),
                    **result
                }
                await websocket.send(json.dumps(response, ensure_ascii=False))
                
            except json.JSONDecodeError:
                await websocket.send(json.dumps({
                    'type': 'error',
                    'message': '无效的JSON'
                }))
            except Exception as e:
                await websocket.send(json.dumps({
                    'type': 'error',
                    'message': str(e)
                }))
    except websockets.exceptions.ConnectionClosed:
        print(f"OpenClaw 断开: {client_ip}")

async def main():
    # TTS 安装提示
    print("提示: sudo apt install espeak-ng 来安装 TTS")
    
    # 启动 WebSocket 服务
    ip = get_ip()
    port = 8765
    print(f"Bridge Server 启动: ws://{ip}:{port}")
    
    async with websockets.serve(handle_client, "0.0.0.0", port):
        await asyncio.Future()  # 永久运行

if __name__ == '__main__':
    asyncio.run(main())
```

### 启动 Bridge

```bash
# 树莓派上
cd ~/robot_bridge
python3 bridge_server.py
# 输出: Bridge Server 启动: ws://192.168.x.x:8765
```

---

## 6.3 OpenClaw Skill：机器人控制

这是最关键的部分！在你的电脑上（OpenClaw已运行）：

```bash
# 创建 skill 目录
mkdir -p ~/.openclaw/workspace/skills/smartcar-control
```

### SKILL.md

```yaml
# ~/.openclaw/workspace/skills/smartcar-control/SKILL.md
---
name: smartcar-control
description: 控制智能小车——巡逻、找东西、送物品、环境巡检。通过WebSocket连接树莓派上的bridge_server。
user-invocable: true
---

# 智能小车控制

## 连接信息

小车 bridge 运行在树莓派上：
- WebSocket: `ws://树莓派IP:8765`
- 视频流: `http://树莓派IP:8080/stream`

## 可用动作

1. **move(distance, speed)** — 前进/后退
   - 用法: `move distance=0.5` 前进50cm，`move distance=-0.3` 后退30cm

2. **rotate(angle)** — 原地旋转
   - 用法: `rotate angle=90` 右转90°，`rotate angle=-45` 左转45°

3. **snap()** — 拍照并返回检测到的物体

4. **scan()** — 扫描视野中的物体

5. **patrol(waypoints)** — 自动巡逻
   - waypoints: 巡逻路线点列表，如 ["客厅", "卧室"]

6. **track(target)** — 跟踪指定物体
   - 用法: `track target="人"` 跟踪人，`track target="手机"`

7. **stop()** — 急停

8. **say(text)** — 语音播报

## 调用方式

通过执行命令调用（WebSocket 通信使用 Python 脚本）:

```bash
# 所有命令通过 bridge_client.py 发送
python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py <action> <params_json>
```

示例:
```bash
# 前进50cm
python bridge_client.py move '{"distance": 0.5}'

# 拍照看有什么
python bridge_client.py snap '{}'

# 跟踪人
python bridge_client.py track '{"target": "人"}'
```

## 常见场景

### 巡逻
用户说"去巡逻" → rotate 360°扫描 → 走预设路线 → 有异常拍照发微信

### 找东西
用户说"找一下钥匙" → 启动 patrol 模式 → YOLO检测到钥匙 → snap → 停止 → 发送图片+位置

### 送东西
用户说"把这个送到书房" → move forward → 到书房 → say "到了"

### 环境检查
用户说"看看温度" → 读取 DHT22 → 回复"26°C"
```

### bridge_client.py

```python
# ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py
"""
OpenClaw → 树莓派 Bridge WebSocket 客户端
在 OpenClaw Skill 中被调用
"""

import asyncio
import websockets
import json
import sys
import os

# 配置：树莓派 IP（修改为你的树莓派IP）
ROBOT_IP = os.environ.get('SMARTCAR_IP', '192.168.1.100')
BRIDGE_PORT = 8765
TIMEOUT = 10  # 超时秒数

async def send_command(action, params=None, request_id=''):
    """发送指令并等待结果"""
    if params is None:
        params = {}
    
    uri = f"ws://{ROBOT_IP}:{BRIDGE_PORT}"
    
    try:
        async with websockets.connect(uri, close_timeout=5) as ws:
            # 发送指令
            msg = {
                'action': action,
                'params': params,
                'request_id': request_id
            }
            await ws.send(json.dumps(msg))
            
            # 等待响应
            response = await asyncio.wait_for(ws.recv(), timeout=TIMEOUT)
            return json.loads(response)
    
    except asyncio.TimeoutError:
        return {'error': '指令超时，小车可能离线'}
    except Exception as e:
        return {'error': f'连接失败: {str(e)}'}

def run_command(action, params=None):
    """同步包装器"""
    return asyncio.run(send_command(action, params))

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("用法: bridge_client.py <action> [params_json]")
        print("示例: bridge_client.py move '{\"distance\": 0.5}'")
        sys.exit(1)
    
    action = sys.argv[1]
    params = {}
    if len(sys.argv) > 2:
        try:
            params = json.loads(sys.argv[2])
        except json.JSONDecodeError:
            print(f"参数JSON格式错误: {sys.argv[2]}")
            sys.exit(1)
    
    result = run_command(action, params)
    print(json.dumps(result, ensure_ascii=False, indent=2))
```

---

## 6.4 测试联通

```bash
# 1. 在树莓派上启动 bridge
python3 ~/robot_bridge/bridge_server.py
# 提示: Bridge Server 启动: ws://192.168.1.xxx:8765

# 2. 在电脑上（OpenClaw这边）测试
# 先设置树莓派IP
export SMARTCAR_IP=192.168.1.xxx

# 测试拍照
python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py snap '{}'

# 测试移动
python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py move '{"distance": 0.3}'

# 测试扫描
python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py scan '{}'

# 应该返回:
# {
#   "action": "scan",
#   "objects": [{"class": "人", "confidence": 0.85, ...}],
#   "count": 1
# }
```

---

## 6.5 在 OpenClaw 中注册 Skill

```bash
# OpenClaw 会自动加载 skills 目录下的 SKILL.md
# 重启 OpenClaw 或重新加载配置

# 检查 skill 是否加载
openclaw skills list
# 应该能看到 smartcar-control

# 测试：直接在 OpenClaw 聊天中（或通过微信）
# "帮我看下车前面有什么"
```

---

## ✅ Day 11-12 验收标准

- [ ] 电脑能 ping 通树莓派
- [ ] `bridge_server.py` 正常运行
- [ ] `bridge_client.py snap` 返回检测结果
- [ ] `bridge_client.py move '{"distance":0.3}'` 车能动
- [ ] 微信发指令 → OpenClaw → 车有反应

### 排错

| 问题 | 排查 |
|------|------|
| WebSocket 连不上 | 检查 IP、端口、防火墙 `sudo ufw allow 8765` |
| 指令没反应 | 检查 ROS2 节点是否在运行、话题是否正确 |
| 返回错误JSON | 检查 Python 版本 (需要 3.7+) |
| OpenClaw 找不到 skill | 检查 SKILL.md 路径和格式 |
