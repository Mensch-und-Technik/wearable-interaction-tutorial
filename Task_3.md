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
        