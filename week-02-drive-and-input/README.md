# Assignment 2: Drive track and Input track

**Due:** Wednesday 10/28 at noon
**Time:** about 2 hours, plus optional extras if you want to learn more
**Hardware needed:** none. Everything runs on [Wokwi](https://wokwi.com) again.

This time you **pick a track**. Each track is one half of the car's brain:

| Track | You build | Real job on the car |
|---|---|---|
| 🚗 **Drive** | Motor speed + direction, steering servo, the 250 ms failsafe | The drive ESP32 that moves the car |
| 🎮 **Input** | Read a joystick, send speed/steer every 50 ms, mode switch + e-stop | The input ESP32 (PS4 controller, commands, safety buttons) |

Paul and Roberto are on Drive. Everyone else, pick the one that sounds fun. We need about 4 people on each, so **Input could use more people**. If you can't decide, pick Input.

Everyone does **Part 1**. Then do **Part 2 for your track**. Part 3 (bonus) and Part 4 (learn more) are optional.

---

## Part 1: How our car is controlled (everyone, about 25 min)

Read this first. It's the plan we agreed on with the Smart Car group.

```
PS4 controller or Smart Car server
          │
          ▼
   Input ESP32  ── sends "S 0.50 T -0.20" every 50 ms ──►  Drive ESP32  ──► motor driver ──► motor
   (reads buttons,                                          (sets speed,  ──► steering servo
    e-stop, mode)                                            failsafe)
```

- **S** = speed, from **-1** (full reverse) to **1** (full forward). 0 = stopped.
- **T** = turn (steering), from **-1** (full left) to **1** (full right). 0 = straight.
- A new command arrives **every 50 ms**.
- If the drive ESP32 hears **nothing for 250 ms**, it stops the motor. That's the **failsafe**.

Skim these two pages (just the parts about wiring and how it works, skip the web server stuff):
- [ESP32 with a DC motor (speed and direction)](https://randomnerdtutorials.com/esp32-dc-motor-l298n-motor-driver-control-speed-direction/)
- [ESP32 with a servo motor](https://randomnerdtutorials.com/esp32-servo-motor-web-server-arduino-ide/) (read until "Web Server")

Answer these 7 questions in a few words each:

1. Commands arrive every 50 ms. How many commands is that per second?
2. The failsafe is 250 ms. How many commands in a row can go missing before the car stops?
3. Why can't we plug a motor straight into an ESP32 pin? (Think about voltage and current.)
4. A motor driver has two inputs, IN1 and IN2. What does the motor do if IN1 = PWM and IN2 = LOW? What if they're swapped? What if both are LOW?
5. A servo gets a pulse every 20 ms. About how long is the pulse when the servo is in the **center**?
6. Why do we use `millis()` instead of `delay()` in car code?
7. Which track did you pick, and why? (One sentence.)

---

## Part 2 🚗: Drive track (about 75 to 90 min)

Start a new project at [wokwi.com/projects/new/esp32](https://wokwi.com/projects/new/esp32).

**Tip:** save each part as its own project so you can send a link for each. (In Wokwi, use the arrow next to **Save** → **Save a copy**.)

### D1. Steering servo with a knob (about 20 min)

Add a **Servo** and a **Potentiometer**.

| Part | Pin | Connects to |
|---|---|---|
| Servo | PWM | GPIO 18 |
| Servo | V+ | 5V (may say VIN) |
| Servo | GND | GND |
| Potentiometer | outer legs | 3V3 and GND |
| Potentiometer | SIG (middle) | GPIO 34 |

Add the servo library: open the **Library Manager** tab, click **+**, search **ESP32Servo**, add it.

```cpp
#include <ESP32Servo.h>

const int SERVO_PIN = 18;
const int POT_PIN = 34;

Servo steering;

void setup() {
  Serial.begin(115200);
  steering.attach(SERVO_PIN);
}

void loop() {
  int raw = analogRead(POT_PIN);          // 0 to 4095
  int angle = map(raw, 0, 4095, 0, 180);  // 0 to 180 degrees
  steering.write(angle);
  Serial.printf("pot=%d  angle=%d\n", raw, angle);
  delay(50);
}
```

**Try:** a real car's wheels can't turn 180°. Change the `map()` so the servo only moves from **60° to 120°** (center is 90°).

### D2. Motor speed AND direction (about 30 min)

Wokwi has no motor driver, so we'll use **two LEDs as a stand-in**:
- **Green LED = forward** (this pin would go to the driver's IN1)
- **Red LED = reverse** (this pin would go to the driver's IN2)
- Brightness = speed.

Keep the potentiometer on GPIO 34. Add two LEDs, each with a 220 Ω resistor:

| Part | Wiring |
|---|---|
| Green LED | GPIO 25 → resistor → LED long leg. Short leg → GND |
| Red LED | GPIO 26 → resistor → LED long leg. Short leg → GND |

Now the knob is a throttle with the **center = stop**: left half is reverse, right half is forward.

```cpp
const int POT_PIN = 34;
const int FWD_PIN = 25;   // green LED, would be IN1
const int REV_PIN = 26;   // red LED, would be IN2

// speed: -1 = full reverse, 0 = stop, 1 = full forward
void setMotor(float speed) {
  speed = constrain(speed, -1.0, 1.0);   // never trust the input
  int duty = abs(speed) * 255;           // 0 to 255

  if (speed > 0) {
    analogWrite(FWD_PIN, duty);
    analogWrite(REV_PIN, 0);
  } else if (speed < 0) {
    // TODO: reverse. Which pin gets the duty, and which gets 0?
  } else {
    // TODO: stop. What should both pins be?
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(FWD_PIN, OUTPUT);
  pinMode(REV_PIN, OUTPUT);
}

void loop() {
  int raw = analogRead(POT_PIN);
  float speed = (raw - 2048) / 2048.0;    // turns 0..4095 into about -1..1
  if (abs(speed) < 0.05) speed = 0;       // "deadzone": near the middle = stop
  setMotor(speed);
  Serial.printf("raw=%d  speed=%.2f\n", raw, speed);
  delay(50);
}
```

**Fill in the two TODOs.** When it works: knob in the middle = both LEDs off, turn right = green gets brighter, turn left = red gets brighter.

**Think:** why do we never turn on both pins at the same time? (Ask in Discord if you're not sure.)

### D3. Take commands + failsafe (about 30 min)

Now the drive ESP32 listens for commands instead of a knob. Remove the knob. Keep the servo (GPIO 18) and both LEDs (25, 26).

You'll type commands in the Serial Monitor in the **same format the Input track sends**: `S 0.5 T -0.3`

```cpp
#include <ESP32Servo.h>

const int SERVO_PIN = 18;
const int FWD_PIN = 25;
const int REV_PIN = 26;
const unsigned long TIMEOUT_MS = 3000;   // 3 s so you can test by typing. The car uses 250.

Servo steering;
unsigned long lastCmd = 0;   // when the last command arrived
bool failsafe = true;        // start stopped

void setMotor(float speed) {
  // paste your finished setMotor() from D2 here
}

// steer: -1 = full left, 0 = straight, 1 = full right
void setSteering(float steer) {
  steer = constrain(steer, -1.0, 1.0);
  int angle = 90 + steer * 30;           // 60 to 120 degrees
  steering.write(angle);
}

void setup() {
  Serial.begin(115200);
  Serial.setTimeout(100);
  steering.attach(SERVO_PIN);            // attach the servo first
  pinMode(FWD_PIN, OUTPUT);
  pinMode(REV_PIN, OUTPUT);
  setMotor(0);
  setSteering(0);
  Serial.println("Type a command like: S 0.5 T -0.3");
}

void loop() {
  // 1. Did a command arrive?
  if (Serial.available()) {
    float speed = Serial.parseFloat();    // skips the "S", reads 0.5
    float steer = Serial.parseFloat();    // skips the "T", reads -0.3
    while (Serial.available()) Serial.read();  // throw away the Enter key
    setMotor(speed);
    setSteering(steer);
    lastCmd = millis();
    failsafe = false;
    Serial.printf("speed=%.2f  steer=%.2f\n", speed, steer);
  }

  // 2. TODO: failsafe.
  // If we're not already in failsafe AND it's been more than TIMEOUT_MS
  // since the last command: stop the motor, center the steering,
  // set failsafe = true, and print "FAILSAFE".
}
```

**Fill in the failsafe TODO.** (Hint: it's the same idea as the Assignment 1 bonus. `millis() - lastCmd` is "how long since the last command.")

**Test it:**
1. Type `S 0.5 T -0.3` → green LED half bright, servo turns left.
2. Type `S -1 T 1` → red LED full, servo turns right.
3. Wait 3 seconds → both LEDs off, servo centers, `FAILSAFE` prints.
4. Type `S 9 T 0` → it should act like `S 1`, not break. (Why?)

**Then:** change `TIMEOUT_MS` to `250`. Now it goes to failsafe almost right after every command, because you can't type 20 commands a second. That's why the Input track sends every 50 ms.

---

## Part 2 🎮: Input track (about 75 to 90 min)

Start a new project at [wokwi.com/projects/new/esp32](https://wokwi.com/projects/new/esp32).

**Tip:** save each part as its own project so you can send a link for each. (In Wokwi, use the arrow next to **Save** → **Save a copy**.)

### I1. Read a joystick (about 25 min)

Our PS4 controller has joysticks, so we'll practice with one. Add an **Analog Joystick**.

| Joystick pin | Connects to |
|---|---|
| VCC | 3V3 (not 5V: ESP32 pins are 3.3 V!) |
| GND | GND |
| VERT | GPIO 34 |
| HORZ | GPIO 35 |
| SEL | GPIO 32 |

```cpp
const int JOY_VERT = 34;   // up/down → speed
const int JOY_HORZ = 35;   // left/right → steering

void setup() {
  Serial.begin(115200);
}

void loop() {
  int vert = analogRead(JOY_VERT);   // 0 to 4095, middle is about 2048
  int horz = analogRead(JOY_HORZ);
  Serial.printf("vert=%d  horz=%d\n", vert, horz);
  delay(200);
}
```

To move the joystick in Wokwi: hover over it and **click the arrows**. (It only moves all the way, not halfway. The real PS4 stick will give in-between values.)

**Write down:** what number do you get for up, down, left, right, and center?

Now change it to print **speed and steer from -1 to 1**:

```cpp
const float DEADZONE = 0.10;

// turns 0..4095 into -1..1, with a deadzone in the middle
float readAxis(int pin) {
  int raw = analogRead(pin);
  float v = (raw - 2048) / 2048.0;
  if (abs(v) < DEADZONE) v = 0;          // sticks never rest at exactly 2048
  return constrain(v, -1.0, 1.0);
}
```

Use `readAxis(JOY_VERT)` for speed and `readAxis(JOY_HORZ)` for steer.

**Check:** up should be speed **+1**, and right should be steer **+1**. One of them comes out backwards. Find it and fix it. (Hint: put a `-` in front.) This is called **calibration**, and we'll have to do it on the real controller too.

### I2. Send a command every 50 ms (about 25 min)

The drive ESP32 expects a command **every 50 ms**. Using `delay(50)` would freeze the whole program, so we use `millis()` like a timer.

```cpp
const unsigned long SEND_MS = 50;
unsigned long lastSend = 0;

void loop() {
  if (millis() - lastSend >= SEND_MS) {
    lastSend += SEND_MS;

    float speed = readAxis(JOY_VERT);    // (with your fix from I1)
    float steer = readAxis(JOY_HORZ);

    Serial.printf("S %.2f T %.2f\n", speed, steer);
  }
  // nothing else here yet, but buttons will go here in I3
}
```

Put your `readAxis()` from I1 above `loop()`, and keep `setup()`.

**Check:** you should see lines like `S 1.00 T 0.00`. Let it run for 10 seconds and count roughly: there should be about **200 lines**. (Why 200?)

`%.2f` means "print the number with 2 decimals." Our message is plain text so we can read it while debugging.

### I3. Mode button + e-stop (about 30 min)

The car needs two buttons:
- **Mode button** (press the joystick, SEL): switches between **MANUAL** (joystick drives) and **AUTO** (Smart Car server drives). For now, AUTO just sends speed 0.
- **E-stop** (a red pushbutton): once pressed, always send speed 0 until the board is reset. It never turns back on by itself.

Add a **Pushbutton** (set its color to red if you like). One side → **GPIO 33**, the other side → **GND**.

```cpp
const int JOY_VERT = 34;
const int JOY_HORZ = 35;
const int MODE_BTN = 32;    // joystick SEL
const int ESTOP_BTN = 33;   // red button
const unsigned long SEND_MS = 50;
const float DEADZONE = 0.10;

unsigned long lastSend = 0;
bool autoMode = false;
bool estop = false;
int lastModeState = HIGH;          // HIGH = not pressed (because of the pull-up)
unsigned long lastModeChange = 0;

// paste your readAxis() from I1 here

void setup() {
  Serial.begin(115200);
  pinMode(MODE_BTN, INPUT_PULLUP);   // pressed = LOW
  pinMode(ESTOP_BTN, INPUT_PULLUP);
}

void loop() {
  // 1. Mode button: switch mode once per press
  int modeState = digitalRead(MODE_BTN);
  if (modeState != lastModeState && millis() - lastModeChange > 50) {
    lastModeChange = millis();
    lastModeState = modeState;
    if (modeState == LOW) {          // just got pressed
      autoMode = !autoMode;
    }
  }

  // 2. TODO: e-stop. If ESTOP_BTN reads LOW, set estop = true.
  //    (Never set it back to false.)

  // 3. Send a command every 50 ms
  if (millis() - lastSend >= SEND_MS) {
    lastSend += SEND_MS;

    float speed = 0;
    float steer = 0;
    // TODO: only read the joystick if we're in MANUAL and NOT e-stopped.

    const char* label = estop ? "ESTOP" : (autoMode ? "AUTO" : "MANUAL");
    Serial.printf("%s S %.2f T %.2f\n", label, speed, steer);
  }
}
```

**Fill in the two TODOs.**

**Test it:**
1. Move the joystick: `MANUAL S 1.00 T 0.00`.
2. Click the middle of the joystick: it says `AUTO` and speed stays 0. Click again: back to `MANUAL`.
3. Press the red button: every line says `ESTOP S 0.00 T 0.00`, even if you move the joystick or press mode.

**Think:** why does the mode button need the `millis() - lastModeChange > 50` check? (Look up "button bounce." Wokwi buttons bounce just like real ones.)

---

## Part 3: Bonus (optional, about 30 to 45 min)

Good for 3rd years or anyone who finished early. Do the one for your track (or both).

### 🚗 Drive bonus: smooth speed (ramping)
Jumping from 0 to full speed instantly is hard on the motor, gears, and battery. Make the speed **ramp**:
- Keep a `target` speed (from the command) and a `current` speed (what the motor is actually at).
- Every 20 ms, move `current` toward `target` by at most **0.05**. (So 0 to full takes 0.4 s.)
- **But the failsafe must stop the motor right away**, with no ramp.

Extra challenge: if the car is going forward and gets a reverse command, ramp down to 0 first, then go backward.

### 🎮 Input bonus: catch bad messages
Wires and Wi-Fi can scramble messages. Add two things to each message:
- A **counter** `N` that goes up by 1 each message, so the drive side can tell if messages are missing.
- A **checksum**: XOR all the characters of the message together and add it at the end in hex.

Example: `S 0.50 T -0.20 N 41*46`

Hints: build the message with `snprintf()` into a `char buf[40]` first, then loop over `buf` doing `sum ^= buf[i]`, then print with `Serial.printf("%s*%02X\n", buf, sum)`.

Question: if the drive ESP32 gets a message with the wrong checksum, what should it do?

Extra challenge: connect the ESP32 to Wi-Fi in Wokwi with `WiFi.begin("Wokwi-GUEST", "", 6);` and print its IP address and signal strength (`WiFi.RSSI()`). The real car will get commands over Wi-Fi.

---

## Part 4: Learn more (optional, no deadline)

For anyone who wants to go deeper. Not graded.

- **Try the other track.** Seriously, both halves have to talk to each other, so knowing both helps a lot.
- **Plug the tracks together (on paper):** copy a line from your Input I3 output (like `MANUAL S 1.00 T -1.00`) and type it into your Drive D3 project. Does it work? What about `ESTOP S 0.00 T 0.00`?
- [Arduino: Blink Without Delay](https://docs.arduino.cc/built-in-examples/digital/BlinkWithoutDelay/): the official explanation of the `millis()` timer trick.
- [Wokwi joystick reference](https://docs.wokwi.com/parts/wokwi-analog-joystick): which way each axis goes.
- [C++ functions (W3Schools)](https://www.w3schools.com/cpp/cpp_functions.asp): if `setMotor()` and `readAxis()` felt confusing.
- Look up the **TB6612FNG** and **BTS7960** motor drivers. What's the max current of each? Which would you pick for a small car motor?

---

## How to turn it in

**Send me (Morris) a DM on Discord** by **Wednesday 10/28 at noon** with:
1. Which track you did
2. Your answers to Part 1
3. Your Wokwi links (one per part: D1/D2/D3 or I1/I2/I3, plus bonus if you did it)

Stuck? Post your question in the Group 4 channel. Someone else probably has the same question, and asking is part of the assignment.
