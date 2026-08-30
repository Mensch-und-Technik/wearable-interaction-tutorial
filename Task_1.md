### Step 1: Set Up the Leader Controller (Input Sensor)

#### 1. Wiring (Analog Sensor: Flex / FSR)
Build a voltage divider on the Leader breadboard:
* **Sensor Leg 1:** Connect to `5V`.
* **Sensor Leg 2:** Connect to `A0`.
* **10 kΩ Resistor:** Connect between `A0` and `GND`.

#### 2. Calibration Sketch (Leader Arduino)
Upload this code to verify your sensor range and note your minimum (relaxed) and maximum (bent/pressed) values in the Serial Monitor.

Code C++
```cpp
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
```
