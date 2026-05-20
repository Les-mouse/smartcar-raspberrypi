# 姝ラ6锛歄penClaw AI澶ц剳 + 寰俊鎺у埗 猸愶紙Day 11-12锛?
## 鐩爣
浣犲湪寰俊閲岃窡灏忚溅璇磋瘽锛屽皬杞﹁兘鍚噦銆佹墽琛屼换鍔°€佸洖澶嶇粨鏋溿€?
---

## 6.1 鏁翠綋閫氫俊鏋舵瀯

```
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?  浣犵殑寰俊       鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?       鈫?(寰俊API)
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹? 浣犵殑鐢佃剳       鈹? 鈫?宸茬粡璺戠潃 OpenClaw
鈹? OpenClaw      鈹?鈹? 鈹溾攢 鐞嗚В鎰忓浘   鈹?鈹? 鈹溾攢 浠诲姟瑙勫垝   鈹?鈹? 鈹斺攢 鎵цSkill  鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?       鈫?WiFi / WebSocket
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹? 鏍戣帗娲?        鈹?鈹? openclaw_bridge鈹? 鈫?鎺ユ敹鎸囦护锛岃繑鍥炵粨鏋?鈹?      鈫?        鈹?鈹?  ROS2 缃戠粶     鈹?鈹?      鈫?        鈹?鈹? Arduino Mega   鈹? 鈫?瀹為檯鎺у埗
鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

**鏍稿績鎬濊矾**锛歄penClaw 閫氳繃 Skill 璋冪敤涓€涓?WebSocket API 鈫?鏍戣帗娲句笂璺?bridge 鈫?杞垚 ROS2 璇濋/鏈嶅姟 鈫?鎺у埗灏忚溅銆?
---

## 6.2 鏍戣帗娲剧锛歐ebSocket Bridge Server

```bash
# 鍦ㄦ爲鑾撴淳涓婂畨瑁呬緷璧?pip install websockets asyncio
```

```python
# ~/robot_bridge/bridge_server.py
"""
OpenClaw Bridge Server
鐩戝惉 WebSocket锛屾帴鏀禣penClaw鍙戞潵鐨勬寚浠わ紝鎺у埗ROS2鏈哄櫒浜?"""

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
        
        # 鍚姩 ROS2 鐩戝惉绾跨▼
        self.ros_thread = threading.Thread(target=self.ros_listener, daemon=True)
        self.ros_thread.start()
    
    def ros_listener(self):
        """鐩戝惉 ROS2 璇濋锛屾洿鏂扮姸鎬?""
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
        """鎵цROS2鍛戒护"""
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
        return {'error': f'鏈煡鍔ㄤ綔: {action}'}
    
    def cmd_move(self, params):
        distance = params.get('distance', 0.5)  # 绫?        speed = params.get('speed', 0.2)
        duration = distance / speed
        cmd = f'ros2 topic pub /cmd_vel geometry_msgs/msg/Twist ' \
              f'"{{linear: {{x: {speed}}}, angular: {{z: 0.0}}}}" -1'
        subprocess.run(cmd, shell=True, timeout=1)
        time.sleep(duration)
        self.cmd_stop({})
        return {'action': 'move', 'distance': distance, 'status': 'done'}
    
    def cmd_rotate(self, params):
        angle = params.get('angle', 90)  # 搴?        speed = params.get('speed', 0.5)
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
        """鎷嶇収骞惰繑鍥炲浘鐗嘦RL"""
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
        return {'error': '鎷嶇収澶辫触'}
    
    def cmd_scan(self, params):
        """鎵弿瑙嗛噹涓殑鐗╀綋"""
        return {
            'action': 'scan',
            'objects': self.last_detections,
            'count': len(self.last_detections)
        }
    
    def cmd_patrol(self, params):
        """鍚姩鑷姩宸￠€?""
        waypoints = params.get('waypoints', [])
        subprocess.run(
            'ros2 run smartcar_nav patrol &',
            shell=True, timeout=1)
        self.mode = 'patrol'
        return {'action': 'patrol', 'status': 'started', 'waypoints': waypoints}
    
    def cmd_track(self, params):
        """璺熻釜鎸囧畾鐗╀綋"""
        target = params.get('target', 'person')
        # 閫氳繃鍙傛暟鏈嶅姟鍣ㄨ缃洰鏍?        subprocess.run(
            f'ros2 param set /avoidance target_object "{target}"',
            shell=True, timeout=1)
        self.mode = 'track'
        return {'action': 'track', 'target': target, 'status': 'tracking'}
    
    def cmd_say(self, params):
        """TTS 璇煶鎾姤"""
        text = params.get('text', '浣犲ソ')
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

# ===== WebSocket 鏈嶅姟 =====

bridge = RobotBridge()

async def handle_client(websocket):
    """澶勭悊 OpenClaw 鍙戞潵鐨?WebSocket 杩炴帴"""
    client_ip = websocket.remote_address
    print(f"OpenClaw 宸茶繛鎺? {client_ip}")
    
    # 鍙戦€佹杩庢秷鎭?    await websocket.send(json.dumps({
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
                
                print(f"鏀跺埌鎸囦护: {action} {params}")
                
                # 鎵ц鍛戒护
                result = bridge.execute_command(action, params)
                
                # 杩斿洖缁撴灉
                response = {
                    'type': 'result',
                    'request_id': data.get('request_id', ''),
                    **result
                }
                await websocket.send(json.dumps(response, ensure_ascii=False))
                
            except json.JSONDecodeError:
                await websocket.send(json.dumps({
                    'type': 'error',
                    'message': '鏃犳晥鐨凧SON'
                }))
            except Exception as e:
                await websocket.send(json.dumps({
                    'type': 'error',
                    'message': str(e)
                }))
    except websockets.exceptions.ConnectionClosed:
        print(f"OpenClaw 鏂紑: {client_ip}")

async def main():
    # TTS 瀹夎鎻愮ず
    print("鎻愮ず: sudo apt install espeak-ng 鏉ュ畨瑁?TTS")
    
    # 鍚姩 WebSocket 鏈嶅姟
    ip = get_ip()
    port = 8765
    print(f"Bridge Server 鍚姩: ws://{ip}:{port}")
    
    async with websockets.serve(handle_client, "0.0.0.0", port):
        await asyncio.Future()  # 姘镐箙杩愯

if __name__ == '__main__':
    asyncio.run(main())
```

### 鍚姩 Bridge

```bash
# 鏍戣帗娲句笂
cd ~/robot_bridge
python3 bridge_server.py
# 杈撳嚭: Bridge Server 鍚姩: ws://192.168.x.x:8765
```

---

## 6.3 OpenClaw Skill锛氭満鍣ㄤ汉鎺у埗

杩欐槸鏈€鍏抽敭鐨勯儴鍒嗭紒鍦ㄤ綘鐨勭數鑴戜笂锛圤penClaw宸茶繍琛岋級锛?
```bash
# 鍒涘缓 skill 鐩綍
mkdir -p ~/.openclaw/workspace/skills/smartcar-control
```

### SKILL.md

```yaml
# ~/.openclaw/workspace/skills/smartcar-control/SKILL.md
---
name: smartcar-control
description: 鎺у埗鏅鸿兘灏忚溅鈥斺€斿贰閫汇€佹壘涓滆タ銆侀€佺墿鍝併€佺幆澧冨贰妫€銆傞€氳繃WebSocket杩炴帴鏍戣帗娲句笂鐨刡ridge_server銆?user-invocable: true
---

# 鏅鸿兘灏忚溅鎺у埗

## 杩炴帴淇℃伅

灏忚溅 bridge 杩愯鍦ㄦ爲鑾撴淳涓婏細
- WebSocket: `ws://鏍戣帗娲綢P:8765`
- 瑙嗛娴? `http://鏍戣帗娲綢P:8080/stream`

## 鍙敤鍔ㄤ綔

1. **move(distance, speed)** 鈥?鍓嶈繘/鍚庨€€
   - 鐢ㄦ硶: `move distance=0.5` 鍓嶈繘50cm锛宍move distance=-0.3` 鍚庨€€30cm

2. **rotate(angle)** 鈥?鍘熷湴鏃嬭浆
   - 鐢ㄦ硶: `rotate angle=90` 鍙宠浆90掳锛宍rotate angle=-45` 宸﹁浆45掳

3. **snap()** 鈥?鎷嶇収骞惰繑鍥炴娴嬪埌鐨勭墿浣?
4. **scan()** 鈥?鎵弿瑙嗛噹涓殑鐗╀綋

5. **patrol(waypoints)** 鈥?鑷姩宸￠€?   - waypoints: 宸￠€昏矾绾跨偣鍒楄〃锛屽 ["瀹㈠巺", "鍗у"]

6. **track(target)** 鈥?璺熻釜鎸囧畾鐗╀綋
   - 鐢ㄦ硶: `track target="浜?` 璺熻釜浜猴紝`track target="鎵嬫満"`

7. **stop()** 鈥?鎬ュ仠

8. **say(text)** 鈥?璇煶鎾姤

## 璋冪敤鏂瑰紡

閫氳繃鎵ц鍛戒护璋冪敤锛圵ebSocket 閫氫俊浣跨敤 Python 鑴氭湰锛?

```bash
# 鎵€鏈夊懡浠ら€氳繃 bridge_client.py 鍙戦€?python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py <action> <params_json>
```

绀轰緥:
```bash
# 鍓嶈繘50cm
python bridge_client.py move '{"distance": 0.5}'

# 鎷嶇収鐪嬫湁浠€涔?python bridge_client.py snap '{}'

# 璺熻釜浜?python bridge_client.py track '{"target": "浜?}'
```

## 甯歌鍦烘櫙

### 宸￠€?鐢ㄦ埛璇?鍘诲贰閫? 鈫?rotate 360掳鎵弿 鈫?璧伴璁捐矾绾?鈫?鏈夊紓甯告媿鐓у彂寰俊

### 鎵句笢瑗?鐢ㄦ埛璇?鎵句竴涓嬮挜鍖? 鈫?鍚姩 patrol 妯″紡 鈫?YOLO妫€娴嬪埌閽ュ寵 鈫?snap 鈫?鍋滄 鈫?鍙戦€佸浘鐗?浣嶇疆

### 閫佷笢瑗?鐢ㄦ埛璇?鎶婅繖涓€佸埌涔︽埧" 鈫?move forward 鈫?鍒颁功鎴?鈫?say "鍒颁簡"

### 鐜妫€鏌?鐢ㄦ埛璇?鐪嬬湅娓╁害" 鈫?璇诲彇 DHT22 鈫?鍥炲"26掳C"
```

### bridge_client.py

```python
# ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py
"""
OpenClaw 鈫?鏍戣帗娲?Bridge WebSocket 瀹㈡埛绔?鍦?OpenClaw Skill 涓璋冪敤
"""

import asyncio
import websockets
import json
import sys
import os

# 閰嶇疆锛氭爲鑾撴淳 IP锛堜慨鏀逛负浣犵殑鏍戣帗娲綢P锛?ROBOT_IP = os.environ.get('SMARTCAR_IP', '192.168.1.100')
BRIDGE_PORT = 8765
TIMEOUT = 10  # 瓒呮椂绉掓暟

async def send_command(action, params=None, request_id=''):
    """鍙戦€佹寚浠ゅ苟绛夊緟缁撴灉"""
    if params is None:
        params = {}
    
    uri = f"ws://{ROBOT_IP}:{BRIDGE_PORT}"
    
    try:
        async with websockets.connect(uri, close_timeout=5) as ws:
            # 鍙戦€佹寚浠?            msg = {
                'action': action,
                'params': params,
                'request_id': request_id
            }
            await ws.send(json.dumps(msg))
            
            # 绛夊緟鍝嶅簲
            response = await asyncio.wait_for(ws.recv(), timeout=TIMEOUT)
            return json.loads(response)
    
    except asyncio.TimeoutError:
        return {'error': '鎸囦护瓒呮椂锛屽皬杞﹀彲鑳界绾?}
    except Exception as e:
        return {'error': f'杩炴帴澶辫触: {str(e)}'}

def run_command(action, params=None):
    """鍚屾鍖呰鍣?""
    return asyncio.run(send_command(action, params))

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("鐢ㄦ硶: bridge_client.py <action> [params_json]")
        print("绀轰緥: bridge_client.py move '{\"distance\": 0.5}'")
        sys.exit(1)
    
    action = sys.argv[1]
    params = {}
    if len(sys.argv) > 2:
        try:
            params = json.loads(sys.argv[2])
        except json.JSONDecodeError:
            print(f"鍙傛暟JSON鏍煎紡閿欒: {sys.argv[2]}")
            sys.exit(1)
    
    result = run_command(action, params)
    print(json.dumps(result, ensure_ascii=False, indent=2))
```

---

## 6.4 娴嬭瘯鑱旈€?
```bash
# 1. 鍦ㄦ爲鑾撴淳涓婂惎鍔?bridge
python3 ~/robot_bridge/bridge_server.py
# 鎻愮ず: Bridge Server 鍚姩: ws://192.168.1.xxx:8765

# 2. 鍦ㄧ數鑴戜笂锛圤penClaw杩欒竟锛夋祴璇?# 鍏堣缃爲鑾撴淳IP
export SMARTCAR_IP=192.168.1.xxx

# 娴嬭瘯鎷嶇収
python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py snap '{}'

# 娴嬭瘯绉诲姩
python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py move '{"distance": 0.3}'

# 娴嬭瘯鎵弿
python ~/.openclaw/workspace/skills/smartcar-control/bridge_client.py scan '{}'

# 搴旇杩斿洖:
# {
#   "action": "scan",
#   "objects": [{"class": "浜?, "confidence": 0.85, ...}],
#   "count": 1
# }
```

---

## 6.5 鍦?OpenClaw 涓敞鍐?Skill

```bash
# OpenClaw 浼氳嚜鍔ㄥ姞杞?skills 鐩綍涓嬬殑 SKILL.md
# 閲嶅惎 OpenClaw 鎴栭噸鏂板姞杞介厤缃?
# 妫€鏌?skill 鏄惁鍔犺浇
openclaw skills list
# 搴旇鑳界湅鍒?smartcar-control

# 娴嬭瘯锛氱洿鎺ュ湪 OpenClaw 鑱婂ぉ涓紙鎴栭€氳繃寰俊锛?# "甯垜鐪嬩笅杞﹀墠闈㈡湁浠€涔?
```

---

## 鉁?Day 11-12 楠屾敹鏍囧噯

- [ ] 鐢佃剳鑳?ping 閫氭爲鑾撴淳
- [ ] `bridge_server.py` 姝ｅ父杩愯
- [ ] `bridge_client.py snap` 杩斿洖妫€娴嬬粨鏋?- [ ] `bridge_client.py move '{"distance":0.3}'` 杞﹁兘鍔?- [ ] 寰俊鍙戞寚浠?鈫?OpenClaw 鈫?杞︽湁鍙嶅簲

### 鎺掗敊

| 闂 | 鎺掓煡 |
|------|------|
| WebSocket 杩炰笉涓?| 妫€鏌?IP銆佺鍙ｃ€侀槻鐏 `sudo ufw allow 8765` |
| 鎸囦护娌″弽搴?| 妫€鏌?ROS2 鑺傜偣鏄惁鍦ㄨ繍琛屻€佽瘽棰樻槸鍚︽纭?|
| 杩斿洖閿欒JSON | 妫€鏌?Python 鐗堟湰 (闇€瑕?3.7+) |
| OpenClaw 鎵句笉鍒?skill | 妫€鏌?SKILL.md 璺緞鍜屾牸寮?|
