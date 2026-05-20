# 姝ラ4锛氭憚鍍忓ご + YOLO 鐩爣妫€娴嬶紙Day 7-8锛?
## 鐩爣
鏍戣帗娲捐窇 YOLOv8n 瀹炴椂鐩爣妫€娴嬶紝璇嗗埆瑙嗛噹涓殑鐗╀綋骞堕€氳繃 ROS2 鍙戝竷缁撴灉銆?
---

## 4.1 鎽勫儚澶存帴鍏?
```bash
# 鎻掍笂 USB 鎽勫儚澶?# 妫€鏌ユ槸鍚﹁瘑鍒?ls /dev/video*
# 搴旇鐪嬪埌 /dev/video0

# 娴嬭瘯鎷嶇収
sudo apt install -y fswebcam
fswebcam -r 640x480 test.jpg
# 鎶?test.jpg 浼犲埌鐢佃剳涓婄湅鏄惁娓呮櫚
```

---

## 4.2 瀹夎 YOLOv8

```bash
# 瀹夎 ultralytics锛圷OLOv8锛?pip install ultralytics

# 涓嬭浇棰勮缁冩ā鍨嬶紙浼氳嚜鍔ㄤ笅杞?yolov8n.pt锛?python -c "
from ultralytics import YOLO
model = YOLO('yolov8n.pt')  # nano鐗堬紝閫傚悎鏍戣帗娲?print('YOLOv8n 鍔犺浇鎴愬姛')
"

# 娴嬭瘯妫€娴嬮€熷害
python -c "
from ultralytics import YOLO
import time
model = YOLO('yolov8n.pt')
# 棰勭儹
import numpy as np
dummy = np.zeros((640, 480, 3), dtype=np.uint8)
for _ in range(5):
    model(dummy, verbose=False)

start = time.time()
for _ in range(30):
    model(dummy, verbose=False)
elapsed = time.time() - start
print(f'鎺ㄧ悊閫熷害: {30/elapsed:.1f} FPS')
# 鏍戣帗娲? 澶ф鑳借窇 8-15 FPS
"
```

---

## 4.3 ROS2 瑙嗚鑺傜偣

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python smartcar_vision \
  --dependencies rclpy sensor_msgs vision_msgs cv_bridge std_msgs
```

```python
# ~/ros2_ws/src/smartcar_vision/smartcar_vision/camera_node.py
"""
鎽勫儚澶磋妭鐐癸細鍙戝竷鍥惧儚 + YOLO 妫€娴嬬粨鏋?"""

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
        
        # 鎽勫儚澶?        self.cap = cv2.VideoCapture(0)
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
        self.cap.set(cv2.CAP_PROP_FPS, 15)
        
        if not self.cap.isOpened():
            self.get_logger().error('鏃犳硶鎵撳紑鎽勫儚澶?')
            return
        
        # YOLO 妯″瀷
        self.get_logger().info('鍔犺浇 YOLOv8n...')
        self.model = YOLO('yolov8n.pt')
        self.get_logger().info('YOLOv8n 灏辩华')
        
        # 鍏虫敞鐨勭墿浣撶被鍒紙COCO鏁版嵁闆嗙殑绫诲埆ID锛?        self.target_classes = {
            0: '浜?, 39: '鐡跺瓙', 41: '鏉瓙', 44: '鐡跺瓙',
            56: '妞呭瓙', 62: '鐢佃', 63: '绗旇鏈?, 64: '榧犳爣',
            67: '鎵嬫満', 73: '涔?, 75: '閬ユ帶鍣?, 76: '閿洏',
            77: '鎵嬫満', 84: '涔?,
        }
        
        # 鍙戝竷鑰?        self.bridge = CvBridge()
        self.image_pub = self.create_publisher(
            CompressedImage, '/camera/image/compressed', 10)
        self.detection_pub = self.create_publisher(
            String, '/camera/detections', 10)
        
        # 瀹氭椂鍣?        self.timer = self.create_timer(0.1, self.process_frame)  # 10Hz
        
        self.fps_timer = time.time()
        self.frame_count = 0
        
        self.get_logger().info('鎽勫儚澶磋妭鐐瑰氨缁?)
    
    def process_frame(self):
        ret, frame = self.cap.read()
        if not ret:
            return
        
        # FPS 缁熻
        self.frame_count += 1
        if time.time() - self.fps_timer > 5:
            fps = self.frame_count / (time.time() - self.fps_timer)
            self.get_logger().info(f'FPS: {fps:.1f}')
            self.frame_count = 0
            self.fps_timer = time.time()
        
        # YOLO 妫€娴?        results = self.model(frame, verbose=False)
        
        detections = []
        for r in results:
            boxes = r.boxes
            if boxes is not None:
                for box in boxes:
                    cls_id = int(box.cls[0])
                    conf = float(box.conf[0])
                    
                    if conf < 0.5:  # 缃俊搴﹁繃婊?                        continue
                    
                    # 鑾峰彇鍧愭爣
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
                    
                    # 鐢绘
                    cv2.rectangle(frame, (int(x1), int(y1)), 
                                  (int(x2), int(y2)), (0, 255, 0), 2)
                    cv2.putText(frame, f'{name} {conf:.2f}',
                               (int(x1), int(y1) - 5),
                               cv2.FONT_HERSHEY_SIMPLEX, 0.5, 
                               (0, 255, 0), 1)
        
        # 鍙戝竷妫€娴嬬粨鏋?        import json
        msg = String()
        msg.data = json.dumps(detections, ensure_ascii=False)
        self.detection_pub.publish(msg)
        
        # 鍙戝竷鍘嬬缉鍥惧儚
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

## 4.4 缂栧啓 Web 瑙嗛娴侊紙鏂逛究鎵嬫満鏌ョ湅锛?
```python
# ~/ros2_ws/src/smartcar_vision/smartcar_vision/stream_server.py
"""
杞婚噺 HTTP 瑙嗛娴侊紝鎵嬫満/PC娴忚鍣ㄥ彲鐩存帴鏌ョ湅
璁块棶: http://鏍戣帗娲綢P:8080
"""

from http.server import HTTPServer, BaseHTTPRequestHandler
import cv2
import threading
import numpy as np

# 鍏ㄥ眬鍙橀噺锛氭渶鏂扮殑甯э紙鐢?ROS2 鑺傜偣鍐欏叆锛?latest_frame = None
frame_lock = threading.Lock()

def update_frame(frame):
    global latest_frame
    with frame_lock:
        latest_frame = frame.copy()

class StreamHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/':
            # 绠€鍗旽TML椤甸潰
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
            # MJPEG 娴?            self.send_response(200)
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
            # 鍗曞紶鐓х墖
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
            # 杩斿洖妫€娴嬬粨鏋淛SON
            import json
            # (杩欓噷浠?ROS2 璇濋鎷垮埌鏁版嵁锛岀畝鍖栫敤鍏ㄥ眬鍙橀噺)
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            self.wfile.write(b'[]')

def start_stream_server(port=8080):
    server = HTTPServer(('0.0.0.0', port), StreamHandler)
    print(f'瑙嗛娴佸凡鍚姩: http://鏍戣帗娲綢P:{port}')
    server.serve_forever()

if __name__ == '__main__':
    # 鐙珛杩愯娴嬭瘯锛堜笉鎺OS2锛?    cap = cv2.VideoCapture(0)
    def capture():
        global latest_frame
        while True:
            ret, frame = cap.read()
            if ret:
                update_frame(frame)
    
    threading.Thread(target=capture, daemon=True).start()
    threading.Thread(target=start_stream_server, daemon=True).start()
    input('鎸夊洖杞﹂€€鍑?..\n')
```

### 淇敼 setup.py 娣诲姞鍏ュ彛

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

## 4.5 鑷畾涔夌墿浣撹缁冿紙鍙€夛級

濡傛灉 YOLOv8 棰勮缁冩ā鍨嬭瘑鍒笉浜嗕綘鍏冲績鐨勭墿浣擄紝鍙互鑷繁璁粌锛?
```python
# train_custom.py - 鍦ㄧ數鑴戜笂璁粌锛堟爲鑾撴淳澶參锛?from ultralytics import YOLO

# 1. 鍑嗗鏁版嵁闆嗭紙鐢?LabelImg 鏍囨敞锛?#    dataset/
#    鈹溾攢鈹€ images/
#    鈹?  鈹溾攢鈹€ train/
#    鈹?  鈹斺攢鈹€ val/
#    鈹溾攢鈹€ labels/
#    鈹?  鈹溾攢鈹€ train/
#    鈹?  鈹斺攢鈹€ val/
#    鈹斺攢鈹€ data.yaml

# 2. 璁粌
model = YOLO('yolov8n.pt')
model.train(
    data='dataset/data.yaml',
    epochs=100,
    imgsz=640,
    batch=16,
    name='smartcar_custom'
)

# 3. 瀵煎嚭涓?ONNX锛堝彲閫夛紝鍔犻€熸帹鐞嗭級
model.export(format='onnx')
# 澶嶅埗 best.pt 鎴?best.onnx 鍒版爲鑾撴淳
```

---

## 鉁?Day 7-8 楠屾敹鏍囧噯

- [ ] 鎽勫儚澶磋兘姝ｅ父閲囬泦鍥惧儚
- [ ] YOLO 鑳借瘑鍒父瑙佺墿浣擄紙浜恒€佹澂瀛愩€佹墜鏈恒€佷功绛夛級
- [ ] ROS2 璇濋 `/camera/detections` 姝ｅ父鍙戝竷
- [ ] 娴忚鍣ㄦ墦寮€ `http://鏍戣帗娲綢P:8080` 鑳界湅鍒板疄鏃惰棰戞祦
- [ ] 妫€娴嬫鍑嗙‘锛堜汉鍦ㄧ敾闈㈤噷妗嗗埌浜嗭級

### 璋冭瘯

```bash
# 鏌ョ湅妫€娴嬬粨鏋?ros2 topic echo /camera/detections

# 鏌ョ湅鍥惧儚璇濋
ros2 topic hz /camera/image/compressed

# 娴嬭瘯鎽勫儚澶寸嫭绔?python -c "
import cv2
cap = cv2.VideoCapture(0)
ret, frame = cap.read()
cv2.imwrite('test.jpg', frame)
print('鎷嶇収鎴愬姛')
cap.release()
"
```

### 鎬ц兘浼樺寲锛堝鏋?FPS 澶綆锛?
```python
# 鍦?camera_node 涓敼鐢?TensorRT 鍔犻€燂紙Jetson 鎵嶈锛屾爲鑾撴淳涓嶆敮鎸侊級
# 鏍戣帗娲句紭鍖栨柟妗堬細
# 1. 鍑忓皯鍒嗚鲸鐜?self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 320)
self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 240)

# 2. 闄嶄綆妫€娴嬮鐜?self.timer = self.create_timer(0.3, self.process_frame)  # 3Hz

# 3. 浣跨敤鏇村皬鐨勬ā鍨?self.model = YOLO('yolov8n.pt')  # 宸茬粡鏄渶灏忕殑浜?```
