# Raspberry Pi and Pixhawk Integration Guide
## Complete Companion Computer Setup for Autonomous Drones (Raspberry Pi 4 & Raspberry Pi 5)

[![Raspberry Pi 4](https://img.shields.io/badge/Companion%20Computer-Raspberry%20Pi%204B-C51A4A?style=for-the-badge&logo=raspberrypi&logoColor=white)](https://www.raspberrypi.com)
[![Raspberry Pi 5](https://img.shields.io/badge/Companion%20Computer-Raspberry%20Pi%205-C51A4A?style=for-the-badge&logo=raspberrypi&logoColor=white)](https://www.raspberrypi.com)
[![ArduPilot](https://img.shields.io/badge/Autopilot-ArduPilot-orange?style=for-the-badge&logo=drone&logoColor=white)](https://ardupilot.org)
[![PX4](https://img.shields.io/badge/Autopilot-PX4%20Autopilot-blue?style=for-the-badge&logo=drone&logoColor=white)](https://px4.io)
[![Protocol](https://img.shields.io/badge/Protocol-MAVLink%202.0-blue?style=for-the-badge)](https://mavlink.io)
[![Python](https://img.shields.io/badge/Python-3.11%20VirtualEnv-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)

---

## Table of Contents
- [1. System Architecture and Hardware Wiring](#1-system-architecture-and-hardware-wiring)
- [2. Raspberry Pi 5 Setup Guide (RP1 and Bookworm 64-bit)](#2-raspberry-pi-5-setup-guide-rp1-and-bookworm-64-bit)
- [3. Raspberry Pi 4 Setup Guide (Bullseye and Bookworm)](#3-raspberry-pi-4-setup-guide-bullseye-and-bookworm)
- [4. Flight Controller Setup: ArduPilot (Mission Planner)](#4-flight-controller-setup-ardupilot-mission-planner)
- [5. Flight Controller Setup: PX4 Autopilot (QGroundControl)](#5-flight-controller-setup-px4-autopilot-qgroundcontrol)
- [6. Telemetry and Heartbeat Verification (MAVProxy and PyMAVLink)](#6-telemetry-and-heartbeat-verification-mavproxy-and-pymavlink)
- [7. Real-Time Camera Streaming](#7-real-time-camera-streaming)
- [8. Troubleshooting and Debugging](#8-troubleshooting-and-debugging)

---

## 1. System Architecture and Hardware Wiring

The companion computer (Raspberry Pi 4 or Raspberry Pi 5) communicates with the Pixhawk flight controller over a dedicated hardware UART serial link using the MAVLink protocol.

```mermaid
flowchart LR
    GCS["Ground Control Station (PC / Laptop)"] <--> |WiFi / UDP / Telemetry| PI["Raspberry Pi (Companion Computer)"]
    PI <--> |Hardware UART / MAVLink (TELEM2)| PX["Pixhawk Flight Controller"]
    CAM["CSI / USB Camera"] --> |rpicam-vid / Flask Stream| PI
```

### Physical Wiring Matrix (Raspberry Pi GPIO to Pixhawk TELEM2)

<div align="center">
  <img src="https://github.com/user-attachments/assets/1deecb7a-8f3a-4dc1-85fe-2afd1fce74d7" alt="Pixhawk to Raspberry Pi Wiring Diagram" width="750"/>
  <p><strong>Figure 1.1:</strong> Hardware wiring connections between Raspberry Pi GPIO header and Pixhawk TELEM2 port.</p>
</div>

| Raspberry Pi Pin | Pin Name | Pixhawk TELEM2 Pin | Pin Description | Signal Direction |
| :--- | :--- | :--- | :--- | :--- |
| **Pin 6** | GND | **Pin 6** | Ground | Common Reference |
| **Pin 8** | GPIO 14 (TXD) | **Pin 3** | RX | Pi Transmit → Pixhawk Receive |
| **Pin 10** | GPIO 15 (RXD) | **Pin 2** | TX | Pixhawk Transmit → Pi Receive |

> **Critical Power Isolation**:
> Do not power the Raspberry Pi from the 5V pin of the Pixhawk telemetry port. The Pixhawk power rail cannot provide sufficient current. Power the Raspberry Pi via a dedicated 5V / 3A (for Pi 4) or 5V / 5A (for Pi 5) step-down voltage regulator (BEC) connected to the main drone battery.

---

## 2. Raspberry Pi 5 Setup Guide (RP1 and Bookworm 64-bit)

### 2.1 OS Preparation via Raspberry Pi Imager
1. Insert the microSD card into your PC.
2. Open **Raspberry Pi Imager**:
   - **Device**: Select `Raspberry Pi 5`.
   - **Operating System**: Select `Raspberry Pi OS (64-bit)` (Debian 12 Bookworm).
   - **Storage**: Select your microSD card.
3. Click **Next** → **Edit Settings**:
   - Set Hostname (e.g., `pi5-drone`), Username, and Password.
   - Enter your Wireless LAN SSID and Password.
   - Enable SSH (**Services** → **Enable SSH** → *Use password authentication*).
4. Save settings and write the OS to the card.

---

### 2.2 SSH and Remote Desktop Setup (Wayland / WayVNC)
1. Insert the card into the Raspberry Pi 5, connect power, and wait 60 seconds for boot.
2. Open a terminal or PuTTY on your PC and establish an SSH connection:
   ```bash
   ssh <username>@<RASPBERRY_PI_IP>
   ```

#### Graphical Desktop Access on Pi 5:
* **Option A: WayVNC (Default on Bookworm)**:
  - Run `sudo raspi-config` → `Interface Options` → `VNC` → Select `Yes`.
  - Connect from your PC using **TigerVNC** or **RealVNC Viewer (v7.x+)** to `<RASPBERRY_PI_IP>:5900`.
* **Option B: Switch Display Server to X11**:
  - Run `sudo raspi-config` → `Advanced Options` → `Wayland` → Select `X11 (Openbox)`.
  - Reboot via `sudo reboot`.

---

### 2.3 Hardware UART Configuration on Pi 5
On Raspberry Pi 5, the primary hardware UART is routed through the RP1 southbridge chip and configured in `/boot/firmware/config.txt`.

#### Step 1: Enable Serial Port in raspi-config
1. Run `sudo raspi-config`.
2. Navigate to: `Interface Options` → `Serial Port`.
3. *Login shell over serial*: Select **No**.
4. *Serial port hardware enabled*: Select **Yes**.

#### Step 2: Configure `/boot/firmware/config.txt`
```bash
sudo nano /boot/firmware/config.txt
```
Add the following configuration at the bottom of the file:
```ini
# Enable primary UART on GPIO 14/15 for Pixhawk Telemetry
enable_uart=1
dtoverlay=uart0
```
Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

#### Step 3: Disable Serial Kernel Console Service
```bash
sudo systemctl stop serial-getty@ttyAMA0.service
sudo systemctl disable serial-getty@ttyAMA0.service
sudo reboot
```

> On Raspberry Pi 5, the primary hardware serial port on GPIO 14/15 is `/dev/ttyAMA0` (aliased as `/dev/serial0`).

---

### 2.4 Python Virtual Environment and MAVLink Installation (PEP 668)
Debian 12 Bookworm requires Python packages to be managed in an isolated virtual environment.

```bash
# 1. Install system build prerequisites
sudo apt update
sudo apt install -y python3-pip python3-venv python3-dev build-essential libxml2-dev libxslt-dev

# 2. Create and activate a dedicated virtual environment
mkdir -p ~/drone_ws && cd ~/drone_ws
python3 -m venv venv --system-site-packages
source ~/drone_ws/venv/bin/activate

# 3. Install MAVLink communication libraries
pip install --upgrade pip
pip install pymavlink mavproxy dronekit pyserial
```

To automatically activate the virtual environment upon login:
```bash
echo "source ~/drone_ws/venv/bin/activate" >> ~/.bashrc
```

---

## 3. Raspberry Pi 4 Setup Guide (Bullseye and Bookworm)

### 3.1 OS Preparation and SSH Access
Follow the same Raspberry Pi Imager procedure as Section 2.1, selecting **Raspberry Pi 4** as the target device.

---

### 3.2 Graphical Desktop Setup (RealVNC on Pi 4)
1. In your SSH session, run:
   ```bash
   sudo raspi-config
   ```
2. Navigate to `Interface Options` → `VNC` → Select `Yes`.
3. Navigate to `Display Options` → `Resolution` → Select `1280x720`.
4. Finish and reboot.
5. Connect using **RealVNC Viewer** from your PC.

<div align="center">
  <img src="https://github.com/user-attachments/assets/9f44503f-d2c7-4ed0-9b51-aa15b580beaa" alt="RealVNC Desktop Access on Raspberry Pi 4" width="850"/>
  <p><strong>Figure 3.1:</strong> Remote desktop session running over RealVNC Viewer on Raspberry Pi 4.</p>
</div>

---

### 3.3 Hardware UART Configuration on Pi 4
On Raspberry Pi 4, the primary hardware UART must be freed from onboard Bluetooth.

#### Step 1: Edit `/boot/config.txt`
```bash
sudo nano /boot/config.txt
```
Append the following lines to the end of the file:
```ini
enable_uart=1
dtoverlay=disable-bt
```
Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

#### Step 2: Disable Bluetooth Service and Configure Serial Port
```bash
sudo systemctl disable hciuart
sudo raspi-config
```

<div align="center">
  <img src="https://github.com/user-attachments/assets/0ed49823-eeb7-480c-83c5-187e908c5dbc" alt="raspi-config Interface Options" width="750"/>
  <p><strong>Figure 3.2:</strong> raspi-config interface configuration for serial communication.</p>
</div>

1. Navigate to: `Interface Options` → `Serial Port`.
2. *Login shell over serial*: Select **No**.
3. *Serial port hardware enabled*: Select **Yes**.
4. Finish and reboot the Raspberry Pi (`sudo reboot`).

---

### 3.4 Software Installation (Pi 4)
```bash
sudo apt-get update
sudo apt-get install -y python3-pip python3-dev
pip3 install pyserial dronekit geopy MAVProxy future
```

---

## 4. Flight Controller Setup: ArduPilot (Mission Planner)

When using **ArduPilot (ArduCopter / ArduPlane)** firmware on the Pixhawk:

### Step 1: Connect Pixhawk to Mission Planner
1. Connect the Pixhawk to your PC using a USB cable.
2. Open **Mission Planner** and click **Connect** in the top-right corner.
3. Navigate to: **CONFIG** → **Full Parameter List** (or **Full Parameter Tree**).

### Step 2: Configure TELEM2 Port Parameters

| Parameter | Value | Description |
| :--- | :--- | :--- |
| `SERIAL2_PROTOCOL` | `2` | MAVLink 2 protocol |
| `SERIAL2_BAUD` | `921` (921600 baud) or `57` (57600 baud) | UART serial transmission baud rate |
| `BRD_SER2_RTSCTS` | `0` | Disable hardware flow control (3-wire connection) |
| `LOG_BACKEND_TYPE` | `3` | Enable data logging backend for companion computer |

4. Click **Write Params** and power cycle the Pixhawk.

---

## 5. Flight Controller Setup: PX4 Autopilot (QGroundControl)

When using **PX4 Autopilot** firmware on the Pixhawk:

### Step 1: Connect Pixhawk to QGroundControl
1. Connect the Pixhawk to your PC via USB.
2. Open **QGroundControl** → Click the **Q** icon → **Vehicle Setup** → **Parameters**.

### Step 2: Configure TELEM2 MAVLink Instance

| Parameter | Value | Description |
| :--- | :--- | :--- |
| `MAV_1_CONFIG` | `TELEM 2` | Assigns MAVLink instance 1 to the TELEM2 port |
| `SER_TEL2_BAUD` | `921600` (or `57600`) | Serial baud rate matching Raspberry Pi |
| `MAV_1_MODE` | `Onboard` (or `Normal`) | Optimizes stream rates for companion computer |
| `MAV_1_RATE` | `0` (or target bandwidth) | Maximum data stream rate (0 = automatic default) |
| `MAV_1_FORWARD` | `1` | Enables forwarding between MAVLink instances |

3. Click **Save** and reboot the Pixhawk via the QGroundControl Tools menu.

---

## 6. Telemetry and Heartbeat Verification (MAVProxy and PyMAVLink)

### 6.1 Running MAVProxy Telemetry Router
MAVProxy reads telemetry from the hardware serial port and forwards it to local scripts and external Ground Control Stations over UDP.

* **For Raspberry Pi 5** (`/dev/ttyAMA0` or `/dev/serial0`):
  ```bash
  mavproxy.py --master=/dev/ttyAMA0 --baudrate 921600 --out=127.0.0.1:14550 --out=<GCS_IP_ADDRESS>:14550
  ```

* **For Raspberry Pi 4** (`/dev/serial0`):
  ```bash
  mavproxy.py --master=/dev/serial0 --baudrate 921600 --out=udp:127.0.0.1:14550 --out=udp:<GCS_IP_ADDRESS>:14550
  ```

> Replace `<GCS_IP_ADDRESS>` with the IP address of your laptop running Mission Planner or QGroundControl (obtainable via `ipconfig` on Windows).

<div align="center">
  <img src="https://github.com/user-attachments/assets/01079a69-06b7-4cf1-aae3-a7ca12f5cc92" alt="MAVProxy Telemetry Output" width="850"/>
  <p><strong>Figure 6.1:</strong> Successful MAVLink heartbeat reception and vehicle telemetry status displayed in MAVProxy console.</p>
</div>

---

### 6.2 Python Verification Script (`check_heartbeat.py`)
Create a standalone verification script to test direct MAVLink communication:

```python
from pymavlink import mavutil
import time

# Define serial device and baud rate
# Use '/dev/ttyAMA0' for Pi 5 or '/dev/serial0' for Pi 4
SERIAL_PORT = '/dev/ttyAMA0'
BAUD_RATE = 921600

print(f"Connecting to Pixhawk on {SERIAL_PORT} at {BAUD_RATE} baud...")
connection = mavutil.mavlink_connection(SERIAL_PORT, baud=BAUD_RATE)

# Wait for the first heartbeat packet
print("Waiting for heartbeat packet from Pixhawk...")
connection.wait_heartbeat()
print(f"Heartbeat received from System ID: {connection.target_system}, Component ID: {connection.target_component}")

# Continuously display vehicle flight mode and armed status
while True:
    msg = connection.recv_match(type='HEARTBEAT', blocking=True)
    if msg:
        flight_mode = mavutil.mode_string_v10(msg)
        is_armed = bool(msg.base_mode & mavutil.mavlink.MAV_MODE_FLAG_SAFETY_ARMED)
        print(f"Flight Mode: {flight_mode:<12} | Armed Status: {'ARMED' if is_armed else 'DISARMED'}")
    time.sleep(1)
```

Run the script:
```bash
python check_heartbeat.py
```

---

## 7. Real-Time Camera Streaming

### 7.1 Raspberry Pi 5: Hardware-Accelerated Video Stream (`rpicam-vid`)
Install dependencies:
```bash
sudo apt install -y rpicam-apps gstreamer1.0-tools
```
Stream low-latency H.264 video over UDP to QGroundControl or Mission Planner:
```bash
rpicam-vid -t 0 --inline --width 1280 --height 720 --framerate 30 --codec h264 -o udp://<GCS_IP_ADDRESS>:5600
```
*In QGroundControl: Application Settings → Video → Source = `UDP h.264 Video Stream`, Port = `5600`.*

---

### 7.2 Raspberry Pi 4: Flask Web Streaming Server
Detailed instructions and Python implementation using `picamera2`, `OpenCV`, and `Flask` are available in [Camera_stream.md](Camera_stream.md).

```bash
python3 cam.py
```
Access the live video stream from any web browser on the same network at:
```text
http://<RASPBERRY_PI_IP>:5000
```

---

## 8. Troubleshooting and Debugging

| Symptom | Root Cause | Solution |
| :--- | :--- | :--- |
| `Permission denied: '/dev/ttyAMA0'` | Current user is not in the `dialout` group | Run `sudo usermod -a -G dialout $USER`, then log out and log back in. |
| `error: externally-managed-environment` | Debian 12 (Bookworm) PEP 668 policy | Activate the Python virtual environment (`source ~/drone_ws/venv/bin/activate`). |
| MAVProxy hangs on *"Waiting for heartbeat"* | Mismatched baud rate or swapped RX/TX wires | Verify Pin 8 (Pi TX) goes to Pixhawk RX, Pin 10 (Pi RX) to Pixhawk TX, and ensure baud rates match (`SERIAL2_BAUD` / `SER_TEL2_BAUD`). |
| VNC viewer displays a black screen on Pi 5 | Wayland display server authentication incompatibility | Connect with TigerVNC Viewer or switch display server to X11 in `raspi-config`. |
| Pixhawk resets or reboots when Pi boots | Voltage drop on power rail | Power the Raspberry Pi from an independent 5V BEC rather than the Pixhawk power port. |
