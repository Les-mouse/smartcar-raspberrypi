# 步骤4：摄像头 + YOLO 目标检测（Day 7-8）

## 目标
树莓派跑 YOLOv8n 实时目标检测，识别视野中的物体并通过 ROS2 发布结果。

---

## 4.1 摄像头接入

```bash
# 插上 USB 摄像头
# 检查是否识别
ls /dev/video*
# 应该看到 /dev/video0

# 测试拍照
sudo apt install -y fswebcam
fswebcam -r 640x480 test.jpg
# 把 test.jpg 传到电脑上看是否清晰
```

---

## 4.2 安装 YOLOv8

```bash
# 安装 ultralytics（YOLOv8）
pip install ultralytics

# 下载预训练模型（会自动下载 yolov8n.pt）
python -c "
from ultralytics import YOLO
model = YOLO('yolov8n.pt')  # nano版，适合树莓派
print('YOLOv8n 加载成功')
"

# 测试检测速度
python -c "
from ultralytics import YOLO
import time
model = YOLO('yolov8n.pt')
# 预热
import numpy as np
dummy = np.zeros((640, 480, 3), dtype=np.uint8)
for _ in range(5):
    model(dummy, verbose=False)

start = time.time()
for _ in range(30):
    model(dummy, verbose=False)
elapsed = time.time() - start
print(f'推理速度: {30/elapsed:.1f} FPS')
# 树莓派5 大概能跑 8-15 FPS
"
```

---

## 4.3 ROS2 视觉节点

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python smartcar_vision \
  --dependencies rclpy sensor_msgs vision_msgs cv_bridge std_msgs
```

```python
# ~/ros2_ws/src/smartcar_vision/smartcar_vision/camera_node.py
"""
摄像头节点：发布图像 + YOLO 检测结果
"""

import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image, CompressedImage
from vision_msgs.msg import Detection2DArray, Detection2D, ObjectHypothesisWithPose
from std_msgs.msg import String
from cv_bridge import CvBridge
import cv2
from ultralytics import YOLO
import numpy as np
import time

class CameraNode(Node):
    def __init__(self):
        super().__init__('camera_node')
        
        # 摄像头
        self.cap = cv2.VideoCapture(0)
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
        self.cap.set(cv2.CAP_PROP_FPS, 15)
        
        if not self.cap.isOpened():
            self.get_logger().error('无法打开摄像头!')
            return
        
        # YOLO 模型
        self.get_logger().info('加载 YOLOv8n...')
        self.model = YOLO('yolov8n.pt')
        self.get_logger().info('YOLOv8n 就绪')
        
        # 关注的物体类别（COCO数据集的类别ID）
        self.target_classes = {
            0: '人', 39: '瓶子', 41: '杯子', 44: '瓶子',
            56: '椅子', 62: '电视', 63: '笔记本', 64: '鼠标',
            67: '手机', 73: '书', 75: '遥控器', 76: '键盘',
            77: '手机', 84: '书',
        }
        
        # 发布者
        self.bridge = CvBridge()
        self.image_pub = self.create_publisher(
            CompressedImage, '/camera/image/compressed', 10)
        self.detection_pub = self.create_publisher(
            String, '/camera/detections', 10)
        
        # 定时器
        self.timer = self.create_timer(0.1, self.process_frame)  # 10Hz
        
        self.fps_timer = time.time()
        self.frame_count = 0
        
        self.get_logger().info('摄像头节点就绪')
    
    def process_frame(self):
        ret, frame = self.cap.read()
        if not ret:
            return
        
        # FPS 统计
        self.frame_count += 1
        if time.time() - self.fps_timer > 5:
            fps = self.frame_count / (time.time() - self.fps_timer)
            self.get_logger().info(f'FPS: {fps:.1f}')
            self.frame_count = 0
            self.fps_timer = time.time()
        
        # YOLO 检测
        results = self.model(frame, verbose=False)
        
        detections = []
        for r in results:
            boxes = r.boxes
            if boxes is not None:
                for box in boxes:
                    cls_id = int(box.cls[0])
                    conf = float(box.conf[0])
                    
                    if conf < 0.5:  # 置信度过滤
                        continue
                    
                    # 获取坐标
                    xyxy = box.xyxy[0].cpu().numpy()
                    x1, y1, x2, y2 = xyxy
                    cx = (x1 + x2) / 2
                    cy = (y1 + y2) / 2
                    
                    name = self.model.names[cls_id]
                    
                    detections.append({
                        'class': name,
                        'confidence': round(conf, 2),
                        'x': round(float(cx), 1),
                        'y': round(float(cy), 1),
                        'width': round(float(x2 - x1), 1),
                        'height': round(float(y2 - y1), 1)
                    })
                    
                    # 画框
                    cv2.rectangle(frame, (int(x1), int(y1)), 
                                  (int(x2), int(y2)), (0, 255, 0), 2)
                    cv2.putText(frame, f'{name} {conf:.2f}',
                               (int(x1), int(y1) - 5),
                               cv2.FONT_HERSHEY_SIMPLEX, 0.5, 
                               (0, 255, 0), 1)
        
        # 发布检测结果
        import json
        msg = String()
        msg.data = json.dumps(detections, ensure_ascii=False)
        self.detection_pub.publish(msg)
        
        # 发布压缩图像
        img_msg = self.bridge.cv2_to_compressed_imgmsg(frame)
        img_msg.header.stamp = self.get_clock().now().to_msg()
        self.image_pub.publish(img_msg)
    
    def destroy_node(self):
        self.cap.release()
        super().destroy_node()

def main():
    rclpy.init()
    node = CameraNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

---

## 4.4 编写 Web 视频流（方便手机查看）

```python
# ~/ros2_ws/src/smartcar_vision/smartcar_vision/stream_server.py
"""
轻量 HTTP 视频流，手机/PC浏览器可直接查看
访问: http://树莓派IP:8080
"""

from http.server import HTTPServer, BaseHTTPRequestHandler
import cv2
import threading
import numpy as np

# 全局变量：最新的帧（由 ROS2 节点写入）
latest_frame = None
frame_lock = threading.Lock()

def update_frame(frame):
    global latest_frame
    with frame_lock:
        latest_frame = frame.copy()

class StreamHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/':
            # 简单HTML页面
            html = """
            <html><head><title>SmartCar View</title></head>
            <body style="margin:0;background:#000">
            <img src="/stream" style="width:100%;max-width:800px">
            </body></html>
            """
            self.send_response(200)
            self.send_header('Content-type', 'text/html; charset=utf-8')
            self.end_headers()
            self.wfile.write(html.encode())
        
        elif self.path == '/stream':
            # MJPEG 流
            self.send_response(200)
            self.send_header('Content-type', 
                'multipart/x-mixed-replace; boundary=frame')
            self.end_headers()
            
            while True:
                with frame_lock:
                    if latest_frame is None:
                        continue
                    frame = latest_frame.copy()
                
                _, jpeg = cv2.imencode('.jpg', frame, 
                    [cv2.IMWRITE_JPEG_QUALITY, 60])
                
                self.wfile.write(b'--frame\r\n')
                self.wfile.write(b'Content-Type: image/jpeg\r\n\r\n')
                self.wfile.write(jpeg.tobytes())
                self.wfile.write(b'\r\n')
                
                import time
                time.sleep(0.1)  # 10 FPS
        
        elif self.path == '/snap':
            # 单张照片
            with frame_lock:
                if latest_frame is None:
                    self.send_error(404)
                    return
                frame = latest_frame.copy()
            
            _, jpeg = cv2.imencode('.jpg', frame)
            self.send_response(200)
            self.send_header('Content-type', 'image/jpeg')
            self.end_headers()
            self.wfile.write(jpeg.tobytes())
        
        elif self.path == '/detections':
            # 返回检测结果JSON
            import json
            # (这里从 ROS2 话题拿到数据，简化用全局变量)
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            self.wfile.write(b'[]')

def start_stream_server(port=8080):
    server = HTTPServer(('0.0.0.0', port), StreamHandler)
    print(f'视频流已启动: http://树莓派IP:{port}')
    server.serve_forever()

if __name__ == '__main__':
    # 独立运行测试（不接ROS2）
    cap = cv2.VideoCapture(0)
    def capture():
        global latest_frame
        while True:
            ret, frame = cap.read()
            if ret:
                update_frame(frame)
    
    threading.Thread(target=capture, daemon=True).start()
    threading.Thread(target=start_stream_server, daemon=True).start()
    input('按回车退出...\n')
```

### 修改 setup.py 添加入口

```python
# ~/ros2_ws/src/smartcar_vision/setup.py
entry_points={
    'console_scripts': [
        'camera_node = smartcar_vision.camera_node:main',
        'stream_server = smartcar_vision.stream_server:main',
    ],
},
```

---

## 4.5 自定义物体训练（可选）

如果 YOLOv8 预训练模型识别不了你关心的物体，可以自己训练：

```python
# train_custom.py - 在电脑上训练（树莓派太慢）
from ultralytics import YOLO

# 1. 准备数据集（用 LabelImg 标注）
#    dataset/
#    ├── images/
#    │   ├── train/
#    │   └── val/
#    ├── labels/
#    │   ├── train/
#    │   └── val/
#    └── data.yaml

# 2. 训练
model = YOLO('yolov8n.pt')
model.train(
    data='dataset/data.yaml',
    epochs=100,
    imgsz=640,
    batch=16,
    name='smartcar_custom'
)

# 3. 导出为 ONNX（可选，加速推理）
model.export(format='onnx')
# 复制 best.pt 或 best.onnx 到树莓派
```

---

## ✅ Day 7-8 验收标准

- [ ] 摄像头能正常采集图像
- [ ] YOLO 能识别常见物体（人、杯子、手机、书等）
- [ ] ROS2 话题 `/camera/detections` 正常发布
- [ ] 浏览器打开 `http://树莓派IP:8080` 能看到实时视频流
- [ ] 检测框准确（人在画面里框到了）

### 调试

```bash
# 查看检测结果
ros2 topic echo /camera/detections

# 查看图像话题
ros2 topic hz /camera/image/compressed

# 测试摄像头独立
python -c "
import cv2
cap = cv2.VideoCapture(0)
ret, frame = cap.read()
cv2.imwrite('test.jpg', frame)
print('拍照成功')
cap.release()
"
```

### 性能优化（如果 FPS 太低）

```python
# 在 camera_node 中改用 TensorRT 加速（Jetson 才行，树莓派不支持）
# 树莓派优化方案：
# 1. 减少分辨率
self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 320)
self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 240)

# 2. 降低检测频率
self.timer = self.create_timer(0.3, self.process_frame)  # 3Hz

# 3. 使用更小的模型
self.model = YOLO('yolov8n.pt')  # 已经是最小的了
```
