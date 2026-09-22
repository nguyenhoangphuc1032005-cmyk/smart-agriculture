# Smart Agriculture System
## Raspberry Pi 4 + ESP32 + STM32

### Kiến trúc hệ thống

```
        CAMERA
           │
           ▼
    ┌──────────────────┐
    │  Raspberry Pi 4  │
    │                  │
    │  • AI Disease    │
    │  • Prediction    │
    │  • Web Server    │
    │  • Database      │
    └────────┬─────────┘
             │ UART/WiFi
             ▼
       ┌───────────┐
       │   ESP32   │
       │ WiFi/IoT  │
       └─────┬─────┘
             │ UART
             ▼
       ┌───────────┐
       │   STM32   │
       │ Realtime  │
       └─────┬─────┘
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
  Pump      Fan      LED
```

### Phân chia nhiệm vụ

**Raspberry Pi 4 (1GB)**
- 📷 Nhận diện bệnh lá cây (Disease Detection)
- 🧠 Dự đoán nhu cầu tưới (Watering Prediction)
- 🚨 Phát hiện bất thường (Anomaly Detection)
- 🌐 Web Dashboard
- 💾 Database (SQLite)
- 📊 Data Analytics

**ESP32**
- 📡 WiFi/MQTT
- ☁️ Cloud connectivity
- 🔄 Data relay Pi ↔ STM32
- 📱 Mobile app interface

**STM32**
- 🌡️ Đọc cảm biến (DHT22, Soil, LDR)
- ⚙️ Điều khiển realtime (Pump, Fan, LED)
- ⏱️ Timer/PWM
- 🛡️ Failsafe control

### Cấu trúc thư mục

```
smart-agriculture/
├── raspberry-pi/
│   ├── ai/
│   │   ├── disease_detection.py
│   │   ├── watering_prediction.py
│   │   └── models/
│   ├── web/
│   │   ├── app.py
│   │   ├── templates/
│   │   └── static/
│   ├── database/
│   │   └── db_manager.py
│   └── comm/
│       └── uart_handler.py
├── esp32/
│   ├── main/
│   │   ├── main.cpp
│   │   ├── wifi_manager.cpp
│   │   └── mqtt_handler.cpp
│   └── platformio.ini
├── stm32/
│   ├── Core/
│   │   ├── Src/
│   │   └── Inc/
│   └── Drivers/
└── docs/
    ├── hardware_setup.md
    ├── api_spec.md
    └── uart_protocol.md
```

### Hardware requirements

| Component | Model | Purpose |
|-----------|-------|---------|
| Main Board | Raspberry Pi 4 1GB | AI + Server |
| IoT Gateway | ESP32 | WiFi/MQTT |
| Controller | STM32F103C8T6 | Sensors + Control |
| Camera | Pi Camera / USB | Disease detection |
| Temp/Humidity | DHT22 | Environment |
| Soil Moisture | Capacitive | Watering control |
| Light Sensor | BH1750/LDR | Grow light |
| Water Pump | 5V Mini | Irrigation |
| Fan | 5V/12V | Ventilation |
| Grow LED | Full spectrum | Lighting |
| Relay | 4CH 5V | Actuator control |

### Giai đoạn phát triển

#### Phase 1: Hardware Setup ✅
- [x] Thiết lập STM32
- [ ] Đọc DHT22, Soil, LDR
- [ ] Test relay control
- [ ] UART communication

#### Phase 2: Basic Control 🔄
- [ ] Logic tưới tự động
- [ ] Control từ ngưỡng
- [ ] Failsafe mechanism

#### Phase 3: IoT Layer
- [ ] ESP32 WiFi
- [ ] MQTT broker
- [ ] Data forwarding

#### Phase 4: Web Dashboard
- [ ] Flask/FastAPI server
- [ ] Real-time monitoring
- [ ] Manual control
- [ ] Data visualization

#### Phase 5: AI Integration
- [ ] Disease detection model
- [ ] Watering prediction
- [ ] Anomaly detection
- [ ] Model optimization

### Quick start

```bash
# Raspberry Pi setup
cd raspberry-pi
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Run web server
python web/app.py

# Train AI model
python ai/train_disease_model.py
```

### API Endpoints

```
GET  /api/sensors          # Current sensor data
GET  /api/devices          # Device status
POST /api/devices/{id}     # Control device
GET  /api/history          # Historical data
GET  /api/ai/predict       # AI prediction
POST /api/ai/analyze       # Analyze image
```

### UART Protocol

```
STM32 → ESP32/Pi:
{temp:29.4,hum:76,soil:38,light:620}

ESP32/Pi → STM32:
{pump:1,fan:0,led:0}
```
