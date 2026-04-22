# 🤖 ESP32 Robot Control Board — Arduino Firmware

Firmware cho board điều khiển robot tự chế dựa trên **ESP32-DevKit**, hỗ trợ động cơ DC, servo, cảm biến line, encoder và tay cầm Bluetooth (PS4/Bluepad32).

---

## 📸 Robot in Action

<p align="center">
  <img src="https://github.com/nhandream2008/ROBOTO2026/blob/main/PICTURE/robocon20261.jpg?raw=true" width="32%" />
  <img src="https://github.com/nhandream2008/ROBOTO2026/blob/main/PICTURE/robocon20262.jpg?raw=true" width="32%" />
  <img src="https://github.com/nhandream2008/ROBOTO2026/blob/main/PICTURE/robocon20263.jpg?raw=true" width="32%" />
</p>

---

## 📦 Tổng quan dự án

Project gồm **3 chương trình firmware** độc lập, dùng cho các kịch bản khác nhau:

| File | Mô tả |
|------|-------|
| `doline.ino` | Robot dò line tự động với điều khiển PD, 2 servo, cảm biến IR & cảm biến chạm |
| `uart-masterservo.ino` | Board **Master** — điều khiển xe Mecanum 4 bánh qua tay cầm PS4 (Bluepad32), gửi lệnh sang Slave qua UART |
| `uart-slaveservo.ino` | Board **Slave** — nhận lệnh từ Master qua UART, điều khiển 1 DC motor + 4 servo |

---

## 🖼️ Hình ảnh board mạch

| Bảng chân GPIO | PCB thực tế |
|:--------------:|:-----------:|
| ![GPIO Pinout Table](https://github.com/nhandream2008/ROBOCON2026/blob/main/z6495436187579_8bf46dbf31f42a0ed399d9836a0657af.jpg?raw=true) | ![PCB Board](https://github.com/nhandream2008/ROBOCON2026/blob/main/z6495436189105_b903e7c0acb39fa9991fd42e7c7c8caf.jpg?raw=true) |

---

## 🔌 Sơ đồ chân (GPIO Pinout)

Theo **Fig2. ESP32-devkit module pinout**:

| Thiết bị | Tên tín hiệu | GPIO | Symbol |
|----------|-------------|------|--------|
| DC Motor 1 | DIR_1_pin | GPIO23 | 23 |
| DC Motor 1 | PWM_1_pin | GPIO16 | 16 |
| DC Motor 2 | DIR_2_pin | GPIO22 | 22 |
| DC Motor 2 | PWM_2_pin | GPIO4 | 4 |
| DC Motor 3 | DIR_3_pin | GPIO18 | 18 |
| DC Motor 3 | PWM_3_pin | GPIO0 | 0 |
| DC Motor 4 | DIR_4_pin | GPIO17 | 17 |
| DC Motor 4 | PWM_4_pin | GPIO15 | 15 |
| Servo 1 | SERVO_1_pin | GPIO5 | 5 |
| Servo 2 | SERVO_2_pin | GPIO13 | 13 |
| Servo 3 | SERVO_3_pin | GPIO12 | 12 |
| Servo 4 | SERVO_4_pin | GPIO14 | 14 |
| Line Sensor EN | LSS_EN_pin | GPIO21 | 21 |
| Line Sensor 5 | LSS_5_pin | GPIO32 | 32 |
| Line Sensor 4 | LSS_4_pin | GPIO33 | 33 |
| Line Sensor 3 | LSS_3_pin | GPIO25 | 25 |
| Line Sensor 2 | LSS_2_pin | GPIO26 | 26 |
| Line Sensor 1 | LSS_1_pin | GPIO27 | 27 |
| Input 1 | input_1_pin | GPIO35 | 35 |
| Input 2 | input_2_pin | GPIO34 | 34 |
| Input 3 | input_3_pin | GPIO39 | VP |
| Input 4 | input_4_pin | GPIO36 | VN |

---

## 📁 Chi tiết từng firmware

---

### 1. `doline.ino` — Robot dò line tự động

Robot dò line dùng **5 cảm biến hồng ngoại**, điều khiển 2 động cơ DC theo thuật toán **PD (Proportional-Derivative)**. Ngoài ra tích hợp 2 servo để thực hiện tác vụ (nhặt/thả vật), hỗ trợ 2 chế độ A/B.

#### ⚙️ Cấu hình nhanh

```cpp
#define KP 50          // Hệ số tỉ lệ PD
#define KD 35          // Hệ số đạo hàm PD
int spd = 75;          // Tốc độ cơ bản (PWM 0–255)
const int threshold = 500;  // Ngưỡng phân biệt vạch đen/trắng
```

#### 🔑 Các hàm quan trọng

##### `read_sensor_values()`
Đọc giá trị 5 cảm biến line và tính toán `error` từ **-4 đến +4** (âm = lệch trái, dương = lệch phải, 0 = thẳng).

```cpp
void read_sensor_values() {
  // Đọc 5 cảm biến (s0..s4) và ánh xạ sang error
  // Ví dụ: chỉ s4 sáng -> error = +4 (lệch phải nhiều nhất)
  //        chỉ s0 sáng -> error = -4 (lệch trái nhiều nhất)
}
```

##### `do_line()`
Gọi `read_sensor_values()` rồi tính tốc độ bù cho 2 bánh theo công thức PD:

```cpp
void do_line() {
  read_sensor_values();
  int motorSpeed = KP * error + KD * (error - lastError);
  lastError = error;
  set_motors(spd + motorSpeed, spd - motorSpeed);
}
```

##### `writeServos(int angle1)` / `writeServosSlow(int targetAngle1)`
Điều khiển 2 servo theo quy luật đối xứng (Servo2 = 180 − angle1 + 5). Hàm `writeServosSlow` xoay từng bước 1°, delay 15ms/bước để servo chuyển động mượt mà, tránh giật.

```cpp
void writeServos(int angle1) {
  int angle2 = (180 - angle1) + 5;
  servo1.write(angle1);
  servo2.write(angle2);
}
```

##### `allSensorsBlack()`
Kiểm tra toàn bộ 5 cảm biến đều đọc vạch đen → dùng để phát hiện **vạch dừng** (giao lộ, điểm đặc biệt).

#### 🗺️ State machine

| `stopCount` | Sự kiện | Hành động |
|-------------|---------|-----------|
| 0 | Đang chạy | Dò line bình thường |
| 1 | Gặp vạch đen lần 1 | Dừng, xoay servo tác vụ, chạy tiếp |
| 2 | Gặp vạch đen lần 2 → state=1 | Chờ tín hiệu IR (chân 14) để qua dốc |
| — | Cảm biến chạm (chân 21) | Dừng vĩnh viễn, servo về vị trí cuối |

---

### 2. `uart-masterservo.ino` — Board Master (Mecanum + Bluepad32)

Điều khiển xe **Mecanum 4 bánh** qua tay cầm **PS4** (Bluepad32 over Bluetooth). Giao tiếp với board Slave qua **UART2 (TX=GPIO14, RX=GPIO12, 115200 baud)**.

#### 🎮 Sơ đồ phím tay cầm PS4

| Nút | Chức năng |
|-----|-----------|
| **Joystick Trái (LX/LY)** | Di chuyển Mecanum (tiến/lùi/trái/phải/chéo) |
| **Joystick Phải (RX)** | Quay trái / Quay phải tại chỗ |
| **R2** (giữ) | Tăng tốc độ Mecanum |
| **L2** (giữ) | Giảm tốc độ Mecanum |
| **R1** (giữ/thả) | Motor Slave tiến / dừng |
| **Home** (giữ/thả) | Motor Slave lùi / dừng |
| **L1** (toggle) | Slave Servo4: 180° ↔ 90° |
| **O / Circle** (toggle) | Slave S1+S2: vị trí 1 ↔ vị trí 2 |
| **X / Cross** (toggle) | Slave S3: 180° ↔ 90° |
| **□ / Square** | Local Servo: Vị trí A (nạp) |
| **△ / Triangle** | Local Servo: Vị trí B (bắn/reset) |
| **DPad Lên/Xuống** (giữ/thả) | Slave S1 tiến/lùi |
| **DPad Phải/Trái** (giữ/thả) | Slave S2 tiến/lùi |
| **Option** | Tăng offset tốc độ bánh trước (+10 PWM) |
| **Share** | Giảm offset tốc độ bánh trước (-10 PWM) |

#### 🔑 Các hàm quan trọng

##### `rampMotors()`
Tăng/giảm PWM từ từ theo từng bước `RAMP_STEP_PWM` mỗi `RAMP_STEP_MS` ms, tránh sụt áp nguồn khi khởi động đột ngột.

```cpp
void rampMotors() {
  // Gọi mỗi vòng loop — không blocking
  // Tự động áp dụng frontOffset cho Motor 1 & 2 (bánh trước)
}
```

##### `xuLyChuDong()`
Ánh xạ Joystick Trái → 8 hướng chuyển động Mecanum: tiến, lùi, trái, phải, và 4 hướng chéo.

##### `guiHeartbeat()`
Mỗi 2 giây gửi lại lệnh motor hiện tại sang Slave để resync trạng thái, phòng mất lệnh do nhiễu UART.

##### `capNhatSquareAnimation()`
State machine **không blocking** cho hoạt ảnh nạp đạn (Square button): điều khiển servo theo 3 bước với timer `millis()`, không dùng `delay()`.

##### `khiKetNoi()` / `khiNgatKetNoi()`
Callback Bluepad32 — khi kết nối: chờ 500ms để joystick ổn định trước khi kích hoạt motor; khi mất kết nối: dừng toàn bộ motor ngay lập tức.

#### 📡 Bảng lệnh UART gửi sang Slave

| Ký tự | Lệnh |
|-------|------|
| `A` | Motor Slave TIẾN |
| `R` | Motor Slave LÙI |
| `S` | Motor Slave DỪNG |
| `I` | S1=0°, S2=60° |
| `J` | S1=90°, S2=60° |
| `K` | S3=180° |
| `L` | S3=90° |
| `N` | S4=180° |
| `P` | S4=90° |
| `B/C/D` | S1 tiến / lùi / dừng |
| `E/F/G` | S2 tiến / lùi / dừng |
| `H` | S1=38°, S2=87° |
| `Q` | S1=5°, S2=55° |

---

### 3. `uart-slaveservo.ino` — Board Slave

Nhận lệnh ASCII qua **UART2 (RX=GPIO12)** từ Master và điều khiển 1 DC motor + 4 servo.

#### 🔑 Các hàm quan trọng

##### `servoWrite(int ch, int degrees)`
Tính duty cycle 16-bit cho LEDC 50Hz từ góc độ (ánh xạ 0–180° → 500–2400 µs pulse).

```cpp
void servoWrite(int ch, int degrees) {
  int pulseUs = map(degrees, 0, 180, 500, 2400);
  int duty = (int)((long)pulseUs * 65535 / 20000);
  ledcWrite(ch, duty);
}
```

##### `servoWriteSafe(int ch, int angleDeg, int &lastWritten)`
Chỉ ghi servo khi góc thực sự thay đổi — tránh gửi lệnh LEDC thừa khi DPad đang giữ.

##### `processCommand(char cmd)`
Switch-case xử lý toàn bộ ký tự lệnh từ UART: điều khiển motor, đặt góc servo cố định, hoặc kích hoạt chế độ chạy liên tục cho S1/S2.

##### Servo sweep liên tục (DPad)
Trong `loop()`, S1 và S2 có **state machine** (`s1State`, `s2State` = 0/±1). Khi đang ở trạng thái chạy, cứ mỗi 40ms servo tăng/giảm 7° — đạt tốc độ ~25°/giây. Chỉ ghi LEDC khi góc thực sự thay đổi.

##### Servo refresh định kỳ
Mỗi 500ms resend duty cho tất cả servo, phòng mất vị trí khi LEDC bị reset do sụt áp nguồn.

---

## 🛠️ Thư viện cần cài

| Thư viện | Dùng cho |
|----------|----------|
| [ESP32Servo](https://github.com/madhephaestus/ESP32Servo) | `doline.ino` — điều khiển servo |
| [Bluepad32](https://github.com/ricardoquesada/bluepad32) | `uart-masterservo.ino` — kết nối tay cầm PS4 |
| Arduino ESP32 core | Tất cả (LEDC, UART, GPIO) |

---

## 🚀 Hướng dẫn nạp code

1. Cài **Arduino IDE** (≥ 2.x) + ESP32 board package.
2. Cài thư viện theo bảng trên qua **Library Manager**.
3. Mở file `.ino` tương ứng với board cần nạp.
4. Chọn board: `ESP32 Dev Module`, chọn đúng COM port.
5. Nạp code (**Upload**).

> ⚠️ **Lưu ý:** `uart-masterservo.ino` và `uart-slaveservo.ino` là 2 board ESP32 riêng biệt — nạp đúng file vào đúng board.

---

## 📐 Kiến trúc hệ thống

```
┌─────────────────────────────────┐       UART (115200)        ┌──────────────────────────────┐
│  Board MASTER                   │ ─────────────────────────► │  Board SLAVE                 │
│  ESP32 #1                       │   TX=GPIO14 → RX=GPIO12    │  ESP32 #2                    │
│                                 │                             │                              │
│  • Mecanum 4 motors (M1–M4)     │                             │  • DC Motor 2                │
│  • Local Servo 1 & 2            │                             │  • Servo 1, 2, 3, 4          │
│  • Bluepad32 (PS4 Bluetooth)    │                             │  • Nhận lệnh A–Q             │
└─────────────────────────────────┘                             └──────────────────────────────┘

┌─────────────────────────────────┐
│  Robot dò line (độc lập)        │
│  ESP32 #3 — doline.ino          │
│  • 5 cảm biến line              │
│  • PD control                   │
│  • 2 Servo (đối xứng)           │
│  • Encoder trái/phải            │
└─────────────────────────────────┘
```

---

## 📄 License

khong share code dc roiii  :))))) 

