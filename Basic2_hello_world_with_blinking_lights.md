
Excellent! Let’s add the blinking LED next.

Most ESP32 DevKit boards include a small **onboard LED** tied to **GPIO 2** (sometimes labeled “IO2” or “LED”). We’ll write a sketch to blink it once per second.

---

## Blinking LED + Serial Output

```cpp
// ESP32 Hello World + Blink

#define LED_PIN 2   // onboard LED on most ESP32 DevKit boards

void setup() {
  Serial.begin(115200);
  delay(2000);
  Serial.println("Hello, world from ESP32 with blinking LED!");

  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Turn LED on
  digitalWrite(LED_PIN, HIGH);
  Serial.println("LED ON");
  delay(1000);

  // Turn LED off
  digitalWrite(LED_PIN, LOW);
  Serial.println("LED OFF");
  delay(1000);
}
```

---

### What you should see

* Serial Monitor:

  ```
  Hello, world from ESP32 with blinking LED!
  LED ON
  LED OFF
  LED ON
  ...
  ```
* The tiny onboard LED should blink **on/off every second**.

---

✅ If the LED doesn’t blink:

* Some ESP32 boards use **GPIO 2**, others **GPIO 5**, or **GPIO 22**.
* If nothing lights up, just change `#define LED_PIN 2` to another candidate and re-upload.
* Or check your board’s seller page/manual for the onboard LED pin number.

---

Would you like me to also show you how to extend this so you can **control the LED over Serial commands** (e.g., type “on” or “off” in Serial Monitor to control it)? That’s often a nice next step.
