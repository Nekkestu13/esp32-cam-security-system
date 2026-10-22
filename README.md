# ESP32-CAM Home Security System

A low-cost home security prototype built around an **ESP32-CAM**, a **magnetic door/window contact sensor** and **Gmail SMTP**. When the sensor detects that a door or window has been opened, the device captures a photo, emails it as an alert and exposes a live video stream on the local network.

> **Stack:** ESP32-CAM (OV2640) · Arduino IDE · C/C++ · Wi-Fi · SMTP (Gmail) · MJPEG streaming

---

## Table of Contents

1. [Overview](#1-overview)
2. [Hardware and Software](#2-hardware-and-software)
3. [Configuration](#3-configuration)
4. [How It Works](#4-how-it-works)
5. [Troubleshooting](#5-troubleshooting)
6. [Conclusion](#6-conclusion)

---

## 1. Overview

### 1.1 Project Description

This project implements a simple yet effective home security system using an ESP32-CAM, a magnetic contact sensor and an SMTP mail service. When the two parts of the sensor are separated, indicating a possible intrusion, the system sends an email alert with a photo and provides access to a live video stream.

### 1.2 Objectives

- Understand how a home security system works end to end.
- Gain hands-on experience programming the ESP32-CAM.
- Integrate several components (sensor, camera, Wi-Fi, mail service) into a working system.

### 1.3 Features

- Door/window opening detection via a magnetic contact sensor
- Automatic photo capture on detection
- Email alert with the photo attached, sent through Gmail SMTP
- Live MJPEG video stream accessible from any browser on the local network
- Alert throttling to prevent repeated emails while the door remains open

---

## 2. Hardware and Software

### 2.1 Hardware

| Component | Role |
|---|---|
| **ESP32-CAM** | Compact development board based on an ESP32-S module with an OV2640 camera. Well suited to low-cost IoT applications such as home monitoring. |
| **Magnetic contact sensor** (two magnets/reed switch) | Acts as a switch. When the two parts are close together the circuit is closed; when they are separated the circuit opens, signalling that a door or window has been opened. |
| **10 kΩ resistor** | Pull-down resistor keeping the input pin at a defined LOW level when the circuit is open. |

### 2.2 Email Alerts over SMTP

SMTP (Simple Mail Transfer Protocol) is used to send the alert emails. A dedicated Gmail account was created for the project. Using a Gmail **App Password**, the ESP32-CAM can authenticate directly with the SMTP server, without an interactive login, and send an email as soon as the alarm is triggered.

### 2.3 Software and Libraries

The code is uploaded to the ESP32-CAM with the **Arduino IDE**. The following libraries are used:

| Library | Purpose |
|---|---|
| `esp_camera.h` | Initializes and controls the camera, captures images and sets camera parameters. |
| `SPI.h` | Arduino's built-in library for SPI communication with external peripherals (sensors, displays, SD modules). |
| `drivers/rtc_io.h` | Part of the ESP-IDF SDK. Provides control of the RTC I/O pins, which keep working in low-power modes. |
| `ESP_Mail_Client.h` | Sends emails (plain text, HTML, attachments) over SMTP/SMTPS/STARTTLS from an ESP32 or ESP8266. |
| `FS.h` | File system access (LittleFS/SPIFFS): read, write, create, delete and rename files. Used to store the captured photo. |
| `WiFi.h` | Wi-Fi connection management (scan, connect, access point). |
| `HTTPClient.h` | HTTP/HTTPS client for communicating with web servers. |
| `camera_pins.h` | Pin definitions for the supported ESP32-CAM board models (shipped with the ESP32 `CameraWebServer` example). |

---

## 3. Configuration

### 3.1 Camera Configuration

The camera is configured through a `camera_config_t` structure that is passed to `esp_camera_init(&config)`.

```cpp
camera_config_t config;
config.ledc_channel = LEDC_CHANNEL_0;
config.ledc_timer   = LEDC_TIMER_0;
config.pin_d0 = Y2_GPIO_NUM;
// ... pin_d1 to pin_d7, XCLK, PCLK, VSYNC, HREF, SCCB, PWDN, RESET
config.xclk_freq_hz = 20000000;
config.frame_size   = FRAMESIZE_UXGA;
config.pixel_format = PIXFORMAT_JPEG;       // JPEG for streaming
config.grab_mode    = CAMERA_GRAB_WHEN_EMPTY;
config.fb_location  = CAMERA_FB_IN_PSRAM;
config.jpeg_quality = 12;
config.fb_count     = 1;

// If PSRAM is available: higher quality and double buffering
if (config.pixel_format == PIXFORMAT_JPEG) {
  if (psramFound()) {
    config.jpeg_quality = 10;
    config.fb_count     = 2;
    config.grab_mode    = CAMERA_GRAB_LATEST;
  } else {
    // Limit the frame size when PSRAM is not available
    config.frame_size  = FRAMESIZE_SVGA;
    config.fb_location = CAMERA_FB_IN_DRAM;
  }
}
```

Key parameters:

| Parameter | Description |
|---|---|
| `ledc_channel`, `ledc_timer` | LEDC (PWM) channel and timer used to generate the camera's external clock signal. |
| `pin_d0` – `pin_d7` | Data pins used to transfer image data from the camera to the ESP32. |
| `pin_xclk`, `pin_pclk` | External clock and pixel clock pins. |
| `pin_vsync`, `pin_href` | Vertical and horizontal synchronization pins. |
| `pin_sccb_sda`, `pin_sccb_scl` | SCCB (I²C-compatible) pins used to control the camera sensor. |
| `pin_pwdn`, `pin_reset` | Camera power-down and reset pins. |
| `xclk_freq_hz` | External clock frequency (default: 20 MHz). |
| `frame_size` | Resolution, from `FRAMESIZE_QQVGA` (160×120) up to `FRAMESIZE_UXGA` (1600×1200). |
| `pixel_format` | `PIXFORMAT_GRAYSCALE`, `PIXFORMAT_RGB565` or `PIXFORMAT_JPEG` (compressed). |
| `grab_mode` | `CAMERA_GRAB_WHEN_EMPTY` or `CAMERA_GRAB_LATEST`. |
| `fb_location` | Frame buffer in internal DRAM or external PSRAM. |
| `jpeg_quality` | JPEG compression level, 0–63. **A lower value means higher quality** (and larger files). |
| `fb_count` | Number of frame buffers. More buffers improve throughput at the cost of memory. |

### 3.2 Wi-Fi Configuration

Wi-Fi is configured in `setup()`:

```cpp
Serial.begin(115200);
Serial.setDebugOutput(true);

WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
WiFi.setSleep(false);

while (WiFi.status() != WL_CONNECTED) {
  delay(500);
  Serial.print(".");
}
Serial.print("IP Address: http://");
Serial.println(WiFi.localIP());
```

1. **Serial communication** is initialized at 115200 baud for debug output.
2. **`WiFi.begin(WIFI_SSID, WIFI_PASSWORD)`** connects the board to the configured network.
3. **`WiFi.setSleep(false)`** disables Wi-Fi power saving to keep the connection stable.
4. A **`while` loop** waits until the connection is established, then prints the board's IP address.

### 3.3 SMTP (Gmail) Configuration

The following steps were used to send emails with a photo attachment through a Gmail account:

1. **Create a dedicated Gmail account** for the project.
2. **Enable 2-Step Verification** in the account's security settings (<https://myaccount.google.com/security>).
3. **Generate an App Password.** In the security settings, open *App passwords*, create a new password with a custom name (e.g. `ESP32-CAM`) and copy the 16-character password that is generated.
4. **Add the credentials to the code** (see below).

> ⚠️ **Never commit credentials to GitHub.** Keep the Wi-Fi password, the Gmail address and the App Password in a separate `secrets.h` file that is listed in `.gitignore`, and publish a `secrets.example.h` with placeholder values instead.

```cpp
// secrets.h (not committed)
#define WIFI_SSID            "your_wifi_ssid"
#define WIFI_PASSWORD        "your_wifi_password"
#define emailSenderAccount   "sender@gmail.com"
#define emailSenderPassword  "your_app_password_here"
#define emailRecipient       "recipient@example.com"

// Constants used by the sketch
#define smtpServer      "smtp.gmail.com"
#define smtpServerPort  465
#define emailSubject    "ESP32-CAM Photo Captured"
```

The SMTP session is configured inside `sendPhoto()`:

```cpp
Session_Config config;
config.server.host_name  = smtpServer;
config.server.port       = smtpServerPort;
config.login.email       = emailSenderAccount;
config.login.password    = emailSenderPassword;
config.login.user_domain = "";
```

The sketch then calls `smtp.connect(&config)` to open the session and `MailClient.sendMail(&smtp, &message, true)` to send the email with the captured photo.

### 3.4 Camera Web Server

`startCameraServer()` starts the web server used to control the camera and stream video. It relies on the ESP-IDF HTTP server (`httpd`) component:

1. **Server configuration:** an `httpd_config_t` is created from `HTTPD_DEFAULT_CONFIG()` with `max_uri_handlers` set to 16.
2. **URI definitions:** several `httpd_uri_t` endpoints are declared (`/`, `/status`, `/control`, `/capture`, `/stream`, …), each bound to an HTTP GET handler (`index_handler`, `status_handler`, `cmd_handler`, …).
3. **Moving-average filter:** `ra_filter_init()` initializes a running-average filter used to smooth frame-timing statistics.
4. **Face recognition (optional):** enabled only if `CONFIG_ESP_FACE_RECOGNITION_ENABLED` is set.
5. **Server start:** `httpd_start()` launches two servers, `camera_httpd` for control and configuration requests and `stream_httpd` for the video stream, then registers the URI handlers with `httpd_register_uri_handler()`.

---

## 4. How It Works

### System Architecture

![System architecture](docs/architecture.svg)

### 4.1 Detecting the Door Opening

![Wiring diagram](docs/wiring-diagram.png)

**Wiring (see diagram above):**

1. One terminal of the magnetic contact sensor is connected to the **3.3 V** pin of the ESP32.
2. The other terminal is connected to **GPIO12**, which reads the sensor state.
3. A **10 kΩ pull-down resistor** connects GPIO12 to **GND**, holding the pin LOW whenever the sensor circuit is open.

**Logic:**

| Door state | Sensor circuit | GPIO12 reading |
|---|---|---|
| Closed (magnets together) | Closed | `HIGH` |
| Open (magnets separated) | Open | `LOW` (pulled down) |

**Main loop:**

```cpp
void loop() {
  // LOW means the door is open
  if (digitalRead(doorPin) == LOW) {
    delay(2000);                 // Short delay before taking the picture
    capturePhotoSaveLittleFS();  // Capture and store the photo
    sendPhoto();                 // Send the alert email
    delay(60000 * 5);            // Wait 5 minutes to avoid repeated alerts

    // Wait for the door to close before re-arming
    while (digitalRead(doorPin) == LOW) {
      delay(1000);
    }
  }
  delay(100);  // Reduce CPU load
}
```

1. The sensor state is read with `digitalRead(doorPin)`. `LOW` means the door is open.
2. When an opening is detected, the program waits 2 seconds before capturing, so the photo is not taken at the exact moment of opening and has a better chance of showing the intruder.
3. `capturePhotoSaveLittleFS()` captures a photo and saves it to the ESP32's LittleFS flash file system.
4. `sendPhoto()` sends the photo by email.
5. A 5-minute cooldown (`delay(60000 * 5)`) prevents a flood of emails while the door stays open.
6. A `while` loop then waits until the door is closed again (`HIGH`) before the system re-arms.
7. A final 100 ms delay at the end of `loop()` limits CPU usage.

### 4.2 Sending the Alert Email

`sendPhoto()` sends the alert email with the photo attached over SMTP:

```cpp
void sendPhoto(void) {
  smtp.debug(1);                  // Basic debug output on the serial port
  smtp.callback(smtpCallback);    // Status callback

  Session_Config config;
  // Time/NTP settings: adjust the offsets to your time zone
  config.time.ntp_server       = F("pool.ntp.org,time.nist.gov");
  config.time.gmt_offset       = 1;
  config.time.day_light_offset = 1;

  config.server.host_name  = smtpServer;
  config.server.port       = smtpServerPort;
  config.login.email       = emailSenderAccount;
  config.login.password    = emailSenderPassword;
  config.login.user_domain = "";

  SMTP_Message message;
  message.enable.chunking = true;
  message.sender.name  = "ESP32-CAM";
  message.sender.email = emailSenderAccount;
  message.subject      = emailSubject;
  message.addRecipient("Recipient", emailRecipient);

  String htmlMsg = "<h2>Photo captured with ESP32-CAM and attached to this email.</h2>"
                   "<h3>Watch the live stream: http://" + WiFi.localIP().toString() + "</h3>";
  message.html.content           = htmlMsg.c_str();
  message.html.charSet           = "utf-8";
  message.html.transfer_encoding = Content_Transfer_Encoding::enc_qp;

  SMTP_Attachment att;
  att.descr.filename          = FILE_PHOTO;
  att.descr.mime              = "image/jpeg";
  att.file.path               = FILE_PHOTO_PATH;
  att.file.storage_type       = esp_mail_file_storage_type_flash;
  att.descr.transfer_encoding = Content_Transfer_Encoding::enc_base64;
  message.addAttachment(att);

  if (!smtp.connect(&config)) return;

  if (!MailClient.sendMail(&smtp, &message, true))
    Serial.println("Error sending Email, " + smtp.errorReason());
}
```

Step by step:

1. Debug output is enabled for the `ESP_Mail_Client` library, and a callback (`smtpCallback`) reports the sending status.
2. A `Session_Config` is filled with the SMTP server, port and credentials.
3. An `SMTP_Message` is created with the sender, recipient, subject and HTML body. The body contains an alert text and a link to the camera's local IP address for the live stream.
4. The photo saved in flash is added as an attachment.
5. `smtp.connect()` opens the session with the SMTP server.
6. `MailClient.sendMail()` sends the email and, because the last argument is `true`, closes the session afterward.
7. If sending fails, the error reason is printed to the serial monitor.

### 4.3 Accessing the Camera and Live Stream

The built-in web server provides a browser interface for the live video feed and camera controls:

- **Home page (`/`)**: served by `index_handler()`. It links to the camera features (capture, stream, settings).
- **Image capture (`/capture`)**: `capture_handler()` takes a photo and returns it as an HTTP response.
- **Camera control**: `cmd_handler()`, `xclk_handler()`, `reg_handler()`, `greg_handler()`, `pll_handler()` and `win_handler()` handle settings such as resolution, gain and exposure.
- **Live stream (`/stream`)**: `stream_handler()` serves an **MJPEG** stream.

To watch the stream, open the following address in a browser (or in a media player that supports MJPEG, such as VLC):

```
http://<ESP32-CAM_IP_ADDRESS>/stream
```

where `<ESP32-CAM_IP_ADDRESS>` is the IP address printed on the serial monitor at startup (also included in the alert email).

---

## 5. Troubleshooting

| Symptom | Possible cause and fix |
|---|---|
| Email is not sent / authentication error | Check that 2-Step Verification is enabled, that you are using a 16-character **App Password** (not the account password) and that port `465` is used with `smtp.gmail.com`. |
| Garbled output in the serial monitor | Make sure the serial monitor baud rate matches `Serial.begin(115200)`. |
| Camera initialization fails or the board resets | Check the camera ribbon cable and the camera pin definitions for your board model. Use a stable 5 V power supply, since weak USB power often causes brownouts. |
| Stream is slow or unstable | Lower `frame_size` (e.g. `FRAMESIZE_VGA`), increase `jpeg_quality` (lower quality), and make sure `WiFi.setSleep(false)` is set. |
| Photo is saved but not attached | Verify `FILE_PHOTO_PATH` and that LittleFS is mounted before capturing. |

---

## 6. Conclusion

### 6.1 Final Thoughts

This project demonstrates how a few inexpensive components can be combined into a working home security prototype: a sensor-driven trigger, image capture, email notification and a live video stream, all running on a single microcontroller. It provided practical experience with embedded programming, networking (Wi-Fi, SMTP, HTTP) and hardware/software integration.

### 6.2 Limitations and Future Improvements

- **Secure the video stream:** the stream is currently unauthenticated and available to anyone on the local network. Adding authentication (and HTTPS or a VPN for remote access) would be the first priority.
- **Remote notifications:** add push notifications (e.g. Telegram or MQTT) as an alternative to email, which is slower and can be delayed.
- **Event logging and dashboard:** record each event (timestamp, sensor state, photo reference) in a database or time-series store and visualize it in a dashboard.
- **Non-blocking logic:** replace the blocking `delay()` calls with timers or interrupts so the device stays responsive, and use a deep-sleep/wake-on-pin approach to cut power consumption.
- **Hardware robustness:** use a dedicated non-strapping GPIO for the sensor and add a battery backup.
- **Smarter detection:** combine the contact sensor with motion detection or on-device image analysis to reduce false alarms.

---

## Repository Structure

```
esp32-cam-security-system/
├── README.md
├── LICENSE
├── .gitignore
├── src/
│   ├── security_system.ino
│   ├── camera_pins.h
│   ├── app_httpd.cpp
│   └── secrets.example.h
├── docs/
│   ├── architecture.svg
│   └── wiring-diagram.png
└── hardware/
    └── bill-of-materials.md
```

## Getting Started

1. Install the [Arduino IDE](https://www.arduino.cc/en/software) and add ESP32 board support.
2. Install the **ESP Mail Client** library (by Mobizt) from the Library Manager.
3. Copy `secrets.example.h` to `secrets.h` and fill in your Wi-Fi and Gmail credentials.
4. Wire the sensor as described in [4.1](#41-detecting-the-door-opening).
5. Select the **AI Thinker ESP32-CAM** board, upload the sketch and open the serial monitor at 115200 baud to get the device's IP address.

## License

Released under the MIT License. See [LICENSE](LICENSE) for details.

## Author

**Ryan** · [LinkedIn](https://www.linkedin.com/) · [GitHub](https://github.com/)
