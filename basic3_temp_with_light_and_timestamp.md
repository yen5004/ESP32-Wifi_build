awesome—let’s add timestamps and make the LED “tap” every time we take a reading (no long delays, so the board stays responsive).

## DHT22 + timestamp + LED pulse on each reading

* Reads every \~2 seconds (the DHT22’s safe rate) using `millis()` timing.
* Prints **HH\:MM\:SS.mmm** uptime timestamps.
* Pulses the LED for 100 ms after each reading (non-blocking).

```cpp
#include "DHT.h"

// === Pins & Sensor ===
#define DHTPIN 4         // DHT22 data pin connected to GPIO 4
#define DHTTYPE DHT22    // Sensor type
#define LED_PIN 2        // Onboard LED (try 2, 5, or 22 if your board differs)
#define LED_ACTIVE_LOW 0 // set to 1 if your LED is inverted

DHT dht(DHTPIN, DHTTYPE);

// === Timing ===
const unsigned long READ_PERIOD_MS = 2000; // DHT22 max ~0.5 Hz
const unsigned long LED_PULSE_MS   = 100;  // LED tap length

unsigned long lastReadMs     = 0;
unsigned long ledPulseStart  = 0;
bool          ledPulsing     = false;

// --- LED helper ---
inline void writeLed(bool on) {
#if LED_ACTIVE_LOW
  digitalWrite(LED_PIN, on ? LOW : HIGH);
#else
  digitalWrite(LED_PIN, on ? HIGH : LOW);
#endif
}

// --- Timestamp helper: HH:MM:SS.mmm since boot ---
String uptimeStamp(unsigned long nowMs) {
  unsigned long totalSec = nowMs / 1000UL;
  unsigned long ms       = nowMs % 1000UL;
  unsigned long hours    = totalSec / 3600UL;
  unsigned long minutes  = (totalSec % 3600UL) / 60UL;
  unsigned long seconds  = totalSec % 60UL;

  char buf[20];
  snprintf(buf, sizeof(buf), "%02lu:%02lu:%02lu.%03lu",
           hours, minutes, seconds, ms);
  return String(buf);
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  writeLed(false);

  Serial.begin(115200);
  delay(300); // small settle so you usually see the banner
  Serial.println(F("\n=== ESP32 DHT22 Logger with LED pulse ==="));
  Serial.println(F("Pins: DHT on GPIO 4, LED on GPIO 2 (adjust if needed)"));
  dht.begin();
}

void loop() {
  unsigned long now = millis();

  // Take a reading every READ_PERIOD_MS
  if (now - lastReadMs >= READ_PERIOD_MS) {
    lastReadMs = now;

    float h = dht.readHumidity();
    float t = dht.readTemperature();      // °C
    float f = dht.readTemperature(true);  // °F

    // Pulse LED to mark the sampling event
    writeLed(true);
    ledPulsing = true;
    ledPulseStart = now;

    String ts = uptimeStamp(now);

    if (isnan(h) || isnan(t) || isnan(f)) {
      Serial.print("["); Serial.print(ts); Serial.println("] Failed to read from DHT sensor!");
    } else {
      Serial.print("["); Serial.print(ts); Serial.print("]  ");
      Serial.print("Humidity: "); Serial.print(h); Serial.print(" %  ");
      Serial.print("Temp: "); Serial.print(t); Serial.print(" °C / ");
      Serial.print(f); Serial.println(" °F");
    }
  }

  // End LED pulse after LED_PULSE_MS (non-blocking)
  if (ledPulsing && (now - ledPulseStart >= LED_PULSE_MS)) {
    writeLed(false);
    ledPulsing = false;
  }

  // (No blocking delay here)
}
```

### wiring (quick check)

* DHT22 module **+ → 3.3V**, **OUT → GPIO 4**, **– → GND**
* LED is onboard (default **GPIO 2**). If yours is different or inverted, change `LED_PIN` or set `LED_ACTIVE_LOW 1`.

want to log to csv next (still over Serial), or push these readings to a simple web page on your LAN?
