
## 📦 Materials Required Per Group

| Component | Qty | Subsystem | Purpose |
| :--- | :---: | :--- | :--- |
| **Arduino UNO R3** | 2 | Leader & Follower | Microcontrollers |
| **USB Cables (A-to-B)** | 2 | Workstation | Programming and power |
| **Half-Size Breadboard** | 2 | Both Stations | Prototyping circuits |
| **Servo Motor (SG90/Standard)** | 1 | Follower | Remote actuator |
| **Flex Sensor / FSR** *(or Distance Sensor)* | 1 | Leader | Wearable control input |
| **5mm LED** | 1 | Leader | Proximity / target-lock feedback |
| **10 kΩ Resistor** | 1 | Leader | Voltage divider pull-down |
| **220 Ω Resistor** | 1 | Leader | LED current limiter |
| **Jumper Wires (M-M / M-F)** | 15+ | Both Stations | Wiring & I2C bus |
| **Velcro / Elastic Band** | 1 | Leader | Wearable finger/wrist mount |
| **Cardboard Box / Barrier** | 1 | Follower | Blind teleoperation challenge |

---

## 🛠️ Step-by-Step Workshop Guide

### Step 1: Set Up the Leader Controller (Input Sensor)

#### 1. Wiring (Analog Sensor: Flex / FSR)
Build a voltage divider on the Leader breadboard:
* **Sensor Leg 1:** Connect to `5V`.
* **Sensor Leg 2:** Connect to `A0`.
* **10 kΩ Resistor:** Connect between `A0` and `GND`.

#### 2. Calibration Sketch (Leader Arduino)
Upload this code to verify your sensor range and note your minimum (relaxed) and maximum (bent/pressed) values in the Serial Monitor.

Code C++

const int sensorPin = A0;

void setup() {
  Serial.begin(9600);
}

void loop() {
  int rawVal = analogRead(sensorPin);
  
  // Replace 200 and 800 with your actual observed min and max
  int angle = map(rawVal, 200, 800, 0, 180);
  angle = constrain(angle, 0, 180);

  Serial.print("Raw: ");
  Serial.print(rawVal);
  Serial.print(" -> Mapped Angle: ");
  Serial.println(angle);

  delay(50);
}

### Step 2: Set Up the Follower (Servo Actuator)

#### 1. Wiring

On the Follower Arduino:

- **Servo Signal (Orange / White / Yellow):** Pin `9`
- **Servo Power (Red):** `5V`
- **Servo Ground (Brown / Black):** `GND`

#### 2. Standalone Verification Sketch (Follower Arduino)

Confirm your servo sweeps cleanly before integrating I2C.

```cpp
include <Servo.h>

Servo followerServo;

void setup() {
  followerServo.attach(9);
}

void loop() {
  followerServo.write(0);
  delay(1000);
  followerServo.write(90);
  delay(1000);
  followerServo.write(180);
  delay(1000);
}
```

### Step 3: Interface Leader & Follower via I2C

Connect the two Arduino boards using 3 jumper wires:

|**Leader Arduino**|**Follower Arduino**|**Function**|
|---|---|---|
|**A4 (SDA)**|**A4 (SDA)**|I2C Serial Data|
|**A5 (SCL)**|**A5 (SCL)**|I2C Serial Clock|
|**GND**|**GND**|**Common Ground (Mandatory)**|

- **Communication Model:**
    
    - **Leader (Master):** Sends commanded servo angle ($0^\circ\text{–}180^\circ$) to Follower address `8`.
        
    - **Follower (Slave #8):** Reads the angle, writes it to the servo, and calculates the error distance to the hidden target.
        

### Step 4: Closed-Loop Telemetry (Pulsing Feedback LED)

#### 1. Feedback LED Wiring (Leader Board)

- **LED Anode (Long Leg):** Connect to Pin `8` via a `220 Ω` resistor.
    
- **LED Cathode (Short Leg):** Connect to `GND`.
    

#### 2. Leader Final Code (Master)

```cpp
#include <Wire.h>

const int sensorPin = A0;
const int ledPin = 8;
const int SLAVE_ADDR = 8;

// Calibration values from Step 1
int sensorMin = 200;
int sensorMax = 800;

int errorDistance = 180;
unsigned long lastBlinkTime = 0;
bool ledState = false;

void setup() {
  Wire.begin(); // Master mode
  pinMode(ledPin, OUTPUT);
}

void loop() {
  // 1. Read and map wearable input
  int rawVal = analogRead(sensorPin);
  int angle = map(rawVal, sensorMin, sensorMax, 0, 180);
  angle = constrain(angle, 0, 180);

  // 2. Transmit target angle to follower
  Wire.beginTransmission(SLAVE_ADDR);
  Wire.write((byte)angle);
  Wire.endTransmission();

  // 3. Request status telemetry from follower (1 byte)
  Wire.requestFrom(SLAVE_ADDR, 1);
  if (Wire.available()) {
    errorDistance = Wire.read();
  }

  // 4. Update pulsing feedback LED
  if (errorDistance <= 5) {
    // Target locked: Solid ON
    digitalWrite(ledPin, HIGH);
  } else {
    // Proximity pulsing: closer = faster pulse
    int blinkInterval = map(errorDistance, 6, 180, 70, 600);
    blinkInterval = constrain(blinkInterval, 70, 600);

    if (millis() - lastBlinkTime >= (unsigned long)blinkInterval) {
      lastBlinkTime = millis();
      ledState = !ledState;
      digitalWrite(ledPin, ledState ? HIGH : LOW);
    }
  }

  delay(20); // 50 Hz loop rate
}
```

#### 3. Follower Final Code (Slave)

```cpp
#include <Wire.h>
#include <Servo.h>

Servo targetServo;
const int SLAVE_ADDR = 8;
const int TARGET_ANGLE = 90; // The hidden target position

volatile byte commandedAngle = 90;
volatile byte errorDistance = 90;

void setup() {
  targetServo.attach(9);
  targetServo.write(commandedAngle);

  // Join I2C bus as Slave on address 8
  Wire.begin(SLAVE_ADDR);
  Wire.onReceive(receiveAngle);
  Wire.onRequest(sendTelemetry);
}

void loop() {
  targetServo.write(commandedAngle);
  delay(10);
}

// Receive new target angle from Leader
void receiveAngle(int numBytes) {
  if (Wire.available()) {
    commandedAngle = Wire.read();
    errorDistance = (byte)abs(commandedAngle - TARGET_ANGLE);
  }
}

// Return telemetry (distance to target) when requested by Leader
void sendTelemetry() {
  Wire.write(errorDistance);
}
```
