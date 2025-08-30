# Raspberry Pi Headless Setup Guide (VNC & SSH)

This guide provides a complete workflow for setting up a Raspberry Pi without needing a dedicated monitor, keyboard, or mouse (a "headless" setup). We'll use a standard PC to prepare the microSD card, connect to the Pi via SSH using **PuTTY**, and finally access the full graphical desktop using **VNC Viewer**.



---
## 📋 Table of Contents
1.  [PC Software Installation](#1--pc-software-installation)
2.  [Preparing the microSD Card with Raspberry Pi Imager](#2--preparing-the-microsd-card-with-raspberry-pi-imager)
3.  [First Boot & Connecting with PuTTY (SSH)](#3--first-boot--connecting-with-putty-ssh)
4.  [Enabling VNC for Full Desktop Access](#4--enabling-vnc-for-full-desktop-access)
5.  [Install Software & Configure Hardware](#-step-2-install-software--configure-hardware)
    * [Software Installation](#-software-installation)
    * [Configure the Hardware Serial Port](#️-configure-the-hardware-serial-port)
    * [Configure the Pixhawk](#configure-the-pixhawk)
    * [Heartbeat from Pixhawk](#heartbeat-from-pixhawk)
6.  [Confirming the Connection](#this-makes-sure-that-you-are-connected-to-pixhawk-and-responding)

---

## 1. 🖥️ PC Software Installation

Before you begin, you'll need to install three essential tools on your computer.

* ### Raspberry Pi Imager
    This is the official tool for writing Raspberry Pi OS to your microSD card. It also includes critical settings for a headless setup.
    * **Download here:** [Raspberry Pi Imager](https://www.raspberrypi.com/software/)

* ### PuTTY (for SSH)
    PuTTY is a lightweight and powerful SSH client that will give you command-line access to your Raspberry Pi over the network.
    * **Download here:** [PuTTY][(https://www.putty.org/)](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

* ### RealVNC Viewer
    This tool will allow you to see and control your Raspberry Pi's full graphical desktop from your computer.
    * **Download here:** [RealVNC Viewer](https://www.realvnc.com/en/connect/download/viewer/)

---

## 2. 💾 Preparing the microSD Card with Raspberry Pi Imager

1.  **Insert your microSD card** into your computer.
2.  **Open Raspberry Pi Imager.**
3.  **Choose your settings:**
    * **Raspberry Pi Device:** Select your model (e.g., Raspberry Pi 4).
    * **Operating System:** Click `CHOOSE OS` and select **Raspberry Pi OS (32-bit)** or **(64-bit)**. The standard version with a desktop is recommended.
    * **Storage:** Click `CHOOSE STORAGE` and select your microSD card.

4.  **Configure Advanced Options (Crucial for Headless Setup):**
    * Before writing, click the **Gear icon ⚙️** to open the Advanced Options menu. 
    * Check the box for **Enable SSH** and select "Use password authentication".
    * Check the box for **Set username and password**. Enter a username and a strong password that you will remember.
    * Check the box for **Configure wireless LAN**. Enter your WiFi network's **SSID** (name) and **password**.
    * Click **SAVE**.

5.  **Write the OS:**
    * Click the **WRITE** button. This will erase the microSD card and install Raspberry Pi OS with your settings. This process can take several minutes.

---

## 3. 🔌 First Boot & Connecting with PuTTY (SSH)

Once the write is complete, you can start your Pi and connect to it.

1.  **Eject the microSD card** from your computer and insert it into your Raspberry Pi.
2.  **Power on your Raspberry Pi.** Wait a minute or two for it to boot up and connect to your WiFi network.
3.  **Find your Pi's IP Address.** . It will likely be named `raspberrypi`.
4.  **Open PuTTY.**
    * In the **Host Name (or IP address)** field, enter the IP address you found.
    * Ensure the **Port** is `22` and the **Connection type** is `SSH`.
    * Click **Open**.

5.  **Log In:**
    * A terminal window will open. You may see a security alert on the first connection; click **Accept**.
    * When prompted for "login as:", enter the **username** you created in the Pi Imager.
    * When prompted for the password, enter the **password** you set.
    
    *   **Note: You will not see the cursor move as you type the password.** This is normal. Press `Enter` when done.

> **Success!** You should now be logged into your Raspberry Pi's command line.

---

## 4. 🖱️ Enabling VNC for Full Desktop Access

The final step is to enable the VNC server so you can access the graphical desktop. You'll do this from inside your PuTTY (SSH) session.

1.  **Run the Raspberry Pi Configuration Tool** by typing the following command and pressing `Enter`:
    ```bash
    sudo raspi-config
    ```

2.  **Enable VNC:**
    * Using your arrow keys, navigate to **`Interface Options`** and press `Enter`.
    * Navigate down to **`VNC`** and press `Enter`.
    * Select **`<Yes>`** to enable the VNC Server and press `Enter`.
   add below two lines at bottom of file sudo nano /boot/config.txt ,if VNC not working
   ```bash
   hdmi_force_hotplug=1
   hdmi_group=2
   hdmi_mode=9
   ```
4.  **Set Screen Resolution (Important!):**
    * A VNC server needs a screen resolution to function correctly without a monitor attached.
    * Back in the main menu, navigate to **`Display Options`** and press `Enter`.
    * Navigate to **`Resolution`** and press `Enter`.
    * Choose a suitable resolution like **`1280x720`** and press `Enter`.

5.  **Finish and Reboot:**
    * Press the right arrow key twice to select **`<Finish>`** in the main menu and press `Enter`.
    * You will be asked if you want to reboot. Select **`<Yes>`**. Your PuTTY session will disconnect as the Pi reboots.

6.  **Connect with RealVNC Viewer:**
    * Wait a minute for the Pi to restart.
    * Open **RealVNC Viewer** on your computer.
    * In the address bar at the top, type your Pi's **IP address** and press `Enter`.
    * A connection window will appear. Enter the **username** and **password** for your Pi.
    * Click **OK**.
  
      <img width="1920" height="1080" alt="167396310-6daf60e3-e97d-4c53-97b8-2ebb3c1907a5" src="https://github.com/user-attachments/assets/9f44503f-d2c7-4ed0-9b51-aa15b580beaa" />


You should now see and have full control of your Raspberry Pi's desktop, all without a monitor connected to it!

## 🚀 Step 2: Install Software & Configure Hardware

Now that your Raspberry Pi is running, the next step is to install the specific software needed for your drone project and configure the Pi's hardware to communicate with the Pixhawk.

---
### 📦 Software Installation

These commands will update your system and install the necessary Python libraries for your project. You can run them in a **PuTTY (SSH)** session or in a **Terminal** window on the VNC desktop.

1.  **Update your package list** to ensure you're getting the latest software versions.
    ```bash
    sudo apt-get update
    ```

2.  **Install the Python libraries** with a single command. This will fetch all the required packages for drone communication and control.
    ```bash
    sudo pip3 install pyserial
    sudo pip3 install dronekit
    sudo pip3 install geopy
    sudo pip3 install MAVProxy
    sudo pip3 install future
    ```

---
### ⚙️ Configure the Hardware Serial Port

By default, the Raspberry Pi uses its powerful hardware serial port for the onboard Bluetooth. We need to disable Bluetooth and enable the port for the GPIO pins so you can connect it to your Pixhawk.

#### A. Edit the Boot Configuration File

1.  Open the boot configuration file using the `nano` text editor.
    ```bash
    sudo nano /boot/config.txt
    ```

2.  Scroll to the **very bottom** of the file and add these two lines:
    ```ini
    enable_uart=1
    dtoverlay=disable-bt
    ```
    * `enable_uart=1`: This **enables the UART** (serial communication) hardware.
    * `dtoverlay=disable-bt`: This **disables the Bluetooth module**, which is crucial to free up the high-performance serial pins for your Pixhawk.

3.  Save the file and exit by pressing `Ctrl+X`, then `Y`, and then `Enter`.

#### B. Use the Raspberry Pi Configuration Tool

  This tool will finalize the serial port setup.

1.  Run the configuration tool:
    ```bash
    sudo raspi-config
    ```
![download](https://github.com/user-attachments/assets/0ed49823-eeb7-480c-83c5-187e908c5dbc)


2.  Using your arrow keys, navigate to **`Interface Options`** and press `Enter`.

3.  Navigate down to **`Serial Port`** and press `Enter`.

4.  You will be asked two questions:
    * "Would you like a login shell to be accessible over serial?" -> Select **`<No>`**.
    * "Would you like the serial port hardware to be enabled?" -> Select **`<Yes>`**.

5.  Navigate to **`<Finish>`** and press `Enter`. When prompted, select **`<Yes>`** to **reboot** your Raspberry Pi and apply all the changes.

6.  ### **Configure the Pixhawk**

   ![AEP_7](https://github.com/user-attachments/assets/1deecb7a-8f3a-4dc1-85fe-2afd1fce74d7)

    
    
    
  Now, you need to tell the Pixhawk to send MAVLink data out of the telemetry/usb port you just connected. You'll be doing this using a ground control station       software called Mission Planner.

   i) Connect your Pixhawk to your computer via USB.

  ii) Open Mission Planner and connect to the Pixhawk.

  iii) Go to the CONFIG → Full Parameter List screen.

  iv) Find and set the following parameters.
    
    `SERIAL2_PROTOCOL = 2`
    
    `SERIAL2_BAUD = 921`
    
    `LOG_BACKEND_TYPE = 3`

   
7.  ### Heartbeat from pixhawk
    Now type the following to get the telemetry data of pixhawk,
    ```bash
    mavproxy.py --master=/dev/serial0 --baudrate 921600   #for serial
    ```
       (or)
    
    ```bash
    mavproxy.py --master=/dev/ttyAMA0 --baudrate 921600   #for uart
    ```
       (or)                       
    ```bash
    mavproxy.py --master=/dev/ttyACM0 --baudrate 115200   #for usb
    ```
 
    ## The output must something be like

     <img width="885" height="635" alt="image" src="https://github.com/user-attachments/assets/01079a69-06b7-4cf1-aae3-a7ca12f5cc92" />


## This makes sure that you are connected to pixhawk and responding

Type the following if you want telemetry data to be displayed in mission planner

mavproxy.py --master=/dev/serial0 --baudrate 921600 --out udp:127.0.0.1:14552

Here,
 '127.0.0.1' Your PC's IP Adress, Obtained by typing 'ipconfig' in command prompt
 
 '14552' is the port to which you need to connect to mission planner using UDP

 ---
## 📹 Bonus Guides
* [**Live Camera Streaming Guide**][(./CAMERA_STREAM.md)](https://github.com/ADITYAGAVIMATH/PI-PIXHAWK-GUIDE/blob/main/Camera_stream.md) ➡️
