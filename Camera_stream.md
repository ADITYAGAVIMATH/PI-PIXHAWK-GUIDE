## 📹 Bonus Step: Streaming Live Video from the Camera

This section will guide you through setting up a simple web server on your Raspberry Pi to stream live video from its camera. You'll be able to view the feed from any web browser on the same network.

---
### 1. Enable the Camera Interface

First, ensure your Raspberry Pi's camera interface is enabled.

1.  Run the configuration tool:
    ```bash
    sudo raspi-config
    ```
2.  Navigate to **`Interface Options`** → **`Legacy Camera`**.
3.  Select **`<Yes>`** to enable the legacy camera support and reboot when prompted.

---
### 2. Install Prerequisites

The Python script requires a few libraries to function. You'll need to install system-level dependencies for the camera and the Python packages for the web server and image processing.

1.  **Install system libraries for OpenCV:**
    ```bash
    sudo apt-get update
    sudo apt-get install -y python3-opencv libcamera-apps
    ```
2.  **Install the required Python packages:**
    ```bash
    pip3 install flask
    pip3 install picamera2
    pip3 install opencv-python
    pip3 install numpy
    ```
> **Note:** `picamera2` is often pre-installed on recent Raspberry Pi OS images, but running the command ensures it's there.

---
### 3. 🐍 The Python Code

This script uses the `picamera2` library to capture frames, `OpenCV` to process them, and `Flask` to serve them over the web.

1.  Create a new file on your Raspberry Pi called `cam.py`:
    ```bash
    nano cam.py
    ```
2.  Copy and paste the entire code block below into the `nano` editor.

    ```python
    from flask import Flask, Response
    from picamera2 import Picamera2
    import cv2
    import numpy as np

    app = Flask(__name__)

    # Initialize the camera
    picam2 = Picamera2()
    config = picam2.create_video_configuration(main={"size": (640, 480)})  # 640x480 for smoother streaming
    picam2.configure(config)
    picam2.start()

    def generate_frames():
        while True:
            # Capture frame from the camera
            frame = picam2.capture_array()

            # Fix potential purple tint (convert from RGB to BGR for OpenCV)
            frame = cv2.cvtColor(frame, cv.COLOR_RGB2BGR)

            # Encode the frame as a JPEG image
            ret, buffer = cv2.imencode('.jpg', frame)
            if not ret:
                continue
            frame = buffer.tobytes()

            # Yield the frame in the multipart format
            yield (b'--frame\r\n'
                   b'Content-Type: image/jpeg\r\n\r\n' + frame + b'\r\n')

    @app.route('/')
    def index():
        # Main page with the video feed embedded
        return """
        <html>
            <body style="background-color: #333; color: white; text-align: center;">
                <h1>Raspberry Pi Camera Stream</h1>
                <img src="/video_feed" style="width: 640px; height: 480px; border: 2px solid #555;">
            </body>
        </html>
        """

    @app.route('/video_feed')
    def video_feed():
        # Route that serves the video stream
        return Response(generate_frames(), mimetype='multipart/x-mixed-replace; boundary=frame')

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=5000, threaded=True)

    ```
3.  Save the file and exit by pressing `Ctrl+X`, then `Y`, and then `Enter`.

---
### 4. 🚀 Run the Application

Now you can run the web server.

1.  In your terminal, execute the script:
    ```bash
    python3 cam.py
    ```
2.  You should see output indicating the server is running, like this:
    ```
     * Serving Flask app 'cam'
     * Running on all addresses.
       WARNING: This is a development server. Do not use it in a production deployment.
     * Running on http://YOUR_PI_IP_ADDRESS:5000
    ```

---
### 5. 🌐 View the Stream

1.  Find your Raspberry Pi's IP address (you can use the `hostname -I` command in another terminal window).
2.  On another computer, tablet, or phone that is connected to the **same WiFi network**, open a web browser.
3.  In the address bar, type `http://` followed by your Pi's IP address and the port number `:5000`.
    * **Example:** `http://192.168.1.10:5000`

You should now see a webpage with a live video feed from your Raspberry Pi's camera!
