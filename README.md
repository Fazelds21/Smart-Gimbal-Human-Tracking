# 🎯 Smart AI Gimbal & Real-Time Human Tracking

[![Python](https://img.shields.io/badge/Python-3.10%20%7C%203.11-blue?logo=python&logoColor=white)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/PyTorch-CUDA%2012.x-EE4C2C?logo=pytorch&logoColor=white)](https://pytorch.org/)
[![YOLOv8](https://img.shields.io/badge/YOLOv8-Pose%20Estimation-00FFFF?logo=ultralytics&logoColor=black)](https://github.com/ultralytics/ultralytics)
[![ESP32](https://img.shields.io/badge/ESP--IDF-ESP32%20DevKit-red?logo=espressif&logoColor=white)](https://www.espressif.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Sistem pelacak manusia (*real-time human tracking*) dan penyeimbang kamera aktif (*self-stabilizing 2-axis gimbal*) berbasis **Hybrid Architecture**. Pemrosesan Computer Vision tingkat lanjut (YOLOv8-pose) dijalankan secara lokal dengan akselerasi GPU CUDA (RTX 4050), sedangkan komputasi stabilisasi IMU serta aktuasi motor servo dikendalikan secara presisi oleh mikrokontroler ESP32.

---

## 📌 Fitur Utama

- **Real-Time Human Body Tracking:** Estimasi 17 *keypoints* anatomi tubuh manusia via `YOLOv8n-pose` pada kecepatan >100 FPS di GPU laptop.
- **Robust Torso Centering:** Menghindari *overshoot* dan goyangan dengan menggunakan titik tengah anatomi tubuh (*midpoint torso*) sebagai target acuan kamera.
- **Active 2-Axis Gimbal Stabilization:** Stabilisasi sudut fisik (*roll/pitch*) menggunakan sensor IMU 6-Axis **MPU6050** yang digabungkan melalui algoritma **Filter Komplementer**.
- **Zero-Latency Video Pipeline:** Optimasi *framebuffer* OpenCV (`CAP_PROP_BUFFERSIZE = 1`) dan *DirectShow* backend untuk respon kamera instan tanpa lag antrean.
- **Low-Cost Prototyping:** Seluruh sistem dirancang dengan total biaya piranti keras di bawah **Rp350.000**.
- **Human Height Estimation (Opsional/Tahap Lanjut):** Estimasi tinggi badan subjek menggunakan proyeksi geometri optik *Fixed-Focus Pinhole Camera*.

---

## 🏗️ Arsitektur Sistem

Proyek ini menggunakan pemisahan tugas secara modular (*hybrid computing*):

```text
  [ Rexus SW-10 Webcam ] ──(USB Direct)──► [ Laptop (RTX 4050) ]
        (Fixed Focus 1080p)                       │
                                                  │ YOLOv8-pose + Error Computing
                                                  │ Protokol UDP / WebSocket (Wi-Fi)
                                                  ▼
                                            [ ESP32 DevKit V1 ]
                                                  ▲
  [ MPU6050 Sensor (IMU) ] ──(I2C: 21, 22)────────┤
        (Gimbal Active Roll/Pitch Correction)     │ PWM (GPIO 18 & 19)
                                                  ▼
                                       [ 2x Micro Servo (MG90S) ]
                                          (Pan & Tilt Mechanism)
```

1. **Vision & AI Unit (Laptop):** Menarik *stream* video mentah dari webcam, mendeteksi kerangka manusia, menghitung selisih galat (*error offset*) dari titik tengah bidang pandang $(640, 360)$, dan mentransmisikan instruksi koreksi sudut.
2. **Motion Control Unit (ESP32):** Mengukur orientasi fisik platform secara lokal, menghitung algoritma peredam goyangan, dan memadukan koreksi sudut gimbal dengan perintah pelacakan visual ke motor servo.

---

## 🔌 Skema Rangkaian Elektronika

> **PERINGATAN:** Jangan mengambil arus daya motor servo langsung dari pin 5V / 3.3V ESP32 atau port USB laptop. Selalu gunakan adaptor eksternal 5V $\ge$ 2A dengan konfigurasi **Common Ground**.

### 1. Pinout Wiring

| Komponen | Pin Komponen | Terhubung ke (ESP32 / Breadboard) | Keterangan |
| :--- | :--- | :--- | :--- |
| **MPU6050** | VCC | Pin `3V3` atau `VIN` ESP32 | Logika 3.3V / 5V |
| | GND | Pin `GND` ESP32 | Ground bersama |
| | SDA | Pin `GPIO 21` ESP32 | Jalur Data I2C |
| | SCL | Pin `GPIO 22` ESP32 | Jalur Clock I2C |
| **Servo Pan (Yaw)** | Sinyal (Oranye/Kuning) | Pin `GPIO 18` ESP32 | Kontrol PWM |
| | VCC (Merah) | Rel Positif (+) Breadboard | 5V DC Eksternal |
| | GND (Cokelat/Hitam)| Rel Negatif (-) Breadboard | Ground Eksternal |
| **Servo Tilt (Pitch)**| Sinyal (Oranye/Kuning) | Pin `GPIO 19` ESP32 | Kontrol PWM |
| | VCC (Merah) | Rel Positif (+) Breadboard | 5V DC Eksternal |
| | GND (Cokelat/Hitam)| Rel Negatif (-) Breadboard | Ground Eksternal |
| **Adaptor 5V 2A** | Terminal (+) | Rel Positif (+) Breadboard | Sumber arus servo |
| | Terminal (-) | Rel Negatif (-) Breadboard | Sumber arus servo |
| **Kapasitor Elco** | Kaki Panjang (+) | Rel Positif (+) Breadboard | Penyangga arus (*decoupling*) |
| (1000 µF 16V) | Kaki Pendek / Strip (-) | Rel Negatif (-) Breadboard | Mencegah *brownout reset* |
| **Jumper Ground** | Pin `GND` ESP32 | Rel Negatif (-) Breadboard | **Wajib: Common Ground** |

---

## 📐 Landasan Matematis & Logika Kontrol

### 1. Komputasi Error Piksel
Diberikan resolusi kamera $W \times H$ ($1280 \times 720$), titik tengah frame didefinisikan sebagai:
$$X_{\text{center}} = \frac{W}{2} = 640, \quad Y_{\text{center}} = \frac{H}{2} = 360$$

Titik target dihitung dari rata-rata posisi bahu kiri $(X_5, Y_5)$ dan bahu kanan $(X_6, Y_6)$:
$$X_{\text{target}} = \frac{X_5 + X_6}{2}, \quad Y_{\text{target}} = \frac{Y_5 + Y_6}{2}$$

Galat tracking dinyatakan dalam delta piksel:
$$e_x = X_{\text{target}} - X_{\text{center}}, \quad e_y = Y_{\text{target}} - Y_{\text{center}}$$

### 2. Deadband Filtering & Proportional Controller
Untuk mencegah getaran (*hunting/jitter*) saat subjek sudah berada di area tengah, diterapkan zona batas mati (*deadband*):
$$e_x' = \begin{cases} 0, & \text{jika } |e_x| \le \text{Threshold} \\ e_x, & \text{jika } |e_x| > \text{Threshold} \end{cases}$$

Pergeseran sudut servo horizontal ($\Delta \theta_{\text{pan}}$) dikendalikan secara proporsional:
$$\Delta \theta_{\text{pan}} = K_p \cdot e_x'$$

### 3. Filter Komplementer Gimbal (MPU6050)
Penyatuan data accelerometer (bebas penyimpangan jangka panjang namun rentan getaran) dan gyroscope (sangat responsif namun mengalami *drift*):
$$\theta_{t} = \alpha \cdot (\theta_{t-1} + \omega \cdot \Delta t) + (1 - \alpha) \cdot \theta_{\text{accel}}$$
*(Nilai tipikal $\alpha = 0.98$)*

---

## 🚀 Panduan Memulai Cepat (Quickstart)

### 1. Klon Repositori & Persiapan Environment
Pastikan Python 3.10+ dan CUDA Toolkit 12.x sudah terpasang.

```bash
# Clone repository
git clone https://github.com/username/smart-gimbal-human-tracking.git
cd smart-gimbal-human-tracking

# Buat virtual environment
python -m venv venv
# Windows:
.\venv\Scripts\activate
# Linux/MacOS:
source venv/bin/activate

# Instal dependensi PyTorch dengan akselerasi CUDA
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121

# Instal paket pendukung lainnya
pip install -r requirements.txt
```

### 2. Verifikasi Akselerasi CUDA
Jalankan perintah berikut untuk memastikan PyTorch mendeteksi GPU RTX 4050:
```bash
python -c "import torch; print('CUDA Ready:', torch.cuda.is_available()); print('Device:', torch.cuda.get_device_name(0))"
```

### 3. Menjalankan Pipeline Vision (Tahap 1)
Uji kamera dan visualisasi pelacakan rangka manusia di laptop:
```bash
python src/vision_pipeline.py
```

---

## 📁 Struktur Direktori

```text
smart-gimbal-human-tracking/
├── firmware/
│   ├── esp32_gimbal_controller/
│   │   └── esp32_gimbal_controller.ino   # Firmware Arduino IDE untuk ESP32
├── src/
│   ├── vision_pipeline.py                # Main script pipeline OpenCV + YOLOv8
│   ├── tracker_controller.py             # Logika P-Controller & filter deadband
│   └── comms/
│       └── udp_sender.py                 # Modul socket transmisi koordinat
├── assets/
│   ├── schematic_wiring.png              # Diagram breadboard Fritzing
│   └── system_architecture.png           # Blok diagram sistem
├── requirements.txt                      # Daftar dependensi Python
├── LICENSE                               # Lisensi MIT
└── README.md                             # Dokumentasi utama
```

---

## 🛠️ Daftar Komponen Piranti Keras

- **1x** ESP32 DOIT DevKit V1 (30 / 38 pin)
- **1x** Rexus SW-10 Full HD 1080p Webcam (Fixed-Focus, FOV 80°)
- **1x** Sensor IMU 6-Axis MPU6050 (GY-521)
- **1x** Bracket Pan-Tilt 2-Axis
- **2x** Motor Servo Micro MG90S (Metal Gear)
- **1x** Kapasitor Elektrolit Elco 1000 µF (16V / 25V)
- **1x** Adaptor Daya 5V 2A / 3A DC
- **1x** Jack DC Female Screw Terminal 5.5x2.1 mm
- **1x** Mini Breadboard 400 titik & paket kabel jumper (M-M / M-F)

---

## 📄 Lisensi
Proyek ini didistribusikan di bawah lisensi **MIT License**. Silakan gunakan, pelajari, dan kembangkan untuk kebutuhan penelitian, tugas akhir, maupun portofolio profesional Anda.