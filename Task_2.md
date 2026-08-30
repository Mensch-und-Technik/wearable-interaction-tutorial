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