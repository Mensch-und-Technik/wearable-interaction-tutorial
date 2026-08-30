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
