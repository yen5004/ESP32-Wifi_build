# ESP32-Wifi_build
Used to see WiFi signal strength using the built in oled display

question:
i have a idea speak esp-32 dev board with intergratd .96 inch oled display, ch340 driver, wifi + ble mdule with mircro usb, its a esp32-wroom-32  ic2 oled sda d21/ gpio21 and scl  d22 / gpio22. how easy is it to upload firmware to it? can i do it over wifi? do i need special software? do i need a sd card in order to program it? i am familar with raspberry pi's and arduiino, and have a good knowledge of python and basic programming. i would like to delevelop a program that will allow me to select different modes of operation. one of the modes of operations will be a simple wifi status indicator, i would like to choose a static hard coded wifi signial or even muliple signals and have it report RSSI levels,   in a bar format and in some type of numeric vaule that i can test distnace and coverage. the second program would show me a list of all wifi signials around, and tell me the ssid name, signal strength, channel, rating (i.e, very strong, strong, ok, poor, very poor), if the ssid is password enbaled and what type of wifi signal is (i.e. wpa). how could i go about doing this?


Great project idea! You're starting from a strong place given your background with Raspberry Pi, Arduino, and Python. The ESP32 Dev Board with integrated OLED is very flexible, and what you’re aiming to do is 100% doable with the ESP32.

---

## ✅ **Summary of What You Have**

* ESP32-WROOM-32-based development board
* Built-in 0.96" I²C OLED (SDA = GPIO21, SCL = GPIO22)
* CH340 USB-to-Serial driver
* Micro USB connector
* WiFi + BLE capable

---

## ✅ **Answers to Your Initial Questions**

### 🔌 **How easy is it to upload firmware to it?**

**Very easy.** Since you have a CH340 USB-to-Serial chip, you can upload firmware via the **USB connection** using:

* **Arduino IDE** (simpler)
* **PlatformIO** (more powerful)
* **ESP-IDF** (official SDK, more complex but powerful)

You don’t need an SD card for flashing or development.

---

### 📡 **Can I upload firmware over Wi-Fi (OTA)?**

Yes, you can! The ESP32 supports **OTA (Over-the-Air) firmware updates**, but:

* You'll **first** need to flash your initial firmware via USB.
* After that, you can implement OTA updates via Wi-Fi.

There are libraries and examples for OTA in Arduino IDE and PlatformIO.

---

### 🛠️ **Do I need special software?**

Just:

* **Arduino IDE** with the ESP32 board support package (good for beginners and prototyping)
* Or **PlatformIO** in VS Code (recommended for larger projects)
* Drivers: You may need to install **CH340 driver** for USB serial on Windows/Mac

---

### 💾 **Do I need an SD card to program it?**

No SD card needed for firmware programming.

---

## 🧠 **Your Project: Multi-mode WiFi Scanner and Status Display**

You want two modes:

### 🟢 **Mode 1: Static Wi-Fi RSSI Display**

* Hardcoded SSID(s)
* Connects to one or more networks
* Displays:

  * RSSI as **numeric value**
  * RSSI as **bar graph**
* Optional: estimate distance/coverage

### 🔍 **Mode 2: WiFi Scanner**

* Scans for **all available networks**
* Displays:

  * SSID
  * Signal strength (numeric + qualitative rating)
  * Channel
  * Security type (WEP, WPA, WPA2, etc.)
  * Password-protected status

---

## 📚 **How to Build It – Step-by-Step**

### 1. **Set Up Arduino IDE or PlatformIO**

#### For Arduino:

* Install Arduino IDE
* Add ESP32 board support via Boards Manager (use URL: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`)
* Select board: `ESP32 Dev Module`

#### For PlatformIO:

* Install VS Code + PlatformIO
* Start a new project with `esp32dev`

---

### 2. **Install Required Libraries**

```cpp
Adafruit_GFX.h
Adafruit_SSD1306.h
WiFi.h
```

---

### 3. **Basic WiFi Scanner Example**

Here’s a basic scanner sketch (Arduino style):

```cpp
#include <WiFi.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define SCREEN_ADDRESS 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  delay(100);

  Wire.begin(21, 22); // GPIO21 = SDA, GPIO22 = SCL
  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("OLED init failed");
    for (;;);
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
}

void loop() {
  Serial.println("Scanning...");
  int n = WiFi.scanNetworks();

  display.clearDisplay();
  if (n == 0) {
    display.setCursor(0, 0);
    display.println("No networks found");
  } else {
    for (int i = 0; i < n && i < 5; ++i) {  // Display max 5 networks
      display.setCursor(0, i * 12);
      String ssid = WiFi.SSID(i);
      int rssi = WiFi.RSSI(i);
      String security = (WiFi.encryptionType(i) == WIFI_AUTH_OPEN) ? "Open" : "Secure";
      display.printf("%s %ddBm\n", ssid.c_str(), rssi);
    }
  }
  display.display();
  delay(5000);
}
```

---

### 4. **RSSI Bars and Ratings**

You can map RSSI values to bars and labels like this:

```cpp
String getSignalRating(int rssi) {
  if (rssi >= -50) return "Very Strong";
  else if (rssi >= -60) return "Strong";
  else if (rssi >= -70) return "OK";
  else if (rssi >= -80) return "Poor";
  else return "Very Poor";
}

int rssiToBars(int rssi) {
  if (rssi >= -50) return 5;
  else if (rssi >= -60) return 4;
  else if (rssi >= -70) return 3;
  else if (rssi >= -80) return 2;
  else return 1;
}
```

You can draw the bar graph with `display.fillRect()` based on this.

---

### 5. **Mode Switching**

Use a button or rotary encoder to switch between modes, or even a simple menu system on the OLED using up/down buttons.

---

### 6. **Advanced: Estimate Distance from RSSI**

The RSSI-to-distance formula is not super accurate, but this is commonly used:

```cpp
float estimateDistance(int rssi) {
  int txPower = -59; // typical value for 1m
  return pow(10.0, ((float)(txPower - rssi)) / 20.0);
}
```

---

## 🔁 Optional Features to Add Later

* OTA firmware updates
* Save scanned SSIDs to SPIFFS or onboard flash
* Store preferred networks and auto-compare
* Add BLE mode for another project feature

---

## 🚀 Final Thoughts

Given your background, this will be a great transition into embedded IoT programming. It’s a hands-on way to learn the ESP32’s capabilities and OLED rendering. Start with Arduino IDE, but consider PlatformIO once the codebase grows.

Would you like me to generate a base PlatformIO project or Arduino sketch with both modes structured and ready to expand?



------



Great — let's dig into **Mode 1**: a sketch that monitors **hard-coded Wi-Fi SSIDs** and shows **RSSI signal strength** in both numeric and bar formats. This is ideal for signal strength mapping or testing coverage in a building, for example.

---

## ✅ **Goal:**

* Hardcode one or more **SSIDs**.
* Scan available networks.
* If one of your target SSIDs is found:

  * Display its **SSID**
  * Show **RSSI (dBm)** value
  * Display **signal strength as bar graph**
  * Optionally estimate **distance**

---

## 🧱 **Project Requirements**

### 🔧 Hardware:

* ESP32 Dev Board with CH340 USB
* Built-in 0.96” OLED display (I²C)
* Micro-USB cable for flashing and power

### 📦 Libraries (Arduino IDE):

Install these via **Library Manager**:

* `WiFi` (built into ESP32 core)
* `Adafruit_GFX`
* `Adafruit_SSD1306`

### 🧰 Pins:

* OLED I²C:

  * `SDA` = GPIO 21
  * `SCL` = GPIO 22

---

## 🧪 Example Sketch: Hardcoded SSID Monitoring

```cpp
#include <WiFi.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define SCREEN_ADDRESS 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Hardcoded SSIDs to monitor
const char* targetSSIDs[] = {"HomeWiFi", "OfficeAP", "LabNode"};
const int numTargets = sizeof(targetSSIDs) / sizeof(targetSSIDs[0]);

// Setup OLED
void setupOLED() {
  Wire.begin(21, 22); // GPIO21 = SDA, GPIO22 = SCL
  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("OLED init failed");
    while (true);
  }
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
}

// Map RSSI to bar count (0–5 bars)
int rssiToBars(int rssi) {
  if (rssi >= -50) return 5;
  else if (rssi >= -60) return 4;
  else if (rssi >= -70) return 3;
  else if (rssi >= -80) return 2;
  else if (rssi >= -90) return 1;
  else return 0;
}

// Estimate distance (approximate)
float estimateDistance(int rssi) {
  int txPower = -59; // Default TX power at 1m
  return pow(10.0, ((float)(txPower - rssi)) / 20.0);
}

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.disconnect(true);
  delay(100);

  setupOLED();
  display.setCursor(0, 0);
  display.println("WiFi RSSI Monitor");
  display.display();
  delay(1000);
}

void loop() {
  Serial.println("Scanning...");
  int n = WiFi.scanNetworks(false, true); // passive scan, show hidden
  display.clearDisplay();
  display.setCursor(0, 0);

  bool foundAny = false;

  for (int i = 0; i < n; i++) {
    String ssid = WiFi.SSID(i);
    int rssi = WiFi.RSSI(i);

    for (int j = 0; j < numTargets; j++) {
      if (ssid == targetSSIDs[j]) {
        foundAny = true;

        int bars = rssiToBars(rssi);
        float distance = estimateDistance(rssi);

        display.printf("%s\n", ssid.c_str());
        display.printf("%ddBm  %.1fm\n", rssi, distance);

        // Draw bar graph (5 segments)
        int barX = 0;
        int barY = 48;
        int barWidth = 10;
        int barGap = 2;

        for (int b = 0; b < 5; b++) {
          int height = (b + 1) * 3;
          int x = barX + b * (barWidth + barGap);
          int y = barY - height;
          if (b < bars) {
            display.fillRect(x, y, barWidth, height, SSD1306_WHITE);
          } else {
            display.drawRect(x, y, barWidth, height, SSD1306_WHITE);
          }
        }

        display.display();
        break; // No need to keep checking this SSID
      }
    }
  }

  if (!foundAny) {
    display.setCursor(0, 0);
    display.println("No target SSIDs found");
    display.display();
  }

  WiFi.scanDelete(); // clear memory
  delay(5000); // wait before next scan
}
```

---

## 🧪 Example Output on OLED:

```
WiFi RSSI Monitor

HomeWiFi
-62dBm  3.1m

 ▓         
 ▓▓        
 ▓▓▓       
 ▓▓▓       
 ▓▓▓       
```

---

## 🔁 Next Steps / Improvements

* Add button or serial input to switch between this mode and full scanner
* Save RSSI data over time to SPIFFS or serial output
* Use a rotary encoder to select SSIDs from a list
* Automatically track RSSI drift over time

---

Would you like a **PlatformIO version** of this as well, or maybe a project structure with both modes selectable from a main menu on the OLED?




---------------------


The **ESP32 Dev Board with built-in 0.96" OLED** (like the ones commonly sold on AliExpress, Amazon, etc.) **usually does *not* include a built-in button or switch** for user interaction, other than the **reset (EN)** and sometimes a **boot** button. Here's what is typically included:

---

## 🔘 **Standard Buttons on This Kind of ESP32 Dev Board**

| Button         | Function                                               | Can You Use It in Code?                            |
| -------------- | ------------------------------------------------------ | -------------------------------------------------- |
| **EN / RESET** | Resets the board (hardware reset)                      | ❌ No                                               |
| **BOOT / IO0** | Used for flashing firmware (hold while pressing reset) | ✅ Yes (GPIO 0) – usable as input when not flashing |

---

### ✅ **Can BOOT Button Be Used in Your Code?**

Yes, **you *can* use the BOOT button (GPIO 0)** as a general-purpose input ***if you're not using it for flashing***. Just treat it as a regular button (with `digitalRead(0)`).

But keep in mind:

* Pressing BOOT while resetting puts the ESP32 into flash mode.
* If you're not holding it during boot, you can use it as a button just fine.

---

## 🛠️ **Best Practice for Buttons**

If you want **dedicated buttons** for switching modes, selecting SSIDs, or navigating menus:

### 🔧 Add Your Own Tactile Buttons:

* Connect buttons to unused GPIO pins (e.g. GPIO 12, 13, 14, 27, etc.).
* Use internal pull-ups with a wiring like:

```plaintext
[3.3V]----[Button]----[GPIO Input]
                      |
                    [GND]
```

In code:

```cpp
pinMode(13, INPUT_PULLUP);  // Button on GPIO13
if (digitalRead(13) == LOW) {
  // Button is pressed
}
```

---

## 🧠 Recommended GPIO Pins for Buttons (Safe to Use)

Avoid pins with special boot or flash functions unless you know what you're doing. These are generally **safe for input buttons**:

| GPIO | Notes          |
| ---- | -------------- |
| 12   | Good for input |
| 13   | Good for input |
| 14   | Good for input |
| 25   | Good for input |
| 26   | Good for input |
| 27   | Good for input |
| 33   | Good for input |
| 32   | Good for input |

---

## 💡 Summary

* ✅ Your board probably has **EN (reset)** and **BOOT (GPIO 0)** buttons.
* ✅ **BOOT can be used as an input button** when not flashing firmware.
* ❌ It does **not have built-in switches** for user control (like menu selection).
* ✅ You can easily add buttons to GPIOs for your mode-switching feature.

---

Would you like a code snippet that includes **mode switching using a button**?

