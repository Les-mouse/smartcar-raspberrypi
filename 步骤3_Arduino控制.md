# 步骤3：Arduino底层通信 + 键盘遥控（Day 5-6）

## 目标
树莓派通过串口控制 Arduino，能用键盘遥控小车跑起来。

---

## 3.1 树莓派 ↔ Arduino 串口连接

### 接线
```
树莓派5                    Arduino Mega
  GND (pin 6)    ──────→  GND
  TX  (pin 8)    ──────→  RX1 (pin 19) ← 实际连 RX
  RX  (pin 10)   ──────→  TX1 (pin 18) ← 实际连 TX
```

> ⚠️ 树莓派 GPIO 是 3.3V，Arduino 是 5V。大部分 Arduino Mega 兼容 3.3V 逻辑，如果不放心可以加一个电平转换模块（¥3）。

---

## 3.2 Arduino 固件：串口指令控制

把这份代码上传到 Arduino Mega：

```cpp
// car_firmware.ino - 完整小车固件
// 通过串口接收指令，控制电机 + 读取传感器

#include <Wire.h>
#include <Adafruit_SSD1306.h>  // OLED 库

// ===== 电机引脚 =====
struct Motor {
  int in1, in2, pwm;
};
Motor motors[4] = {
  {22, 23, 6},   // 前左
  {24, 25, 7},   // 前右
  {26, 27, 8},   // 后左
  {28, 29, 9}    // 后右
};

// ===== 编码器 =====
volatile long encoder_count[4] = {0, 0, 0, 0};
const int encoder_pins[4] = {18, 20, 2, 4};  // A相

// ===== 超声波 =====
struct Ultrasonic {
  int trig, echo;
  float distance;
};
Ultrasonic us_sensors[4] = {
  {30, 31, 0},  // 前
  {32, 33, 0},  // 后
  {34, 35, 0},  // 左
  {36, 37, 0}   // 右
};

// ===== IMU (GY-85 通过 I2C) =====
float heading = 0; // 航向角

// ===== OLED =====
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// ===== 小车状态 =====
float linear_x = 0;   // 期望线速度
float angular_z = 0;  // 期望角速度
bool emergency_stop = false;

// ===== PID 参数 =====
float Kp = 0.5, Ki = 0.1, Kd = 0.05;
long target_ticks[4] = {0};
float integral[4] = {0};
long last_error[4] = {0};

// 编码器中断服务函数
void encISR0() { encoder_count[0]++; }
void encISR1() { encoder_count[1]++; }
void encISR2() { encoder_count[2]++; }
void encISR3() { encoder_count[3]++; }

void setup() {
  Serial.begin(115200);
  Serial1.begin(115200);  // 与树莓派通信的串口
  
  // 电机初始化
  for (int i = 0; i < 4; i++) {
    pinMode(motors[i].in1, OUTPUT);
    pinMode(motors[i].in2, OUTPUT);
    pinMode(motors[i].pwm, OUTPUT);
    digitalWrite(motors[i].in1, LOW);
    digitalWrite(motors[i].in2, LOW);
  }
  
  // 超声波初始化
  for (int i = 0; i < 4; i++) {
    pinMode(us_sensors[i].trig, OUTPUT);
    pinMode(us_sensors[i].echo, INPUT);
  }
  
  // 编码器中断
  attachInterrupt(digitalPinToInterrupt(18), encISR0, RISING);
  attachInterrupt(digitalPinToInterrupt(20), encISR1, RISING);
  attachInterrupt(digitalPinToInterrupt(2),  encISR2, RISING);
  attachInterrupt(digitalPinToInterrupt(4),  encISR3, RISING);
  
  // OLED
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("SmartCar Ready");
  display.display();
  
  // IMU 初始化
  Wire.begin();
  // GY-85 初始化 (MPU6050 + HMC5883L)
  // 后面步骤详细实现
}

void loop() {
  // 1. 读取超声波
  readUltrasonics();
  
  // 2. 读取 IMU
  // readIMU();
  
  // 3. 处理串口指令（来自树莓派）
  processSerial();
  
  // 4. PID 速度控制
  pidControl();
  
  // 5. 更新 OLED
  updateDisplay();
  
  // 6. 发送传感器数据回树莓派
  sendTelemetry();
  
  delay(20); // 50Hz 控制循环
}

// ===== 串口指令解析 =====
void processSerial() {
  if (Serial1.available()) {
    String cmd = Serial1.readStringUntil('\n');
    cmd.trim();
    
    if (cmd.startsWith("VEL ")) {
      // 格式: VEL 线速度 角速度
      // 例如: VEL 0.5 0.0   (前进半速)
      //       VEL 0.0 0.3   (原地左转)
      int sp1 = cmd.indexOf(' ', 4);
      linear_x = cmd.substring(4, sp1).toFloat();
      angular_z = cmd.substring(sp1 + 1).toFloat();
      calculateMotorSpeeds();
    }
    else if (cmd == "STOP") {
      linear_x = 0; angular_z = 0;
      emergency_stop = false;
      stopAll();
    }
    else if (cmd == "ESTOP") {
      emergency_stop = true;
      stopAll();
    }
    else if (cmd == "RESET_ENC") {
      for (int i = 0; i < 4; i++) encoder_count[i] = 0;
    }
    else if (cmd == "PING") {
      Serial1.println("PONG");
    }
    else if (cmd.startsWith("PID ")) {
      // PID 0.5 0.1 0.05
      int sp1 = cmd.indexOf(' ', 4);
      int sp2 = cmd.indexOf(' ', sp1 + 1);
      Kp = cmd.substring(4, sp1).toFloat();
      Ki = cmd.substring(sp1 + 1, sp2).toFloat();
      Kd = cmd.substring(sp2 + 1).toFloat();
    }
  }
}

void calculateMotorSpeeds() {
  if (emergency_stop) { stopAll(); return; }
  
  // 差速驱动力学
  // 左右轮速度 = 线速度 ± (角速度 * 轮距/2)
  float wheel_base = 0.25; // 轮距 25cm
  float left_speed = linear_x - angular_z * wheel_base / 2.0;
  float right_speed = linear_x + angular_z * wheel_base / 2.0;
  
  // 转换为 PWM (0-255)
  int pwm_left = constrain(abs(left_speed) * 255, 0, 255);
  int pwm_right = constrain(abs(right_speed) * 255, 0, 255);
  
  // 设置 4 个电机（左：前左+后左，右：前右+后右）
  setMotor(0, left_speed > 0 ? 1 : -1, pwm_left);
  setMotor(2, left_speed > 0 ? 1 : -1, pwm_left);
  setMotor(1, right_speed > 0 ? 1 : -1, pwm_right);
  setMotor(3, right_speed > 0 ? 1 : -1, pwm_right);
}

void setMotor(int idx, int dir, int pwm) {
  digitalWrite(motors[idx].in1, dir > 0 ? HIGH : LOW);
  digitalWrite(motors[idx].in2, dir < 0 ? HIGH : LOW);
  analogWrite(motors[idx].pwm, pwm);
}

void stopAll() {
  for (int i = 0; i < 4; i++) {
    digitalWrite(motors[i].in1, LOW);
    digitalWrite(motors[i].in2, LOW);
    analogWrite(motors[i].pwm, 0);
  }
}

// ===== 超声波 =====
void readUltrasonics() {
  for (int i = 0; i < 4; i++) {
    digitalWrite(us_sensors[i].trig, LOW);
    delayMicroseconds(2);
    digitalWrite(us_sensors[i].trig, HIGH);
    delayMicroseconds(10);
    digitalWrite(us_sensors[i].trig, LOW);
    
    long duration = pulseIn(us_sensors[i].echo, HIGH, 30000);
    us_sensors[i].distance = duration * 0.034 / 2.0; // cm
    
    // 前方超声波 < 20cm 自动急停
    if (i == 0 && us_sensors[i].distance < 20 && us_sensors[i].distance > 0) {
      emergency_stop = true;
      stopAll();
    }
  }
}

// ===== PID 速度控制 =====
void pidControl() {
  unsigned long now = millis();
  static unsigned long last_time = 0;
  float dt = (now - last_time) / 1000.0;
  if (dt < 0.01) return;
  last_time = now;
  
  for (int i = 0; i < 4; i++) {
    long error = target_ticks[i] - encoder_count[i];
    integral[i] += error * dt;
    float derivative = (error - last_error[i]) / dt;
    last_error[i] = error;
    
    // PID 输出作为 PWM 微调
    // float adjustment = Kp * error + Ki * integral[i] + Kd * derivative;
    // (在 calculateMotorSpeeds 的基础上微调)
  }
}

// ===== 发送遥测数据 =====
void sendTelemetry() {
  static unsigned long last_send = 0;
  if (millis() - last_send < 200) return; // 5Hz
  last_send = millis();
  
  // JSON 格式发送到树莓派
  Serial1.print("{");
  Serial1.print("\"enc\":[");
  for (int i = 0; i < 4; i++) {
    Serial1.print(encoder_count[i]);
    if (i < 3) Serial1.print(",");
  }
  Serial1.print("],\"us\":[");
  for (int i = 0; i < 4; i++) {
    Serial1.print(us_sensors[i].distance);
    if (i < 3) Serial1.print(",");
  }
  Serial1.print("],\"heading\":");
  Serial1.print(heading);
  Serial1.print(",\"estop\":");
  Serial1.print(emergency_stop ? "true" : "false");
  Serial1.println("}");
}

// ===== OLED 显示 =====
void updateDisplay() {
  static unsigned long last_display = 0;
  if (millis() - last_display < 500) return;
  last_display = millis();
  
  display.clearDisplay();
  display.setCursor(0, 0);
  display.print("V:"); display.print(linear_x, 1);
  display.print(" W:"); display.print(angular_z, 1);
  display.setCursor(0, 16);
  display.print("F:"); display.print(us_sensors[0].distance, 0);
  display.print(" B:"); display.print(us_sensors[1].distance, 0);
  display.setCursor(0, 32);
  display.print("L:"); display.print(us_sensors[2].distance, 0);
  display.print(" R:"); display.print(us_sensors[3].distance, 0);
  display.setCursor(0, 48);
  display.print(emergency_stop ? "!! ESTOP !!" : "STATUS: OK");
  display.display();
}
```

### 需要的 Arduino 库
在 Arduino IDE 中：
1. 工具 → 管理库
2. 搜索并安装：
   - `Adafruit SSD1306`
   - `Adafruit GFX Library`
   - `Adafruit MPU6050`
   - `Adafruit HMC5883L`

---

## 3.3 树莓派端：ROS2 串口驱动节点

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python smartcar_driver \
  --dependencies rclpy std_msgs geometry_msgs sensor_msgs
```

```python
# ~/ros2_ws/src/smartcar_driver/smartcar_driver/arduino_bridge.py
"""
Arduino Bridge - ROS2 ↔ Arduino Mega 串口通信
订阅 /cmd_vel → 发给 Arduino → 读取传感器 → 发布 ROS2 话题
"""

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from sensor_msgs.msg import Range, Imu
from nav_msgs.msg import Odometry
import serial
import json
import threading

class ArduinoBridge(Node):
    def __init__(self):
        super().__init__('arduino_bridge')
        
        # 串口连接
        self.declare_parameter('port', '/dev/ttyAMA0')  # 树莓派硬件串口
        port = self.get_parameter('port').value
        
        try:
            self.ser = serial.Serial(port, 115200, timeout=0.1)
            self.get_logger().info(f'串口已连接: {port}')
        except Exception as e:
            self.get_logger().error(f'串口连接失败: {e}')
            self.ser = None
        
        # 订阅速度指令
        self.cmd_sub = self.create_subscription(
            Twist, '/cmd_vel', self.cmd_callback, 10)
        
        # 发布传感器
        self.odom_pub = self.create_publisher(Odometry, '/odom', 10)
        self.range_pubs = [
            self.create_publisher(Range, f'/ultrasonic/{d}', 10)
            for d in ['front', 'back', 'left', 'right']
        ]
        
        # 接收线程
        if self.ser:
            self.rx_thread = threading.Thread(target=self.read_serial, daemon=True)
            self.rx_thread.start()
        
        self.get_logger().info('Arduino Bridge 就绪')
    
    def cmd_callback(self, msg: Twist):
        """将 ROS2 Twist 转为 Arduino 指令"""
        if not self.ser:
            return
        cmd = f"VEL {msg.linear.x:.3f} {msg.angular.z:.3f}\n"
        self.ser.write(cmd.encode())
    
    def read_serial(self):
        """读取 Arduino 发来的传感器数据"""
        while rclpy.ok():
            try:
                line = self.ser.readline().decode().strip()
                if line.startswith('{'):
                    data = json.loads(line)
                    self.publish_sensors(data)
            except:
                pass
    
    def publish_sensors(self, data):
        """发布传感器数据到 ROS2"""
        now = self.get_clock().now().to_msg()
        
        # 编码器 → 里程计
        odom = Odometry()
        odom.header.stamp = now
        odom.header.frame_id = 'odom'
        # 简化：直接用编码器均值推算
        avg_ticks = sum(data.get('enc', [0])) / 4.0
        odom.pose.pose.position.x = avg_ticks * 0.001  # 1 tick = 1mm
        self.odom_pub.publish(odom)
        
        # 超声波 → Range
        directions = ['front', 'back', 'left', 'right']
        for i, d in enumerate(directions):
            if i < len(data.get('us', [])):
                rng = Range()
                rng.header.stamp = now
                rng.header.frame_id = f'ultrasonic_{d}'
                rng.radiation_type = Range.ULTRASOUND
                rng.range = data['us'][i] / 100.0  # cm → m
                rng.min_range = 0.02
                rng.max_range = 4.0
                self.range_pubs[i].publish(rng)
    
    def destroy_node(self):
        if self.ser:
            self.ser.close()
        super().destroy_node()

def main():
    rclpy.init()
    node = ArduinoBridge()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

---

## 3.4 键盘遥控

```bash
# 在树莓派上
sudo apt install -y ros-humble-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

按 `i` 前进、`,` 后退、`j` 左转、`l` 右转、`k` 停止。

> 配合上面的 arduino_bridge 节点在另一个终端运行，小车就能键盘遥控了！

---

## ✅ Day 5-6 验收标准

- [ ] 树莓派能通过串口与 Arduino 通信（PING→PONG）
- [ ] `ros2 run smartcar_driver arduino_bridge` 正常运行
- [ ] 键盘能遥控车前进/后退/转向
- [ ] 前方超声波 < 20cm 自动急停
- [ ] OLED 显示速度和传感器数据

### 调试技巧

```bash
# 测试串口
echo "PING" > /dev/ttyAMA0
cat /dev/ttyAMA0  # 应看到 PONG

# 查看 ROS2 话题
ros2 topic list
ros2 topic echo /cmd_vel
ros2 topic echo /ultrasonic/front
```
