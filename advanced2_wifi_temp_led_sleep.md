Love it — here’s a battery-friendly firmware that:

* Sleeps most of the time and **wakes every X minutes** (you set it) to take DHT22 readings
* While **Wi-Fi is connected:** shows a **heartbeat = 2 blinks every 2 seconds**
* When **taking a reading:** shows a **2-second beat**
* If **Wi-Fi isn’t connected:** shows **6 short flashes** every 5 seconds (then rests)
* Serves the latest reading at `http://<ip>/metrics` during a short **awake window**, then deep-sleeps again

All the tunables you asked for are grouped at the top and explained.

---

### ESP32 + DHT22 — Hourly Deep Sleep + LED Patterns + `/metrics`

> Libraries: **DHT sensor library** (Adafruit) + **Adafruit Unified Sensor**

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include "DHT.h"
#ifdef ESP32
  #include <ESPmDNS.h>
#endif

// ─────────────────────────────────────────────────────────────
// 🛠️ EDIT THESE: easy-to-find knobs (with explanations)
// ─────────────────────────────────────────────────────────────

// Wi-Fi credentials (used during the short awake window)
const char* WIFI_SSID = "YOUR_WIFI_SSID";
const char* WIFI_PASS = "YOUR_WIFI_PASSWORD";

// Device identity (for /metrics and mDNS)
const char* DEVICE_NAME = "esp32-shed";   // <- change per node

// DHT22 data pin
#define DHTPIN 4
#define DHTTYPE DHT22

// LED config
#define LED_PIN 2          // onboard LED pin (2 on most ESP32 DevKit)
#define LED_ACTIVE_LOW 0   // set to 1 if your LED is inverted

// Deep sleep interval — in MINUTES (converted under the hood)
const uint32_t SLEEP_INTERVAL_MINUTES = 60;   // wake every 60 minutes

// How long to stay awake (serve /metrics & show heartbeat) before sleeping again
const uint32_t AWAKE_WINDOW_SECONDS   = 20;   // keep it small for battery life

// Wi-Fi connect timeout
const uint32_t WIFI_CONNECT_TIMEOUT_MS = 15000;  // stop trying after 15s

// HEARTBEAT (when Wi-Fi is CONNECTED):
// "2 blinks every 2 seconds"
uint8_t  WIFI_HEARTBEAT_BLINKS      = 2;     // number of blinks in each heartbeat
uint16_t WIFI_HEARTBEAT_ON_MS       = 100;   // each blink ON time
uint16_t WIFI_HEARTBEAT_GAP_MS      = 120;   // gap between blinks in the pair
uint16_t WIFI_HEARTBEAT_PERIOD_MS   = 2000;  // total period for heartbeat cycle

// NO-WIFI PATTERN: "6 short flashes every 5 seconds with 5 second rest"
// (the 5 second rest is the remainder of the 5s period after the flashes)
uint8_t  NO_WIFI_FLASHES            = 6;     // how many short flashes
uint16_t NO_WIFI_FLASH_ON_MS        = 80;    // ON time for each short flash
uint16_t NO_WIFI_FLASH_OFF_MS       = 80;    // OFF gap between flashes
uint16_t NO_WIFI_CYCLE_PERIOD_MS    = 5000;  // whole pattern repeats every 5s

// DATA BEAT (when taking a temperature reading): "a 2 second beat"
uint16_t DATA_BEAT_MS               = 2000;  // LED on for 2 seconds during sample

// NOTE ON "temp take times": since this node deep-sleeps, it takes ONE reading
// per wake cycle. Adjust SLEEP_INTERVAL_MINUTES above to change how often readings occur.
// If you want to think in minutes (e.g., 15), just set SLEEP_INTERVAL_MINUTES = 15.

// ─────────────────────────────────────────────────────────────

DHT dht(DHTPIN, DHTTYPE);
WebServer server(80);

// Reading cache
float lastT = NAN, lastH = NAN;
String lastUptime;

// LED helpers
inline void ledWrite(bool on) {
#if LED_ACTIVE_LOW
  digitalWrite(LED_PIN, on ? LOW : HIGH);
#else
  digitalWrite(LED_PIN, on ? HIGH : LOW);
#endif
}

// Uptime stamp HH:MM:SS.mmm
String uptimeStamp(unsigned long nowMs) {
  unsigned long totalSec = nowMs / 1000UL;
  unsigned long ms       = nowMs % 1000UL;
  unsigned long h        = totalSec / 3600UL;
  unsigned long m        = (totalSec % 3600UL) / 60UL;
  unsigned long s        = totalSec % 60UL;

  char buf[24];
  snprintf(buf, sizeof(buf), "%02lu:%02lu:%02lu.%03lu", h, m, s, ms);
  return String(buf);
}

// ─────────────────────────────────────────────────────────────
//  Patterns (non-blocking, driven by millis())
// ─────────────────────────────────────────────────────────────
enum Pattern { PATTERN_NONE, PATTERN_WIFI, PATTERN_NO_WIFI, PATTERN_DATA_BEAT };
Pattern currentPattern = PATTERN_NONE;

struct {
  // For WIFI heartbeat
  uint8_t  blinkIndex = 0;
  bool     inBlink    = false;
  unsigned long cycleStart = 0;
  unsigned long stateStart = 0;

  // For NO-WIFI flashes
  uint8_t  flashIndex = 0;
} pat;

void setPattern(Pattern p) {
  currentPattern = p;
  pat.blinkIndex = 0;
  pat.inBlink = false;
  pat.flashIndex = 0;
  pat.cycleStart = millis();
  pat.stateStart = pat.cycleStart;
  ledWrite(false);
}

void tickPattern() {
  unsigned long now = millis();
  switch (currentPattern) {
    case PATTERN_WIFI: {
      // 2 blinks every 2s (configurable)
      unsigned long elapsed = now - pat.cycleStart;
      if (elapsed >= WIFI_HEARTBEAT_PERIOD_MS) {
        // start new cycle
        pat.cycleStart = now;
        pat.stateStart = now;
        pat.blinkIndex = 0;
        pat.inBlink = false;
        ledWrite(false);
      }

      // within the cycle, perform N blinks then stay off
      if (pat.blinkIndex < WIFI_HEARTBEAT_BLINKS) {
        if (!pat.inBlink) {
          // Start a new blink
          pat.inBlink = true;
          pat.stateStart = now;
          ledWrite(true);
        } else {
          if (now - pat.stateStart >= WIFI_HEARTBEAT_ON_MS) {
            // End of ON, go to gap
            ledWrite(false);
            pat.inBlink = false;
            pat.stateStart = now + WIFI_HEARTBEAT_GAP_MS; // target end-of-gap marker
            // advance time by waiting logic: we’ll only start next blink after gap
            if (now >= pat.stateStart) {
              // gap already passed; start next blink immediately
              pat.blinkIndex++;
            }
          }
        }
        // Handle the gap timing
        if (!pat.inBlink && (pat.blinkIndex < WIFI_HEARTBEAT_BLINKS)) {
          if (now >= pat.stateStart) {
            // start next blink
            pat.blinkIndex++;
            pat.inBlink = true;
            pat.stateStart = now;
            ledWrite(true);
          }
        }
      }
    } break;

    case PATTERN_NO_WIFI: {
      // 6 short flashes, then rest until the 5s cycle ends
      unsigned long elapsed = now - pat.cycleStart;
      unsigned long flashesTotalTime =
        NO_WIFI_FLASHES * (NO_WIFI_FLASH_ON_MS + NO_WIFI_FLASH_OFF_MS);

      if (elapsed >= NO_WIFI_CYCLE_PERIOD_MS) {
        // new cycle
        pat.cycleStart = now;
        pat.stateStart = now;
        pat.flashIndex = 0;
        ledWrite(false);
      } else if (elapsed < flashesTotalTime) {
        // We're in the flashing section
        unsigned long perFlash = NO_WIFI_FLASH_ON_MS + NO_WIFI_FLASH_OFF_MS;
        uint8_t idx = elapsed / perFlash;
        unsigned long pos = elapsed % perFlash;
        if (idx != pat.flashIndex) {
          pat.flashIndex = idx;
        }
        ledWrite(pos < NO_WIFI_FLASH_ON_MS); // ON for ON_MS, then OFF
      } else {
        // Rest section
        ledWrite(false);
      }
    } break;

    case PATTERN_DATA_BEAT: {
      // Solid ON for DATA_BEAT_MS, then caller will switch pattern back
      // (We do not auto-switch here; caller manages timing.)
    } break;

    case PATTERN_NONE:
    default:
      ledWrite(false);
      break;
  }
}

// ─────────────────────────────────────────────────────────────
//  Web server (/metrics) for the brief awake window
// ─────────────────────────────────────────────────────────────
String jsonEscape(const String& s) {
  String out; out.reserve(s.length()+4);
  for (size_t i=0;i<s.length();++i){
    char c=s[i];
    if(c=='"'||c=='\\') { out+='\\'; out+=c; }
    else if (c=='\n') out+="\\n";
    else out+=c;
  }
  return out;
}

void handleMetrics() {
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.sendHeader("Access-Control-Allow-Headers", "*");
  server.sendHeader("Access-Control-Allow-Methods", "GET,OPTIONS");

  String body = "{";
  body += "\"device\":\"" + jsonEscape(DEVICE_NAME) + "\",";
  body += "\"ip\":\"" + WiFi.localIP().toString() + "\",";
  body += "\"uptime\":\"" + lastUptime + "\",";
  body += "\"rssi\":" + String(WiFi.isConnected() ? WiFi.RSSI() : 0) + ",";
  if (isnan(lastT) || isnan(lastH)) {
    body += "\"ok\":false";
  } else {
    body += "\"ok\":true,";
    body += "\"temperature_c\":" + String(lastT, 2) + ",";
    body += "\"temperature_f\":" + String(lastT * 9.0/5.0 + 32.0, 2) + ",";
    body += "\"humidity_pct\":" + String(lastH, 2);
  }
  body += "}";
  server.send(200, "application/json", body);
}

// ─────────────────────────────────────────────────────────────

void connectWiFiWithPatterns() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  unsigned long start = millis();
  setPattern(PATTERN_NO_WIFI);  // show "no wifi" pattern while trying

  while (WiFi.status() != WL_CONNECTED && (millis() - start) < WIFI_CONNECT_TIMEOUT_MS) {
    server.handleClient(); // harmless before begin(), keeps loop snappy
    tickPattern();
    delay(10); // yield
  }

  if (WiFi.status() == WL_CONNECTED) {
    setPattern(PATTERN_WIFI);
  } else {
    // stay with NO_WIFI pattern; we’ll still take a reading and then sleep
    setPattern(PATTERN_NO_WIFI);
  }
}

void startServerMDNS() {
  server.on("/metrics", HTTP_OPTIONS, [](){
    server.sendHeader("Access-Control-Allow-Origin", "*");
    server.sendHeader("Access-Control-Allow-Headers", "*");
    server.sendHeader("Access-Control-Allow-Methods", "GET,OPTIONS");
    server.send(204);
  });
  server.on("/metrics", HTTP_GET, handleMetrics);
  server.onNotFound([](){ server.send(404, "text/plain", "Not found"); });
  server.begin();

#ifdef ESP32
  if (MDNS.begin(DEVICE_NAME)) {
    MDNS.addService("http", "tcp", 80);
  }
#endif
}

// Take one reading with a 2s “data beat”
void takeReadingWithBeat() {
  // Start DATA_BEAT pattern (solid ON), we’ll leave it on for DATA_BEAT_MS
  setPattern(PATTERN_DATA_BEAT);
  unsigned long beatStart = millis();
  ledWrite(true); // ensure on

  // Read DHT (takes ~250ms)
  float h = dht.readHumidity();
  float t = dht.readTemperature();

  lastT = t;
  lastH = h;
  lastUptime = uptimeStamp(millis());

  // Keep LED on until DATA_BEAT_MS elapsed, ticking patterns for responsiveness
  while (millis() - beatStart < DATA_BEAT_MS) {
    server.handleClient();
    tickPattern();
    delay(5);
  }

  // Return to Wi-Fi/No-Wi-Fi pattern
  if (WiFi.status() == WL_CONNECTED) setPattern(PATTERN_WIFI);
  else setPattern(PATTERN_NO_WIFI);
}

// ─────────────────────────────────────────────────────────────

void setup() {
  pinMode(LED_PIN, OUTPUT);
  ledWrite(false);

  // We start a short serial session (optional for battery builds)
  Serial.begin(115200);
  delay(100);
  Serial.println("\n=== ESP32 DHT22 (battery) — deep sleep node ===");

  dht.begin();

  // Connect Wi-Fi with visual patterns
  connectWiFiWithPatterns();

  // Start /metrics server (works even if Wi-Fi failed; just won’t be reachable)
  startServerMDNS();

  // Take one reading, with the 2s data beat
  takeReadingWithBeat();

  // Keep a tiny awake window so your dashboard can catch it
  unsigned long awakeStart = millis();
  while (millis() - awakeStart < (AWAKE_WINDOW_SECONDS * 1000UL)) {
    server.handleClient();
    tickPattern();
    delay(5);
  }

  // Prepare and enter deep sleep
  Serial.println("Going to deep sleep…");
  ledWrite(false);

  // Convert minutes -> microseconds for deep sleep timer
  uint64_t sleep_us = (uint64_t)SLEEP_INTERVAL_MINUTES * 60ULL * 1000000ULL;

  // Cleanly shut down Wi-Fi to save power
  if (WiFi.isConnected()) {
    WiFi.disconnect(true, true);
  }
  WiFi.mode(WIFI_OFF);
  btStop();   // ensure BT off on classic stacks

  esp_sleep_enable_timer_wakeup(sleep_us);
  esp_deep_sleep_start();
}

void loop() {
  // never reached (we deep sleep from setup())
}
```

---

### How this behaves

* **Boot/wake →** tries Wi-Fi.

  * While **trying/not connected**: 6 quick flashes, then a rest to complete a 5-second cycle, repeating.
  * When **connected**: 2 short blinks every 2 seconds (heartbeat).
* **Sampling:** LED turns on solid for **2 seconds** during the DHT22 read (your “data beat”).
* **Awake window:** keeps serving `/metrics` (and blinking heartbeat) for `AWAKE_WINDOW_SECONDS` (default 20 s), then **deep-sleeps** for `SLEEP_INTERVAL_MINUTES`.
* **Power:** Wi-Fi/BT turned off before sleep.

> Because this node sleeps, your dashboard will only show it **online during the awake window**. For always-visible history, the next evolution is to have the node **push** the reading to a central server (or MQTT/Node-RED/Home Assistant) at wake, then sleep immediately. Happy to wire that up when you’re ready.

If you want any of the blink timings or counts tweaked (longer/shorter flashes, different heartbeat period, etc.), just change the variables at the top — the pattern engine will follow your settings.
