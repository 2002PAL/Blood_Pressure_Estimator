# Design of ML Assisted Beat to Beat Blood Pressure Estimator and It’s Related Hardware Implementation (Non-Invasive Method)



A cuffless, non-invasive blood pressure estimation system using PPG (Photoplethysmography) sensor module (TCRT1000), built with ESP32, Raspberry Pi 4, and I2C Display.

**Final Year B.Tech Project — Radio Physics \& Electronics, University of Calcutta (2026)**



## Table of Contents |

**-------------------------------------------**

* Overview
* System Architecture
* Hardware Required
* Circuit Connections
* Software Requirements
* Installation \& Setup
* How to Use
* ML Pipeline
* Results
* Auto-Start on Boot
* Troubleshooting

**---------------------**

## Overview

**---------------------**



This project estimates **Systolic Blood Pressure (SBP)** and **Diastolic Blood Pressure (DBP)** non-invasively using a PPG sensor placed on the fingertip. The system:

1. **ESP32** samples the PPG signal at 101 Hz and sends 1-second windows to the Raspberry Pi over USB serial
2. **Raspberry Pi** runs a trained ML model to predict SBP and DBP in real time
3. Results are displayed on a **16×2 I2C LCD** and in the terminal

The ML approach uses a **Variational Autoencoder (VAE)** to extract latent features from the raw PPG waveform, followed by a **Random Forest Regressor** that maps those features to BP values.



## System Architecture



\------------------------------------------------------------------------------------------------------

## &#x20; **PPG Sensor  ──────►  ESP32(ADC)  ──────►  Raspberry Pi**  ──────►  **I2C LCD Display \[SBP / DBP (mmHg)]**

## 

## Hardware Required



|Component|Specification|Quantity|
|-|-|-|
|Microcontroller|ESP32 (any variant with ADC)|1|
|Single Board Computer|Raspberry Pi 4 Model B|1|
|PPG Sensor Module|TCRT1000 reflective IR sensor|1|
|LCD Display|16×2 I2C LCD (PCF8574 backpack)|1|
|USB Cable|USB-A to USB-B micro/type-C (ESP32 to RPi)|1|
|Jumper Wires|Male-Female, Male-Male|assorted|

#### **------------------------------------------------------------------**

## Circuit Connections ESP32 ↔ PPG Sensor

#### **------------------------------------------------------------------**





|PPG / AFE Output|ESP32 Pin|
|-|-|
|Sensor VCC|3.3V|
|Sensor GND|GND|
|AFE Output (filtered)|GPIO34 (ADC)|
|ESP32 USB|Raspberry Pi USB port|

#### **------------------------------------------------------------------**

### Raspberry Pi ↔ LCD (I2C)

#### **------------------------------------------------------------------**



|LCD Pin|RPi Pin|GPIO|
|-|-|-|
|VCC|Pin 2|5V|
|GND|Pin 6|GND|
|SDA|Pin 3|GPIO2|
|SCL|Pin 5|GPIO3|





## Software Requirements

### Raspberry Pi (Python 3)



---bash
pip install tensorflow scikit-learn numpy pandas matplotlib \\
pyserial joblib scipy RPLCD smbus2 \\
--break-system-packages

---



### ESP32 (Arduino IDE)



No additional libraries needed — uses built-in `analogRead()` and `Serial`.

**Arduino IDE Board settings:**

* Board: `ESP32 Dev Module` (or your specific variant)
* Upload Speed: `921600`
* Flash Frequency: `80MHz`

### 

### CSV Format



All CSV files (training, test, collected) follow this column layout:

```
col 0     → SBP (mmHg)
col 1     → DBP (mmHg)
col 2     → timestamp
col 3–103 → PPG waveform samples s1...s101
```

---

## Installation & Setup

### 1. Clone the repository

```bash
git clone https://github.com/2002PAL/Blood_Pressure_Estimator.git
cd bp-estimator
```

### 2. Install Python dependencies on Raspberry Pi

```bash
pip install tensorflow scikit-learn numpy pandas matplotlib \
            pyserial joblib scipy RPLCD smbus2 \
            --break-system-packages
```

### 3. Enable I2C on Raspberry Pi (for LCD)

```bash
sudo raspi-config
# Interface Options → I2C → Enable
```

Verify LCD is detected:

```bash
sudo i2cdetect -y 1
# Should show 0x27 or 0x3F
```

If your LCD shows address `0x3F`, update `LCD_I2C_ADDRESS` in `bp_main.py`:

```python
LCD_I2C_ADDRESS = 0x3F
```

### 4. Flash ESP32 firmware

Open `ppg.ino` in Arduino IDE and flash to your ESP32. Ensure:

```cpp
#define NUM_SAMPLES  101   // must match training data waveform columns
#define SAMPLE_RATE  101   // Hz
```

### 5. Transfer files to Raspberry Pi (via SCP)

```bash
scp bp_main.py <username_of_your_system>@<your_RPi_IP>:/home/<username>/bp_project/bp_main.py
```

\---

## How to Use

Run the main script:

```bash
cd /home/ritam2002/bp_project
python3 bp_main.py
```

You will see:

```
+----------------------------------+
|      BP ESTIMATION SYSTEM        |
|  Model: Not trained              |
+----------------------------------+
|  1. Train and Save Model         |
|  2. Test on CSV                  |
|  3. Live Inference (ESP32)       |
|  4. Collect and Save Data        |
|  5. Exit                         |
+----------------------------------+
```

### First time — train the model



1. Choose **Option 1** → select `train_data_cleaned.csv`
2. Training takes \~5 minutes (VAE: 20 epochs + Random Forest: 200 trees)
3. Model is **automatically saved to disk** — you never need to retrain again





### Every time after — go straight to live inference



The saved model loads automatically at startup. Just choose **Option 3**.



### Option descriptions



|**Option**|**What it does**|
|-|-|
|1. Train \& Save|Trains VAE + RF on your CSV, saves model to disk|
|2. Test on CSV|Runs saved model on test CSV, shows MAE and SD|
|3. Live Inference|Reads live PPG from ESP32, shows SBP/DBP on LCD|
|4. Collect \& Save Data|Gathers new labelled PPG data and saves to CSV|
|5. Exit|Shuts down cleanly|

---

## ML Pipeline

**-------------------------**

### 1. Signal Acquisition

* ESP32 samples PPG at **101 Hz** using 12-bit ADC (0–4095)
* Every 1-second window (101 samples) is sent over USB serial
* Packet format: `START:v1,v2,...,v101:END\n`



### 2. Preprocessing

* Per-sample z-score normalization:  
`X_norm = (X - mean) / std`



### 3. Variational Autoencoder (VAE)

* **Encoder**: `101 → Dense(128) → Dense(64) → z_mean, z_log_var (dim=8)`
* **Decoder**: `8 → Dense(64) → Dense(128) → 101`
* **Loss**: Reconstruction loss + KL divergence
* **Epochs**: 20, Batch size: 64, Optimizer: Adam (lr=1e-3)
* Only the **encoder** is used at inference time



### 4. Random Forest Regressor

* Input: 8-dimensional latent vector `z_mean` from encoder
* Output: `[SBP, DBP]` in mmHg
* 200 trees, max depth 10



### 5. BP Classification



|Label|SBP|DBP|
|-|-|-|
|Normal|< 120|< 80|
|Elevated|120–129|< 80|
|High Stage 1|130–139|80–89|
|High Stage 2|≥ 140|≥ 90|

---

## Results



|Metric|Value|
|-|-|
|SBP MAE|11.93 mmHg|
|DBP MAE|6.05 mmHg|
|SBP SD|16.13 mmHg|
|DBP SD|8.01 mmHg|

> **Note:** Accuracy can be improved by collecting more subject-specific data using Option 4 and retraining with a larger, more diverse dataset.

---

## Auto-Start on Boot



To make the system launch automatically when the Raspberry Pi powers on, use the provided systemd service:

```bash
# Copy service file
sudo cp bp-estimator.service /etc/systemd/system/

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable bp-estimator
sudo systemctl start bp-estimator
```

**Useful commands:**

```bash
# Watch live output
sudo journalctl -u bp-estimator -f

# Stop the service
sudo systemctl stop bp-estimator

# Disable auto-start
sudo systemctl disable bp-estimator
```

\---

## Troubleshooting



|Problem|Likely Cause|Fix|
|-|-|-|
|`LCD init failed`|Wrong I2C address|Run `sudo i2cdetect -y 1`, update `LCD_I2C_ADDRESS` in config|
|`No ESP32 port found`|USB not detected|Check USB cable, run `ls /dev/ttyUSB* /dev/ttyACM*`|
|`ESP32_READY not received`|ESP32 boot time varies|Harmless — system proceeds anyway after 15s|
|`Matrix size incompatible`|NUM\_SAMPLES mismatch|Set `#define NUM_SAMPLES 101` in ESP32 code|
|`Encoder save FAILED`|Old TF/Keras version|Ensure `ENCODER_SAVE_PATH` ends in `.keras`|
|`TabError`|Mixed tabs and spaces|Run `python3 -c "open('f').read().expandtabs(4)"` to fix|
|Model not loading|Missing `.keras` or `.joblib` file|Retrain using Option 1|
|High MAE|Limited training data|Collect more data with Option 4 and retrain|

\---

## Author



**1. Ritam Pal** —  **B.Sc(H)** **Physics**, University of Calcutta (2022) \&

&#x20;            **B.Tech(ECE)**, Institute of Radiophysics \& Electronics, University of Calcutta (2026)

&#x20;
**2. Swapnil Saha** —  **B.Sc(H)** **Physics**, University of Calcutta (2023) \&

&#x20;            **B.Tech(ECE)**, Institute of Radiophysics \& Electronics, University of Calcutta (2026)




**Built as a Final Year Capstone Project. All hardware connections and ML architecture are original designs.**

