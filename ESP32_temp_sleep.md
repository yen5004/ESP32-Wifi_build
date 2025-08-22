Absolutely — running an ESP32 on **battery outdoors** means you **must use deep sleep** to conserve power. The ESP32 supports several sleep modes, and **deep sleep** is the most effective one, reducing current draw to as low as \~10–150 µA (depending on configuration).

Let’s update your **ESP32 temperature/humidity monitoring code** to:

* Read temperature and humidity
* Send the data to your server
* Enter **deep sleep** for a specified interval (e.g. 5 minutes)

This is perfect for outdoor use where power is limited.

---

### 🛠️ Updated ESP32 Deep Sleep Firmware

This example assumes you are using a **DHT22** (or DHT11) sensor.

#### ✅ Key Features:

* Connects to Wi-Fi
* Reads DHT22 temperature/humidity
* Sends JSON to the Flask server
* Goes into **deep sleep**
* Wakes after 5 minutes

---

### 📄 `main.ino`

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include "DHT.h"
#include "esp_sleep.h"

// === CONFIGURATION ===
#define DHTPIN 4           // GPIO where the DHT is connected
#define DHTTYPE DHT22      // DHT11 or DHT22
#define uS_TO_S_FACTOR 1000000ULL  // Convert seconds to microseconds
#define TIME_TO_SLEEP 300          // Sleep for 5 minutes

const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";
const char* serverURL = "http://192.168.1.x:5000/submit";  // Replace with your server's IP

#define NODE_ID "outdoor_sensor"  // Unique ID for this node

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  delay(1000);  // Give time for serial monitor to start

  dht.begin();

  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  int retries = 0;
  while (WiFi.status() != WL_CONNECTED && retries < 20) {
    delay(500);
    Serial.print(".");
    retries++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();

    if (!isnan(temp) && !isnan(hum)) {
      HTTPClient http;
      http.begin(serverURL);
      http.addHeader("Content-Type", "application/json");

      String payload = "{\"node_id\":\"" + String(NODE_ID) +
                       "\",\"temperature\":" + String(temp, 1) +
                       ",\"humidity\":" + String(hum, 1) + "}";

      int responseCode = http.POST(payload);
      Serial.println("HTTP POST code: " + String(responseCode));
      http.end();
    } else {
      Serial.println("Failed to read from DHT sensor!");
    }

    WiFi.disconnect(true);
    WiFi.mode(WIFI_OFF);  // Turn off Wi-Fi to save power
  } else {
    Serial.println("WiFi not connected");
  }

  Serial.println("Entering deep sleep for " + String(TIME_TO_SLEEP) + " seconds...");
  esp_sleep_enable_timer_wakeup(TIME_TO_SLEEP * uS_TO_S_FACTOR);
  esp_deep_sleep_start();
}

void loop() {
  // Not used in deep sleep mode
}
```

---

### 🔋 Power-Saving Considerations

| Tip                               | Details                                                    |
| --------------------------------- | ---------------------------------------------------------- |
| Use **deep sleep**                | Cuts power use dramatically (\~10–150 µA)                  |
| Turn off **Wi-Fi** after use      | `WiFi.disconnect(true); WiFi.mode(WIFI_OFF);`              |
| Remove **Serial prints**          | Disable for production to save cycles                      |
| Use **DHT22** over DHT11          | More accurate and stable for outdoor use                   |
| Prefer **Li-ion + LDO regulator** | ESP32 likes \~3.3V regulated power                         |
| Consider **solar charging**       | Use with a 18650 battery + charge controller for longevity |

---

### 🧪 Example Power Estimate

| Phase                 | Current Draw                                  |
| --------------------- | --------------------------------------------- |
| Wi-Fi + DHT active    | \~80–120 mA                                   |
| Deep Sleep            | \~10–150 µA                                   |
| Avg at 5 min interval | **\~100–200 µAh per cycle** (very efficient!) |

---

### 📝 Optional: GPIO Wakeup for Manual Triggering

You can also add a **push button** or external interrupt (e.g., motion detection) to wake it up outside the timer.

---

Let me know if you'd like a **battery + solar wiring diagram**, or help optimizing sensor warm-up time to reduce wake period power!
