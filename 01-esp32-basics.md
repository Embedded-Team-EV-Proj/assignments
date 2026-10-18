# Assignment 1: ESP32 basics

**Due:** Thursday 10/22 at noon
**Time:** about 1 to 2 hours
**Hardware needed:** none. Everything runs in your browser on [Wokwi](https://wokwi.com) (free, sign in with Google or GitHub).

Everyone does Parts 1 and 2. Part 3 is a bonus for anyone who wants more (especially if you already know C++).

---

## Part 1: Get to know the ESP32 (about 30 min)

Open these two documents:
- [ESP32 DevKitC user guide and pinout](https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32/esp32-devkitc/user_guide.html)
- [ESP32 datasheet (PDF)](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf)

You don't need to read them cover to cover. Skim and answer these 6 questions in a few words each:

1. What voltage do the ESP32's GPIO pins use: 3.3 V or 5 V? Why does that matter if a sensor runs on 5 V?
2. Which GPIO numbers are **input only** (they can't drive an LED)?
3. Which GPIO numbers are connected to the flash memory and should not be used?
4. The ESP32 has two groups of analog pins (ADC1 and ADC2). Which one stops working while Wi-Fi is on?
5. Name 3 peripherals the ESP32 has built in (for example: a way to make PWM, a serial port, a sensor bus).
6. What's the difference between the ESP32 and the ESP32-S3 that matters for our PS4 controller?

---

## Part 2: Wokwi hands-on (about 45 to 60 min)

Start a new project at [wokwi.com/projects/new/esp32](https://wokwi.com/projects/new/esp32). It opens an ESP32 with a code editor. Press the green ▶ button to run.

### 2a. Blink an LED
Add an **LED** and a **220 Ω resistor** (click the **+** button). Wire: GPIO 18 → resistor → LED long leg (anode). LED short leg → GND.

```cpp
const int LED_PIN = 18;

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(500);
  digitalWrite(LED_PIN, LOW);
  delay(500);
}
```
**Try:** make it blink twice as fast.

### 2b. Fade the LED with PWM
PWM turns the pin on and off very fast. The % of time it's on sets the brightness, and it's the same idea we'll use for motor speed.

```cpp
const int LED_PIN = 18;

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  for (int duty = 0; duty <= 255; duty++) {   // 0 = off, 255 = full on
    analogWrite(LED_PIN, duty);
    delay(5);
  }
  for (int duty = 255; duty >= 0; duty--) {
    analogWrite(LED_PIN, duty);
    delay(5);
  }
}
```

### 2c. Control brightness with a knob
Add a **potentiometer**. Wire: one outer leg → 3V3, other outer leg → GND, middle leg (SIG) → GPIO 34.

```cpp
const int LED_PIN = 18;
const int POT_PIN = 34;   // an ADC1 pin, so it still works with Wi-Fi on

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int raw = analogRead(POT_PIN);          // 0 to 4095
  int duty = map(raw, 0, 4095, 0, 255);   // scale to 0 to 255
  analogWrite(LED_PIN, duty);
  Serial.printf("pot=%d  duty=%d\n", raw, duty);
  delay(100);
}
```
**Try:** turn the knob while it runs and watch the Serial Monitor. Think of the knob as a throttle and the LED as a motor.

**Save** your project (Ctrl+S) and copy the link.

---

## Part 3: Bonus, a failsafe (optional, about 30 min)

Our car must stop if commands stop arriving. Build a tiny version:

- Type a number from 0 to 255 in the Serial Monitor and press Enter. The LED goes to that brightness.
- If no new number arrives for **2 seconds**, the LED turns off and the Serial Monitor prints `FAILSAFE`.

Hints: `Serial.available()`, `Serial.parseInt()`, and `millis()` to remember when the last command came in. Don't use `delay()` for the timer.

Extra challenge: accept `S 0.5` (a speed from -1 to 1, like the Smart Car group will send) and convert it to a duty value.

---

## How to turn it in

Post in the Group 4 Discord channel by **Thursday noon**:
1. Your answers to Part 1 (a short message is fine)
2. Your Wokwi link from Part 2 (and Part 3 if you did it)

Stuck? Post your question in Discord. Asking is part of the assignment.
