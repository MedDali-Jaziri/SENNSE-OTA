
# ESP32 Firmware Developer Guide for OTA Updates

## Overview

This guide is designed for firmware developers looking to implement OTA updates for ESP32 nodes. It covers the essential steps, including firmware upload, OTA implementation, Docker integration, and server updates. By following this guide, you’ll be able to streamline OTA updates for ESP32 nodes, allowing for easy and automatic firmware updates over a network.

## Prerequisites

- Basic understanding of ESP32 development using Arduino IDE or similar tools.
- Knowledge of Docker for building and deploying containers.
- Access to the ESP32 device.
- Internet connectivity for OTA updates.

---

## Step 1: Upload Firmware to ESP32 Client Node

Begin by uploading the initial firmware to your ESP32 device. Use the following script to set up the ESP32 for OTA updates.

1. Connect your ESP32 node to your development machine.
2. Flash the following code onto the ESP32 device using Arduino IDE:

```cpp
#include <WiFiMulti.h>
#include <WiFi.h>
#include <WiFiManager.h>
#include <HTTPClient.h>
#include <Update.h>

// THE IP ADDRESS WILL COMING FROM THE REAL SERVER
const char* firmware_url = "http://150.145.56.115:80/firmware.bin";

void setup() {
  Serial.begin(115200);
  // Create an instance of WiFiManager
  WiFiManager wm;
  // Try to connect to saved WiFi credentials
  if(!wm.autoConnect("SENNSE")) {
    Serial.println("Failed to connect and hit timeout");
    ESP.restart(); // Reset and try again, or go to deep sleep
    delay(1000);
  }
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    performOTAUpdate();
  }
}

void performOTAUpdate() {
  HTTPClient http;
  http.begin(firmware_url);
  int httpCode = http.GET();

  if (httpCode == HTTP_CODE_OK) {
    int contentLength = http.getSize();
    bool canBegin = Update.begin(contentLength);
    if (canBegin) {
      Serial.println("Begin OTA update");
      WiFiClient *client = http.getStreamPtr();
      size_t written = Update.writeStream(*client);
      if (written == contentLength) {
        Serial.println("Written : " + String(written) + " successfully");
      } else {
        Serial.println("Written only : " + String(written) + "/" + String(contentLength) + ". Retry?");
      }
      if (Update.end()) {
        Serial.println("OTA done!");
        if (Update.isFinished()) {
          Serial.println("Update successfully completed. Rebooting.");
          ESP.restart();
        } else {
          Serial.println("Update not finished? Something went wrong!");
        }
      } else {
        Serial.println("Error Occurred. Error #: " + String(Update.getError()));
      }
    } else {
      Serial.println("Not enough space to begin OTA");
    }
  } else {
    Serial.println("HTTP error code: " + String(httpCode));
  }
  http.end();
}

```

3. Compile and upload the firmware to the ESP32 node.

---

## Step 2: Implement Automatic OTA Updates

### Variable Declaration

In your firmware code, add the necessary variables for checking updates:

```cpp
unsigned long previousMillis = 0;
const long interval = 60000; // Check for updates every 1 minute
const char* firmware_url = "http://150.145.56.115:80/firmware.bin";
```

### Add OTA Update Functionality

Implement the OTA update logic:

```cpp
void performOTAUpdate() {
  HTTPClient http;
  http.begin(firmware_url); // Initiate connection to the server
  int httpCode = http.GET(); // Perform the GET request

  if (httpCode == HTTP_CODE_OK) {
    int contentLength = http.getSize(); // Get content length
    bool canBegin = Update.begin(contentLength); // Start OTA update

    if (canBegin) {
      Serial.println("Begin OTA update");
      WiFiClient *client = http.getStreamPtr();
      size_t written = Update.writeStream(*client); // Write the update

      if (written == contentLength) {
        Serial.println("Written: " + String(written) + " successfully");
      } else {
        Serial.println("Error: Written only " + String(written) + "/" + String(contentLength));
      }

      if (Update.end()) { // Finalize OTA update
        Serial.println("OTA completed");
        if (Update.isFinished()) {
          Serial.println("Restarting...");
          ESP.restart(); // Restart ESP32 after update
        } else {
          Serial.println("Update failed!");
        }
      } else {
        Serial.println("Update error: " + String(Update.getError()));
      }
    } else {
      Serial.println("Not enough space for OTA update");
    }
  } else {
    Serial.println("HTTP Error: " + String(httpCode));
  }

  http.end();
}
```

### Check for OTA Updates Periodically

Add the following check to your \`loop()\` function to verify if an OTA update is required every minute:

```cpp
unsigned long currentMillis = millis();
if (currentMillis - previousMillis >= interval) {
  previousMillis = currentMillis;
  if (WiFi.status() == WL_CONNECTED) {
    performOTAUpdate();
  }
}
```

---

## Step 3: Export the Compiled Binary

After modifying the firmware code for OTA, generate the binary firmware:

1. Go to **Sketch > Export Compiled Binary** in Arduino IDE.
2. The binary file will be saved in the **build** folder within your project directory.

---

## Step 4: Install Docker on your Local Machine (Optional)

If you are unfamiliar with Docker and you are not connecting with a server where an existing Docker Engine is running, follow the installation instructions below:

### Windows

1. Download the Docker Desktop installer from the [official website](https://www.docker.com/get-started).
2. Run `Docker Desktop Installer.exe`.
3. Follow the setup wizard, ensuring that **WSL 2** is selected if available.
4. After installation, start Docker Desktop.

### Linux (Ubuntu)

1. Update the package index:
   ```bash
   sudo apt update
   ```
2. Install Docker’s packages:
   ```bash
   sudo apt install docker.io
   ```
3. Start and enable Docker:
   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

### macOS

1. Download Docker Desktop from the [official website](https://www.docker.com/get-started).
2. Open the downloaded `.dmg` file and drag Docker into Applications.
3. Start Docker Desktop.

---

## Step 5: Build Docker Image for OTA Firmware Server

To deploy the firmware, create a Docker image with the binary firmware:

1. In the firmware project folder, create a `Dockerfile`:

```Dockerfile
# Use Nginx as the base image
FROM nginx:alpine

# Copy the firmware binary into the Nginx web directory
COPY build/esp32.esp32.esp32/ESP32_OTA.ino.bin /usr/share/nginx/html/firmware.bin
```

2. Note: Ensure you have the necessary authorization from the SENNSE Tech Team before pushing new images to Docker Hub.



3. Log in to Docker Hub:
   ```bash
   docker login
   ```

4. Build the Docker image:
   ```bash
   docker build -t meddali/sennse-ota-firmware-server:1.x .
   ```

5. Push the image to Docker Hub:
   ```bash
   docker push meddali/sennse-ota-firmware-server:1.x
   ```
Note: You need to ask SENSE TECH Team for the Docker Hub Authorization

## Step 6: Update the Server
Note: To access the server, you need to request permission from the SENNSE Tech Team. Make sure you have the necessary credentials before proceeding.
1. SSH into the server:
   ```bash
   ssh username@server_ip
   ```
2. Adding the user to the `Docker` group
    ```bash
   newgrp docker
   ```
3. Stop the old `Docker` container:
   ```bash
   docker stop esp32-ota-container
   ```

4. Pull the new `Docker` image:
   ```bash
   docker pull meddali/sennse-ota-firmware-server:1.x
   ```

5. Navigate to the Docker Compose directory:
   ```bash
   mkdir DockerCompose_OTAServer
   cd DockerCompose_OTAServer/
   ```

6. Create `docker-compose.yml` with the new image tag:
   ```yaml
   version: '3'
   services:
     esp32-ota-nginx:
       container_name: esp32-ota-container
       image: yourusername/sennse-ota-firmware-server:1.x
       ports:
         - "80:80"
   ```

7. Start the new container:
   ```bash
   docker-compose up -d
   ```

---

## Conclusion

By following the steps in this guide, you can easily set up and manage OTA updates for ESP32 nodes. This process ensures seamless and automatic updates, improving the maintainability of your IoT devices. For further assistance, contact the SENNSE Tech Team.

---
