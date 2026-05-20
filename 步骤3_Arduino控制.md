# 姝ラ3锛欰rduino搴曞眰閫氫俊 + 閿洏閬ユ帶锛圖ay 5-6锛?
## 鐩爣
鏍戣帗娲鹃€氳繃涓插彛鎺у埗 Arduino锛岃兘鐢ㄩ敭鐩橀仴鎺у皬杞﹁窇璧锋潵銆?
---

## 3.1 鏍戣帗娲?鈫?Arduino 涓插彛杩炴帴

### 鎺ョ嚎
```
鏍戣帗娲?                    Arduino Mega
  GND (pin 6)    鈹€鈹€鈹€鈹€鈹€鈹€鈫? GND
  TX  (pin 8)    鈹€鈹€鈹€鈹€鈹€鈹€鈫? RX1 (pin 19) 鈫?瀹為檯杩?RX
  RX  (pin 10)   鈹€鈹€鈹€鈹€鈹€鈹€鈫? TX1 (pin 18) 鈫?瀹為檯杩?TX
```

> 鈿狅笍 鏍戣帗娲?GPIO 鏄?3.3V锛孉rduino 鏄?5V銆傚ぇ閮ㄥ垎 Arduino Mega 鍏煎 3.3V 閫昏緫锛屽鏋滀笉鏀惧績鍙互鍔犱竴涓數骞宠浆鎹㈡ā鍧楋紙楼3锛夈€?
---

## 3.2 Arduino 鍥轰欢锛氫覆鍙ｆ寚浠ゆ帶鍒?
鎶婅繖浠戒唬鐮佷笂浼犲埌 Arduino Mega锛?
```cpp
// car_firmware.ino - 瀹屾暣灏忚溅鍥轰欢
// 閫氳繃涓插彛鎺ユ敹鎸囦护锛屾帶鍒剁數鏈?+ 璇诲彇浼犳劅鍣?
#include <Wire.h>
#include <Adafruit_SSD1306.h>  // OLED 搴?
// ===== 鐢垫満寮曡剼 =====
struct Motor {
  int in1, in2, pwm;
};
Motor motors[4] = {
  {22, 23, 6},   // 鍓嶅乏
  {24, 25, 7},   // 鍓嶅彸
  {26, 27, 8},   // 鍚庡乏
  {28, 29, 9}    // 鍚庡彸
};

// ===== 缂栫爜鍣?=====
volatile long encoder_count[4] = {0, 0, 0, 0};
const int encoder_pins[4] = {18, 20, 2, 4};  // A鐩?
// ===== 瓒呭０娉?=====
struct Ultrasonic {
  int trig, echo;
  float distance;
};
Ultrasonic us_sensors[4] = {
  {30, 31, 0},  // 鍓?  {32, 33, 0},  // 鍚?  {34, 35, 0},  // 宸?  {36, 37, 0}   // 鍙?};

// ===== IMU (GY-85 閫氳繃 I2C) =====
float heading = 0; // 鑸悜瑙?
// ===== OLED =====
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// ===== 灏忚溅鐘舵€?=====
float linear_x = 0;   // 鏈熸湜绾块€熷害
float angular_z = 0;  // 鏈熸湜瑙掗€熷害
bool emergency_stop = false;

// ===== PID 鍙傛暟 =====
float Kp = 0.5, Ki = 0.1, Kd = 0.05;
long target_ticks[4] = {0};
float integral[4] = {0};
long last_error[4] = {0};

// 缂栫爜鍣ㄤ腑鏂湇鍔″嚱鏁?void encISR0() { encoder_count[0]++; }
void encISR1() { encoder_count[1]++; }
void encISR2() { encoder_count[2]++; }
void encISR3() { encoder_count[3]++; }

void setup() {
  Serial.begin(115200);
  Serial1.begin(115200);  // 涓庢爲鑾撴淳閫氫俊鐨勪覆鍙?  
  // 鐢垫満鍒濆鍖?  for (int i = 0; i < 4; i++) {
    pinMode(motors[i].in1, OUTPUT);
    pinMode(motors[i].in2, OUTPUT);
    pinMode(motors[i].pwm, OUTPUT);
    digitalWrite(motors[i].in1, LOW);
    digitalWrite(motors[i].in2, LOW);
  }
  
  // 瓒呭０娉㈠垵濮嬪寲
  for (int i = 0; i < 4; i++) {
    pinMode(us_sensors[i].trig, OUTPUT);
    pinMode(us_sensors[i].echo, INPUT);
  }
  
  // 缂栫爜鍣ㄤ腑鏂?  attachInterrupt(digitalPinToInterrupt(18), encISR0, RISING);
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
  
  // IMU 鍒濆鍖?  Wire.begin();
  // GY-85 鍒濆鍖?(MPU6050 + HMC5883L)
  // 鍚庨潰姝ラ璇︾粏瀹炵幇
}

void loop() {
  // 1. 璇诲彇瓒呭０娉?  readUltrasonics();
  
  // 2. 璇诲彇 IMU
  // readIMU();
  
  // 3. 澶勭悊涓插彛鎸囦护锛堟潵鑷爲鑾撴淳锛?  processSerial();
  
  // 4. PID 閫熷害鎺у埗
  pidControl();
  
  // 5. 鏇存柊 OLED
  updateDisplay();
  
  // 6. 鍙戦€佷紶鎰熷櫒鏁版嵁鍥炴爲鑾撴淳
  sendTelemetry();
  
  delay(20); // 50Hz 鎺у埗寰幆
}

// ===== 涓插彛鎸囦护瑙ｆ瀽 =====
void processSerial() {
  if (Serial1.available()) {
    String cmd = Serial1.readStringUntil('\n');
    cmd.trim();
    
    if (cmd.startsWith("VEL ")) {
      // 鏍煎紡: VEL 绾块€熷害 瑙掗€熷害
      // 渚嬪: VEL 0.5 0.0   (鍓嶈繘鍗婇€?
      //       VEL 0.0 0.3   (鍘熷湴宸﹁浆)
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
  
  // 宸€熼┍鍔ㄥ姏瀛?  // 宸﹀彸杞€熷害 = 绾块€熷害 卤 (瑙掗€熷害 * 杞窛/2)
  float wheel_base = 0.25; // 杞窛 25cm
  float left_speed = linear_x - angular_z * wheel_base / 2.0;
  float right_speed = linear_x + angular_z * wheel_base / 2.0;
  
  // 杞崲涓?PWM (0-255)
  int pwm_left = constrain(abs(left_speed) * 255, 0, 255);
  int pwm_right = constrain(abs(right_speed) * 255, 0, 255);
  
  // 璁剧疆 4 涓數鏈猴紙宸︼細鍓嶅乏+鍚庡乏锛屽彸锛氬墠鍙?鍚庡彸锛?  setMotor(0, left_speed > 0 ? 1 : -1, pwm_left);
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

// ===== 瓒呭０娉?=====
void readUltrasonics() {
  for (int i = 0; i < 4; i++) {
    digitalWrite(us_sensors[i].trig, LOW);
    delayMicroseconds(2);
    digitalWrite(us_sensors[i].trig, HIGH);
    delayMicroseconds(10);
    digitalWrite(us_sensors[i].trig, LOW);
    
    long duration = pulseIn(us_sensors[i].echo, HIGH, 30000);
    us_sensors[i].distance = duration * 0.034 / 2.0; // cm
    
    // 鍓嶆柟瓒呭０娉?< 20cm 鑷姩鎬ュ仠
    if (i == 0 && us_sensors[i].distance < 20 && us_sensors[i].distance > 0) {
      emergency_stop = true;
      stopAll();
    }
  }
}

// ===== PID 閫熷害鎺у埗 =====
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
    
    // PID 杈撳嚭浣滀负 PWM 寰皟
    // float adjustment = Kp * error + Ki * integral[i] + Kd * derivative;
    // (鍦?calculateMotorSpeeds 鐨勫熀纭€涓婂井璋?
  }
}

// ===== 鍙戦€侀仴娴嬫暟鎹?=====
void sendTelemetry() {
  static unsigned long last_send = 0;
  if (millis() - last_send < 200) return; // 5Hz
  last_send = millis();
  
  // JSON 鏍煎紡鍙戦€佸埌鏍戣帗娲?  Serial1.print("{");
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

// ===== OLED 鏄剧ず =====
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

### 闇€瑕佺殑 Arduino 搴?鍦?Arduino IDE 涓細
1. 宸ュ叿 鈫?绠＄悊搴?2. 鎼滅储骞跺畨瑁咃細
   - `Adafruit SSD1306`
   - `Adafruit GFX Library`
   - `Adafruit MPU6050`
   - `Adafruit HMC5883L`

---

## 3.3 鏍戣帗娲剧锛歊OS2 涓插彛椹卞姩鑺傜偣

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python smartcar_driver \
  --dependencies rclpy std_msgs geometry_msgs sensor_msgs
```

```python
# ~/ros2_ws/src/smartcar_driver/smartcar_driver/arduino_bridge.py
"""
Arduino Bridge - ROS2 鈫?Arduino Mega 涓插彛閫氫俊
璁㈤槄 /cmd_vel 鈫?鍙戠粰 Arduino 鈫?璇诲彇浼犳劅鍣?鈫?鍙戝竷 ROS2 璇濋
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
        
        # 涓插彛杩炴帴
        self.declare_parameter('port', '/dev/ttyAMA0')  # 鏍戣帗娲剧‖浠朵覆鍙?        port = self.get_parameter('port').value
        
        try:
            self.ser = serial.Serial(port, 115200, timeout=0.1)
            self.get_logger().info(f'涓插彛宸茶繛鎺? {port}')
        except Exception as e:
            self.get_logger().error(f'涓插彛杩炴帴澶辫触: {e}')
            self.ser = None
        
        # 璁㈤槄閫熷害鎸囦护
        self.cmd_sub = self.create_subscription(
            Twist, '/cmd_vel', self.cmd_callback, 10)
        
        # 鍙戝竷浼犳劅鍣?        self.odom_pub = self.create_publisher(Odometry, '/odom', 10)
        self.range_pubs = [
            self.create_publisher(Range, f'/ultrasonic/{d}', 10)
            for d in ['front', 'back', 'left', 'right']
        ]
        
        # 鎺ユ敹绾跨▼
        if self.ser:
            self.rx_thread = threading.Thread(target=self.read_serial, daemon=True)
            self.rx_thread.start()
        
        self.get_logger().info('Arduino Bridge 灏辩华')
    
    def cmd_callback(self, msg: Twist):
        """灏?ROS2 Twist 杞负 Arduino 鎸囦护"""
        if not self.ser:
            return
        cmd = f"VEL {msg.linear.x:.3f} {msg.angular.z:.3f}\n"
        self.ser.write(cmd.encode())
    
    def read_serial(self):
        """璇诲彇 Arduino 鍙戞潵鐨勪紶鎰熷櫒鏁版嵁"""
        while rclpy.ok():
            try:
                line = self.ser.readline().decode().strip()
                if line.startswith('{'):
                    data = json.loads(line)
                    self.publish_sensors(data)
            except:
                pass
    
    def publish_sensors(self, data):
        """鍙戝竷浼犳劅鍣ㄦ暟鎹埌 ROS2"""
        now = self.get_clock().now().to_msg()
        
        # 缂栫爜鍣?鈫?閲岀▼璁?        odom = Odometry()
        odom.header.stamp = now
        odom.header.frame_id = 'odom'
        # 绠€鍖栵細鐩存帴鐢ㄧ紪鐮佸櫒鍧囧€兼帹绠?        avg_ticks = sum(data.get('enc', [0])) / 4.0
        odom.pose.pose.position.x = avg_ticks * 0.001  # 1 tick = 1mm
        self.odom_pub.publish(odom)
        
        # 瓒呭０娉?鈫?Range
        directions = ['front', 'back', 'left', 'right']
        for i, d in enumerate(directions):
            if i < len(data.get('us', [])):
                rng = Range()
                rng.header.stamp = now
                rng.header.frame_id = f'ultrasonic_{d}'
                rng.radiation_type = Range.ULTRASOUND
                rng.range = data['us'][i] / 100.0  # cm 鈫?m
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

## 3.4 閿洏閬ユ帶

```bash
# 鍦ㄦ爲鑾撴淳涓?sudo apt install -y ros-humble-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

鎸?`i` 鍓嶈繘銆乣,` 鍚庨€€銆乣j` 宸﹁浆銆乣l` 鍙宠浆銆乣k` 鍋滄銆?
> 閰嶅悎涓婇潰鐨?arduino_bridge 鑺傜偣鍦ㄥ彟涓€涓粓绔繍琛岋紝灏忚溅灏辫兘閿洏閬ユ帶浜嗭紒

---

## 鉁?Day 5-6 楠屾敹鏍囧噯

- [ ] 鏍戣帗娲捐兘閫氳繃涓插彛涓?Arduino 閫氫俊锛圥ING鈫扨ONG锛?- [ ] `ros2 run smartcar_driver arduino_bridge` 姝ｅ父杩愯
- [ ] 閿洏鑳介仴鎺ц溅鍓嶈繘/鍚庨€€/杞悜
- [ ] 鍓嶆柟瓒呭０娉?< 20cm 鑷姩鎬ュ仠
- [ ] OLED 鏄剧ず閫熷害鍜屼紶鎰熷櫒鏁版嵁

### 璋冭瘯鎶€宸?
```bash
# 娴嬭瘯涓插彛
echo "PING" > /dev/ttyAMA0
cat /dev/ttyAMA0  # 搴旂湅鍒?PONG

# 鏌ョ湅 ROS2 璇濋
ros2 topic list
ros2 topic echo /cmd_vel
ros2 topic echo /ultrasonic/front
```
