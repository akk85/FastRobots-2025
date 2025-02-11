# **Lab Report: IMU Setup and Data Analysis**
**Author:** Tony Kariuki  
**Date:** February 10, 2025  

## **1. Lab Tasks Overview**
This lab focuses on setting up and testing the ICM-20948 IMU on the Artemis board, collecting accelerometer and gyroscope data, analyzing sensor accuracy, and transmitting IMU data over Bluetooth. The final task involves using the IMU to record a stunt.

---

## **2. IMU Setup**
### **IMU Hardware Connections**
The ICM-20948 IMU is connected to the Artemis board via I2C using the following connections:

| **IMU Pin** | **Artemis Pin** |
|------------|--------------|
| SDA (I2C Data) | SDA (A4) |
| SCL (I2C Clock) | SCL (A5) |
| VCC | 3.3V |
| GND | GND |

**Image: IMU Connections**  
![IMU Connection Diagram](path_to_image.png)

### **IMU Example Code Verification**
To verify functionality, I ran the **ICM-20948 example code**, confirming that acceleration and gyroscope readings were correctly displayed in the serial monitor.

### **AD0_VAL Definition**
The **AD0_VAL** bit determines the IMU’s I2C address:
- `AD0_VAL = 1` (Default, I2C Address: 0x69)
- `AD0_VAL = 0` (When ADR jumper is closed, I2C Address: 0x68)

Since I kept the default configuration, **AD0_VAL = 1**.

---

## **3. Accelerometer Data Analysis**
### **Pitch and Roll Data at {-90, 0, 90} Degrees**
The accelerometer data was recorded at:
- **0° (Flat)**
- **-90° (Upside Down)**
- **90° (Sideways)**

**Equation for Pitch and Roll Calculation**  
The IMU computes pitch and roll from accelerometer readings using:

\[
\theta_{\text{pitch}} = \arctan \left(\frac{a_y}{a_z}\right) \times \frac{180}{\pi}
\]

\[
\theta_{\text{roll}} = \arctan \left(\frac{a_x}{a_z}\right) \times \frac{180}{\pi}
\]

**Images: Accelerometer Output**  
- *Pitch and Roll at -90°, 0°, and 90°*
  ![Accelerometer Data](path_to_image.png)

### **Accelerometer Accuracy and Noise Analysis**
A Fourier Transform was applied to analyze **noise** in the accelerometer data. High-frequency components indicate measurement noise.

**Graph: Fourier Transform of Accelerometer Data**  
![Fourier Transform](path_to_graph.png)

- The results show minor fluctuations due to external vibrations and noise from the environment.
- Filtering techniques (e.g., low-pass filter) could be applied to smooth the signal.

---

## **4. Gyroscope Data Analysis**
### **Pitch, Roll, and Yaw Readings**
The gyroscope measures angular velocity in **degrees per second (°/s)**. Over time, integrating these values gives an estimate of orientation.

\[
\theta_{\text{pitch}} = \theta_{\text{pitch}} + \omega_y \cdot dt
\]
\[
\theta_{\text{roll}} = \theta_{\text{roll}} + \omega_x \cdot dt
\]
\[
\theta_{\text{yaw}} = \theta_{\text{yaw}} + \omega_z \cdot dt
\]

**Images: Gyroscope Output at Different Positions**  
- *Yaw Change While Rotating IMU*  
  ![Gyroscope Data](path_to_image.png)

### **Complementary Filter for Smoother Readings**
To reduce **drift** in gyroscope readings, I implemented a **complementary filter**, which combines accelerometer and gyroscope data:

\[
\theta_{\text{comp}} = \alpha (\theta_{\text{comp}} + \omega \cdot dt) + (1 - \alpha) \cdot \theta_{\text{accel}}
\]

where **α = 0.98** (favoring gyroscope readings).  
- The complementary filter **improves stability**, making pitch and roll measurements more reliable.

---

## **5. Sample Data Collection**
### **IMU Sampling Speed**
- Without delays, the **IMU sampling rate** was **~250 Hz**.
- The **Artemis executes loops at ~48 MHz**, so sampling does **not** block execution.

### **Stored Time-Stamped IMU Data**
IMU data (pitch & roll) was stored in **arrays** and transmitted over BLE.

| Sample | Timestamp (ms) | Pitch (°) | Roll (°) |
|--------|--------------|----------|---------|
| 0 | 100 | -1.23 | 0.45 |
| 1 | 104 | -1.24 | 0.43 |
| ... | ... | ... | ... |
| 50 | 500 | -1.31 | 0.38 |

- The Artemis **stored 5s of IMU data** before sending it via BLE.

### **IMU Data Over Bluetooth**
The IMU data was sent to a **Python script** over BLE, using the `START_IMU_DATA` and `STOP_IMU_DATA` commands.

```python
# Start BLE notifications
ble.start_notify(ble.uuid['RX_STRING'], imu_notification_handler)

# Start collecting IMU data
ble.send_command(CMD.START_IMU_DATA.value, "")

# Collect for 5 seconds
time.sleep(5)

# Stop collecting
ble.send_command(CMD.STOP_IMU_DATA.value, "")
