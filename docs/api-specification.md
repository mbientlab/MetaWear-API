# MetaWear API Specification

MetaWear API 1.0.0 Specification.

## Bluetooth Low Energy

MetaSensors (MetaWears or MetaWear sensors) measure movement (accelerometer, gyroscope, magnetometer, sensor fusion), temperature, air pressure, and present this via the [**GATT**](https://bluetoothle.wiki/gatt) connection.

The **Generic Attributes (GATT)** is the name of the interface used to connect to Bluetooth LE (Low Energy) devices. The interface has one or more Bluetooth **Services**, identified by unique ids, that contain Bluetooth **Characteristics** also identified by ids.

A **GATT** **client** (such as your smartphone or laptop) scans for devices that are [advertising](https://bluetoothle.wiki/advertising) and connects to a **GATT** **server** (a chosen MetaWear device). Once connected, the **client** discovers the **Services** and **Characteristics** of the **server**. Then, it reads from, writes to, or sets up a connection to receive notifications from the **Characteristic**.

![BLE](./assets/images/image2.png)

Here's a brief breakdown:

1. **Scanning for Devices**: The GATT client (e.g., a smartphone or laptop) scans for nearby BLE devices that are advertising their presence.  
2. **Connecting to a GATT Server**: Once a device (e.g., a MetaWear device) is found, the client can connect to it. The device acts as the GATT server.  
3. **Discovering Services and Characteristics**: After connecting, the GATT client queries the server to discover the available Services and Characteristics. Services are collections of related functionalities, and Characteristics are attributes that contain specific data.  
4. **Interacting with Characteristics**:  
   1. **Reading**: The client can read data from a Characteristic.  
   2. **Writing**: The client can write data to a Characteristic.  
   3. **Notifications**: The client can set up notifications to receive updates from a Characteristic automatically when its value changes.

![BLE](./assets/images/image1.png)

The [BluetoothSIG](https://www.bluetooth.com/) provides [a set of standard **UUID**s](https://www.bluetooth.com/specifications/assigned-numbers/) for **Services** and **Characteristics** which the MetaSensors support:

| Service UUID | Characteristic UUID | Description           | MetaWear Result |
| :----------- | :------------------ | :-------------------- | :-------------- |
| `180A`       | `2A24`              | For Model Number      | 8 (MetaMotionS) |
| `180A`       | `2A19`              | For Battery Life      | 1.2             |
| `180A`       | `2A25`              | For Serial Number     | 055B9E          |
| `180A`       | `2A26`              | For Firmware Revision | 1.7.2           |
| `180A`       | `2A27`              | For Hardware Revision | 0.1             |
| `180A`       | `2A29`              | For Manufacturer Name | MbientLab Inc   |

For example, a heart rate sensor will typically use the Bluetooth adopted and standardized **Heart Rate Service** (UUID: `0x180D`) and **Heart Rate Measurement Characteristic** (UUID: `0x2A37`).

However, this is where the standards stop for the MetaWear spec.

MetaWears do not use the standard SIG **Services** and **Characteristics** because there are too many sensors on board and there are not enough **Services** and **Characteristics** available to support them all. Additionally, standardized **Characteristics** and **Services** for data such as “quaternions” do not exist.

Instead, the MetaWear advertises a unique **Service** and **Characteristics** that use a custom serial protocol to send commands and receive data.

MetaWear advertises the **Service** **UUID**: `326A9000-85CB-9195-D9DD-464CFBBAE75A`.

The MetaWear protocol uses the **Characteristic** **UUID**: `326A9001-85CB-9195-D9DD-464CFBBAE75A` to send commands to the board.

The MetaWear protocol uses the **Characteristic** **UUID**: `326A9006-85CB-9195-D9DD-464CFBBAE75A` to send responses. The responses may contain ACKs, device information, or sensor data. You must subscribe to this **Characteristic**.

| Name                            | 128-bit UUID                           | Mode          | Length |
| :------------------------------ | :------------------------------------- | :------------ | :----- |
| MetaWear **Service**            | `326A9000-85CB-9195-D9DD-464CFBBAE75A` |               |        |
| Command **Characteristic**      | `326A9001-85CB-9195-D9DD-464CFBBAE75A` | Read / Write  | 20     |
| Notification **Characteristic** | `326A9006-85CB-9195-D9DD-464CFBBAE75A` | Read / Notify | 20     |

## Modules

Sensors and peripherals on the MetaWear are referred to as **Modules**. 

The following **Modules** are available:

| Module Name        | Opcode | Description                                                  | State      |
| :----------------- | :----- | :----------------------------------------------------------- | :--------- |
| Switch             | `0x01` | Mechanical button                                            | Active     |
| LED                | `0x02` | Color LED                                                    | Active     |
| Accelerometer      | `0x03` | BMI160 or BMI270 acceleration sensor (3D accelerometer)      | Active     |
| Temperature        | `0x04` | Thermistor, external, and internal temperature sensor        | Active     |
| GPIO               | `0x05` | Analog and digital IOs                                       | Active     |
| Neo Pixel          | `0x06` |                                                              | Deprecated |
| iBeacon            | `0x07` |                                                              | Deprecated |
| Haptic             | `0x08` | Buzzer driver (for MMS+ or MMRL+)                            | Active     |
| Data Processor     | `0x09` | Internal math for sensor filter                              | Active     |
| Event              | `0x0A` | Events such as losing the Bluetooth connection               | Active     |
| Logging            | `0x0B` | Start, Stop, download or trigger sensor data logging         | Active     |
| Timer              | `0x0C` | Built-in timer (used for delays, counting or debug)          | Active     |
| Serial Passthrough | `0x0D` | I2C bus                                                      | Active     |
| ANCS               | `0x0E` |                                                              | Deprecated |
| Macro              | `0x0F` | Store commands in the memory                                 | Active     |
| GSR (Conductance)  | `0x10` |                                                              | Deprecated |
| Settings           | `0x11` | Bluetooth settings such as device name or TX power           | Active     |
| Barometer          | `0x12` | BMP280 pressure sensor                                       | Active     |
| Gyro               | `0x13` | BMI160 or BMI270 rotational (angular velocity) sensor        | Active     |
| Ambient Light      | `0x14` |                                                              | Deprecated |
| Magnetometer       | `0x15` | BMM150 magnetic field sensor                                 | Active     |
| Humidity           | `0x16` |                                                              | Deprecated |
| Color Detection    | `0x17` |                                                              | Deprecated |
| Proximity          | `0x18` |                                                              | Deprecated |
| Sensor Fusion      | `0x19` | Bosch algorithm for orientation (Quaternion or Euler angles) | Active     |
| Debug              | `0xFE` | Reset device or jump to bootloader                           | Active     |

## Serial Protocol

Commands to the MetaWear are sent via **Writes** to the **Command Characteristic**.

Responses from the MetaWear are received via **Notifications** from the **Notification Characteristic**.

### Data Encoding

All multi-byte values in the MetaWear serial protocol are encoded in **little-endian** byte order. This means the least significant byte comes first.

For example, the uint16\_t value `0x01F4` (500 in decimal) is sent as `[0xF4, 0x01]`.

The data types used in the protocol are:

| Type      | Size    | Description                                             |
| :-------- | :------ | :------------------------------------------------------ |
| uint8\_t  | 1 byte  | Unsigned 8-bit integer (0 to 255)                       |
| int8\_t   | 1 byte  | Signed 8-bit integer (-128 to 127)                      |
| uint16\_t | 2 bytes | Unsigned 16-bit integer, little-endian (0 to 65535)     |
| int16\_t  | 2 bytes | Signed 16-bit integer, little-endian (-32768 to 32767)  |
| uint32\_t | 4 bytes | Unsigned 32-bit integer, little-endian                  |
| int32\_t  | 4 bytes | Signed 32-bit integer, little-endian                    |
| float     | 4 bytes | IEEE 754 single-precision floating-point, little-endian |

Bit fields within a byte are packed from the least significant bit (LSB) first. For example, if a byte contains a 2-bit field `a` and a 3-bit field `b`, the layout is: bits 0-1 \= `a`, bits 2-4 \= `b`, bits 5-7 \= unused.

### Module Discovery

When a host connects to a MetaWear for the first time, it should discover which modules are available and their implementations by reading the **Module Info** register (`0x00`) from each module opcode.

The discovery process is:

1. Subscribe to the **Notification Characteristic** (`326A9006`)
2. For each known module opcode (`0x01` through `0x19`, and `0xFE`), send a read command: \[`opcode, 0x80`\] (reading register `0x00`)
3. The MetaWear responds with: \[`opcode, 0x80, impl_id, revision, ...`\]
4. If a module is not present on the board, the response will indicate an invalid or empty implementation

The **Implementation ID** identifies which hardware variant is present (e.g., BMI160 vs BMI270 for the accelerometer), and the **Revision** indicates the firmware revision for that module. Together they determine which registers are available and how data should be interpreted.

### Packed Data

Several sensor modules (Accelerometer, Gyroscope, Magnetometer) provide a **Packed Data** register that sends three consecutive XYZ samples in a single 18-byte notification instead of one 6-byte sample at a time.

The packed format is:

| Bytes 0-5   | Bytes 6-11  | Bytes 12-17 |
| :---------- | :---------- | :---------- |
| Sample 1    | Sample 2    | Sample 3    |
| X, Y, Z     | X, Y, Z     | X, Y, Z     |

Each sample is three int16\_t values (X, Y, Z) in little-endian order. Using packed data reduces BLE overhead by transmitting 3x the data per notification, which is important at high sample rates (200Hz+) where individual notifications cannot keep up.

### Command Format

When sending commands or receiving data, the serial protocol format is as follows:

| Byte 0        | Byte 1          | Bytes 2 \- 19 |
| :------------ | :-------------- | :------------ |
| Module Opcode | Setting Address | Value         |

The **Opcode** refers to the table in the **Module** section above.  
The **Address** and **Value** depend on which **Module** is specifically addressed. 

If the command involves reading data once, the **Setting Address** should be bitwise OR’d with `0x80`.

| Byte 0        | Byte 1 (bits 0-7) | Byte 1 (bit 8)                                    | Bytes 2 \- 19 |
| :------------ | :---------------- | :------------------------------------------------ | :------------ |
| Module Opcode | Setting Address   | 1 means a one time read0 means a write or notify | Value         |

For example, when sending the following command bytes to the MetaWear **Command Characteristic** \[`0x03, 0x0f, 0x01, 0x00`\]:

* `0x03` is the **Module** **Opcode** for the accelerometer  
* `0x0f` is the **Address** for the orientation interrupt enable register, we want to write to it  
* `0x01 0x00` is the 2 byte **Value** that the user wants to write to that register which corresponds to turning it on

The MetaWear will turn on the accelerometer and immediately start sending orientation data to the smartphone or computer.

For example, when sending the following command bytes to the MetaWear **Command Characteristic** `[0x04, 0x81, 0x00`\]:

* `0x04` is the **Module** **Opcode** for the temperature sensor  
* `0x01` is the **Address** for the temperature sensor data (`0x81` means we want to read it)  
* `0x00` is the 1 byte **Value** that a user must send to represent which temperature sensor we want to read from (internal, thermistor, external), `0x00` represents the internal sensor

The MetaWear will see the read temperature command and send back the temperature value. 

To understand the **Addresses** and **Values**, users must refer to the tables in the corresponding **Module** chapter:

| Setting Address                         | Mode                                                                                              | Wlen                                   | Rlen                                                 | Value                                                |
| :-------------------------------------- | :------------------------------------------------------------------------------------------------ | :------------------------------------- | :--------------------------------------------------- | :--------------------------------------------------- |
| The 2 byte **Address** for the register | The modes supported:<br>R = Readable register<br>W = Writable register<br>N = Notifiable register | Length of **Value** bytes when writing | Length of **Value** bytes when reading and notifying | Byte or bit level detail of what the **Value** means |

Let’s revisit the data packet sent to the **Command Characteristic** in the last example `[0x04, 0x81, 0x00`\]:

* `0x04` is the **Module** **Opcode** for the temperature sensor  
* `0x81` is the **Address** for the temperature sensor data register `0x01` OR’d with `0x80` because we want to read it  
* `0x00` is the **Value** that represents the 1 byte index where a value of `1` represents the on-die temperature sensor, `2` means the on-board thermistor, and `3` is a external temperature sensor that can be added to the GPIO pins

The temperature data received from the read above is \[`0x04, 0x81, 0x00, 0x00, 0x01`\]:

* `0x04` is the **Module** **Opcode** for the temperature sensor  
* `0x81` is the **Address** for the temperature sensor data with a read  
* `0x00 0x00 0x01` is the 3 byte **Value** where the first byte is the index described above and the next 2 bytes int16\_t in units of 0.25 C

For example, you may receive the following data: \[`0x03, 0x11, 0x07`\] where:

* `0x03` is the **Module** **Opcode** for the accelerometer  
* `0x11` is the **Address** for the orientation data register  
* `0x07` is the 1 byte **Value** that represents the `FACE_UP_LANDSCAPE_RIGHT` orientation

## 0x01 \- Switch Module

The **Module Opcode** is `0x01`.

The MMS and MMRL both have a switch on board that doesn’t have functionality out of the box. The functionality can be programmed so that it can be used in conjunction with the **Event Module** to start recording data, turn on the LED, etc.

The button has two states:

* Pressed / Down (`0x01`)  
* Unpressed / Up (`0x00`)

The button is already debounced and stays at value 1 when pressed.

| Setting      | Address | Mode | Wlen | Rlen | Value                                                                                 |
| :----------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------ |
| Module Info  | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t) |
| Switch State | `0x01`  | RN   | 1    | 1    | Byte 0: Switch State (Value \= `0x00` or `0x01`)                                      |

## 0x02 \- LED Module

The **Module Opcode** is `0x02`.

By default the RGB LED on the MetaSensors turn on when the device is charging. It is green when the device is fully charged and blinks blue when charging (when plugged in).

The LED is programmable and this default can be overwritten. The LED can be turned off using the **Stop** command, paused using the **Pause** command, and turned on using the **Play** command.

Before the LED can be turned on, the **Mode** should be set to determine how the LED will behave when turned on including color, time on/off, brightness, and so on:

* Mode `0x00`: Solid  
* Mode `0x01`: Blink  
* Mode `0x02`: Flash 

Depending on the **Mode**, you can create a pattern that determines how bright the blink mode is or how many times to flash the LED.

| Setting     | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| :---------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Module Info | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 1 (uint8\_t) Byte 2: Channel Count (uint8\_t) Byte 3: Secondary Mode Length (uint8\_t)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Play        | `0x01`  | RW   | 1    | 1    | Byte 0:  Value `0x00` \= Pause pattern Value `0x01` \= Play pattern Value `0x02` \= Autoplay                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Stop        | `0x02`  | RW   | 1    | 1    | Byte 0:  Value `0x00` \= Stop pattern Value `0x01` \= Stop and Reset Channels                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Mode        | `0x03`  | RW   | 18   | 18   | For a Write: Byte 0: Channel: 0: G, 1: R, 2: B Byte 1: Mode Mode (0x00) "Solid" (no pattern) Mode (0x01) "Blink" pattern: Byte 2: On Intensity (0-31) Byte 3: Off Intensity (0-31) Byte 4-5: Time On (ms) Byte 6-7: Time Period (ms) Byte 8-9: Time Offset (ms) Byte 10: Repeat Count (0-254, 255: Forever) Mode (0x02) "Flash" patter: Byte 2: On Intensity (0-31) Byte 3: Off Intensity (0-31) Byte 4-5: Time Rise (ms) Byte 6-7: Time On (ms) Byte 8-9: Time Fall (ms) Byte 10-11: Time Period (ms) Byte 12-13: Delayed Start Time (ms) Byte 14: Repeat Count (0-254, 255: Forever) For a Read: Input:  Byte 0: Channel Output:  Byte 0-17: Matches Write format |

For example, to stop and clear the LED, the command is \[`0x02, 0x02, 0x01`\].

* `0x02` is the **Opcode**  
* `0x03` is the **Stop Setting Register**  
* `0x00` is the **Byte 0** of the **Value**, it means Stop when the value is 0

For example, to Flash the LED, the command is \[`0x02, 0x03, 0x00, 0x02, 0x1f, 0x00, 0x00, 0x00, 0x32, 0x00, 0x00, 0x00, 0xf4, 0x01, 0x00, 0x00, 0x0a`\].

* `0x02` is the **Opcode**  
* `0x03` is the **Mode Setting Register**  
* `0x00` is the **Byte 0** of the **Value** for the Green channel  
* `0x02` is the **Byte 1** of the **Value** for the Mode where 0x02 is the Flash mode  
* `0x1f` is the **Byte 2** of the **Value** is the intensity when the led is on  
* `0x00` is the **Byte 3** of the **Value** is the intensity when the led is off (we want it all the way off)  
* `0x00 0x00` is the **Byte 4-5** of the **Value** is the time to turn on in ms (0ms)  
* `0x32 0x00` is the **Byte 6-7** of the **Value** is the time to stay on in ms (12800ms)  
* `0x00 0x00` is the **Byte 8-9** of the **Value** is the time to turn off in ms (0ms)  
* `0xf4 0x01` is the **Byte 10-11** of the **Value** is the time period (62465 ms)  
* `0x00 0x00` is the **Byte 12-13** of the **Value** is how long to delay the pattern (0 ms)  
* `0x0a` is the **Byte 14** of the **Value** is how many times the pattern should be repeated (10 times)

## 0x03 \- Accelerometer Module

The **Module Opcode** is `0x03`.

The **Accelerometer Module** allows users to get data from the 3-axis accelerometer. It is also used to set the accelerometer to sense taps, steps walked, flatness, or orientation.

An accelerometer is an electromechanical device that will measure acceleration forces. These forces may be static, like the constant force of gravity pulling at your feet, or they could be dynamic \- caused by moving or vibrating the accelerometer.

Acceleration is measured in units of gravities (g) or units of m/s2. One g unit \= 9.81 m/s2.

The accelerometer settings such as the range, the power mode, or the sampling rate are available via the various **Setting** registers.

Because the MMS and the MMRL have different accelerometers, the **Module Info Setting** determines which accelerometer is available:

* Revision **Value** 0 \= BOSCH BMI270  
* Revision **Value** 2 \= BOSCH BMI160

The **Accelerometer Module** exposes almost all of the registers of the BMI160 and the BMI270 via the **Setting** registers. Refer to their datasheets for the definition of the exposed registers such as PMU\_STATUS or ACC\_CONF:

* [BOSCH BMI160 DATASHEET](https://www.bosch-sensortec.com/bst/products/all_products/bmi160)  
* [BOSCH BMI270 DATASHEET](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi270/)

For the BMI160:

| Setting                        | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                                                                                                                                                                 |
| :----------------------------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Module Info                    | 0x00    | R    |      | 2    | Module information. **Read**: Byte 1: Module Implementation ID: `0x01` (uint8\_t) Byte 2: Module Revision: `0x02` (uint8\_t)                                                                                                                                                                                                                                                          |
| Accel Power Mode               | 0x01    | W    | 1    | 1    | Switch Accelerometer Power modes.  **Write**: To the corresponding BMI160 Register. Byte 1: *PMU\_STATUS* Value `0x00` \= Suspend Value `0x01` \= Normal Value `0x02` \= Low Power                                                                                                                                                                                                    |
| Accel Data Interrupt Enable    | 0x02    | RW   | 2    | 1    | Turn the data ready interrupt ON/OFF. **Write**:  Byte 1: Enable Bit Mask Bit 0: enable accel data ready int Byte 2: Disable Bit Mask Bit 0: disable accel data ready int **Read**:  Byte 1 \- Enable Bit Mask                                                                                                                                                                        |
| Accel Data Config              | 0x03    | RW   | 2    | 2    | Sets ODR and Bandwith. **Read/Write**: To the corresponding BMI160 Registers. Byte 1: *ACC\_CONF* Bit 0 \- 3: uint8\_t odr Bit 4 \- 6: uint8\_t bwp Bit 7: uint8\_t us Byte 2: *ACC\_RANGE* Bit 0 \- 3: uint8\_t range Bit 4 \- 7: unused                                                                                                                                             |
| Accel Data Interrupt           | 0x04    | RN   |      | 6    | Accelerometer data. **Read/Notify**:  Byte 1 \- 2: X: int16\_t Byte 3 \- 4: Y: int16\_t Byte 5 \- 6: Z: int16\_t                                                                                                                                                                                                                                                                      |
| Accel Data Interrupt Config    | 0x05    | RW   | 2    | 2    | Controls the input filtering to several of the motion state machines.  **Read/Write**: To the corresponding BMI160 Register. Byte 1 \- 2: *INT\_DATA* See the datasheet                                                                                                                                                                                                               |
| Low-G/High-G Interrupt Enable  | 0x06    | RW   | 2    | 1    | Turn the low/high-G data ready interrupt ON/OFF. **Write**:  Byte 1: Enable Bit Mask Bit 0: High-G X Bit 1: High-G Y Bit 2: High-G Z Bit 3: Low-G Byte 2: Disable Bit Mask Bit 0: High-G X Bit 1: High-G Y Bit 2: High-G Z Bit 3: Low-G **Read**: Byte 1: Enable Bit Mask                                                                                                             |
| Low-G/High-G Config            | 0x07    | RW   | 5    | 5    | Configures the low/highG mode. **Read/Write**: To the corresponding BMI160 Register. Byte 1 \- 5: *INT\_LOWHIGH* See the datasheet                                                                                                                                                                                                                                                    |
| Low-G/High-G Interrupt         | 0x08    | N    |      | 1    | Accelerometer low/high-G data. **Notify**: To the corresponding BMI160 Register: Byte 1: *INT\_STATUS* \- Notification Bitmask Bit 0: High-G Int Bit 1: Low-G Int Bit 2: High First X Bit 3: High First Y Bit 4: High First Z Bit 5: High Sign                                                                                                                                        |
| Motion Interrupt Enable        | 0x09    | RW   | 2    | 1    | Turn the motion data ready interrupt ON/OFF. **Write**:  Byte 1: Enable Bit Mask Bit 0: Any Motion X Bit 1: Any Motion Y Bit 2: Any Motion Z Bit 3: No Motion X Bit 4: No Motion Y Bit 5: No Motion Z Byte 2: Disable Bit Mask Bit 0: Any Motion X Bit 1: Any Motion Y Bit 2: Any Motion Z Bit 3: No Motion X Bit 4: No Motion Y Bit 5: No Motion Z **Read**: Byte 1: Enable Bit Mask |
| Motion Config                  | 0x0A    | RW   | 4    | 4    | Configures the motion mode. **Read/Write**: To the corresponding BMI160 Register: Byte 1 \- 4: *INT\_MOTION* See the datasheet                                                                                                                                                                                                                                                        |
| Motion Interrupt               | 0x0B    | N    |      | 1    | Accelerometer motion data. **Notify**: To the corresponding BMI160 Register: Byte 1: *INT\_STATUS* \- Notification Bitmask: Bit 0: Significant Motion Int Bit 1: Any Motion Int Bit 2: No Motion Int Bit 3: Any Motion First X Bit 4: Any Motion First Y Bit 5: Any Motion First Z Bit 6: Any Motion Sign                                                                             |
| Tap Interrupt Enable           | 0x0C    | RW   | 2    | 1    | Turn the tap ready interrupt ON/OFF. **Write**: Byte 1: Enable Bit Mask Bit 0: Double Tap Bit 1: Single Tap Byte 2: Disable Bit Mask Bit 0: Double Tap Bit 1: Single Tap **Read**: Enabled Bit Mask                                                                                                                                                                                   |
| Tap Config                     | 0x0D    | RW   | 2    |      | Configures the tap mode. **Read/Write**: To the corresponding BMI160 Register: Byte 1 \- 2: *INT\_TAP* See the datasheet                                                                                                                                                                                                                                                              |
| Tap Interrupt                  | 0x0E    | N    |      | 1    | Accelerometer tap data. **Notify**: To the corresponding BMI160 Register Byte 1: *INT\_STATUS* \- Notification Bitmask: Bit 0: Double Tap Int Bit 1: Single Tap Int Bit 2: Tap First X Bit 3: Tap First Y Bit 4: Tap First Z Bit 5: Tap Sign                                                                                                                                          |
| Orient Interrupt Enable        | 0x0F    | RW   | 2    | 1    | Turn the orientation ready interrupt ON/OFF. **Write**: Byte 1: Enable Bit Mask Bit 0: Orientation Byte 2: Disable Bit Mask Bit 0: Orientation **Read**: Byte 1: Enable Bit Mask                                                                                                                                                                                                      |
| Orient Config                  | 0x10    | RW   | 2    | 2    | Configures the orientation mode. **Read/Write**: To the corresponding BMI160 Register: Byte 1 \- 2: *INT\_ORIENT* See the datasheet                                                                                                                                                                                                                                                   |
| Orient Interrupt               | 0x11    | N    |      | 1    | Accelerometer orientation data. **Notify**: To the corresponding BMI160 Register Byte 1: *INT\_STATUS* \- Notification Bitmask: Bit 0: Orientation Int Bit 1-2: Orientation                                                                                                                                                                                                           |
| Flat Interrupt Enable          | 0x12    | RW   | 2    | 1    | Turn the flatness ready interrupt ON/OFF. **Write**: Byte 1: Enable Bit Mask Bit 0: Flat Byte 2: Disable Bit Mask Bit 0: Flat **Read**: Byte 1: Enable Bit Mask                                                                                                                                                                                                                       |
| Flat Config                    | 0x13    | RW   | 2    | 2    | Configures the flatness mode. **Read/Write**: To the corresponding BMI160 Register: Byte 1 \- 2: *INT\_FLAT* See the datasheet                                                                                                                                                                                                                                                        |
| Flat Interrupt                 | 0x14    | N    |      | 1    | Accelerometer flatness data. **Notify**: To the corresponding BMI160 Register Byte 1: *INT\_STATUS* \- Notification Bitmask: Bit 0: Flat Int Bit 1: Z-Axis Orientation (0: Face Up, 1: Face Down) Bit 2: Flat                                                                                                                                                                         |
| Data Offset                    | 0x15    | RW   | 7    | 7    | Offset compensations values for acc and gyro. **Read/Write**: To the corresponding BMI160 Register: *OFFSET* Byte 1: acc\_off\_x Byte 2: acc\_off\_y Byte 3: acc\_off\_z Byte 4: gyr\_off\_x Byte 5: gyr\_off\_y Byte 6: gyr\_off\_z Byte 7: enable and extra bits, see datasheet                                                                                                     |
| Power Mode Status              | 0x16    | R    |      | 1    | Present power state of accel/gyro functions. **Read**: To the corresponding BMI160 Register: *PMU\_STATUS* Byte 1: *PMU\_STATUS* Bit 0 \- 1: mag pmu status Bit 2 \- 3: gyro pmu status Bit 4 \- 5: acc\_pmu status Bit 6 \- 7: unused                                                                                                                                                |
| Step Detector Interrupt Enable | 0x17    | RW   | 2    | 1    | Turn the step detector ready interrupt ON/OFF. **Write**: Byte 1: Enable Bit Mask Bit 0: Step Detector Int Byte 2: Disable Bit Mask Bit 0: Step Detector Int **Read**: Byte 1: Enable Bit Mask                                                                                                                                                                                        |
| Step Detector Config           | 0x18    | RW   | 2    | 2    | Configures the step detector mode. **Read/Write**: To the corresponding BMI160 Register: Byte 1 \- 2: *STEP\_CONF* See the datasheet                                                                                                                                                                                                                                                  |
| Step Detector Status           | 0x19    | N    |      | 1    | Step data interrupt. **Notify**: To the corresponding BMI160 Register Byte 1: *INT\_STATUS* \- Notification Bitmask: Bit 0: Step Detector Int                                                                                                                                                                                                                                         |
| Step Counter Data              | 0x1a    | R    | 2    | 2    | Step count data. **Read**: Byte 1 \- 2: uint16\_t Step Count                                                                                                                                                                                                                                                                                                                          |
| Step Counter Reset             | 0x1b    | W    | 1    | 1    | Reset step counter. **Write**: Any write causes a reset                                                                                                                                                                                                                                                                                                                               |
| Data Packed Accel Data         | 0x1c    | N    | 18   | 18   | Accumulated Vector Output of Register 0x04 **Notify**: Data Value int16\_t (X, Y, Z)\[3\]: Byte 1 \- 2: X: int16\_t Byte 3 \- 4: Y: int16\_t Byte 5 \- 6: Z: int16\_t Byte 7 \- 8: X: int16\_t Byte 9 \- 10: Y: int16\_t Byte 11 \-12: Z: int16\_t Byte 13 \- 14: X: int16\_t Byte 15 \- 16: Y: int16\_t Byte 17 \- 18: Z: int16\_t                                                   |

For the BMI270:

| Setting                     | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| :-------------------------- | :------ | :--- | :--- | :--- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info                 | 0x00    | R    |      | 2    | Module information. **Read**: Byte 1: Module Implementation ID: `0x04` (uint8\_t) Byte 2: Module Revision: `0x00` (uint8\_t)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Accel Power Mode            | 0x01    | W    | 1    | 1    | Switch Accelerometer Power modes.  **Write**: To the corresponding BMI270 Register. Byte 1: *PMU\_STATUS* Value `0x00` \= Suspend Value `0x01` \= Normal Value `0x02` \= Low Power                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Accel Data Interrupt Enable | 0x02    | RW   | 2    | 1    | Turn the data ready interrupt ON/OFF. **Write**:  Byte 1: Enable Bit Mask Bit 0: enable accel data ready int Byte 2: Disable Bit Mask Bit 0: disable accel data ready int **Read**:  Byte 1 \- Enable Bit Mask                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Accel Data Config           | 0x03    | RW   | 2    | 2    | Sets ODR and Bandwith. **Read/Write**: To the corresponding BMI270 Registers. Byte 1: *ACC\_CONF* Bit 0 \- 3: uint8\_t odr Bit 4 \- 6: uint8\_t bwp Bit 7: uint8\_t filter\_perf Byte 2: *ACC\_RANGE* Bit 0 \- 1: uint8\_t range Bit 2 \- 7: unused                                                                                                                                                                                                                                                                                                                                                                                                    |
| Accel Data Interrupt        | 0x04    | RN   |      | 6    | Accelerometer data. **Read/Notify**:  Byte 1 \- 2: X: int16\_t Byte 3 \- 4: Y: int16\_t Byte 5 \- 6: Z: int16\_t                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Data Packed Accel Data      | 0x05    | N    | 18   | 18   | Accumulated Vector Output of Register 0x04 **Notify**: Data Value int16\_t (X, Y, Z)\[3\]: Byte 1 \- 2: X: int16\_t Byte 3 \- 4: Y: int16\_t Byte 5 \- 6: Z: int16\_t Byte 7 \- 8: X: int16\_t Byte 9 \- 10: Y: int16\_t Byte 11 \-12: Z: int16\_t Byte 13 \- 14: X: int16\_t Byte 15 \- 16: Y: int16\_t Byte 17 \- 18: Z: int16\_t                                                                                                                                                                                                                                                                                                                    |
| Feature Enable              | 0x06    | RW   | 2    | 1    | Enabes the different motion features of the BMI270 **Write**: Byte 1: Enable Bit Mask Byte 2: Disable Bit Mask Bit 0: Sig Motion Bit 1: Step Counter Bit 2: Activity Out Bit 3: Wrist Wakeup Bit 4: Wrist Gesture Bit 5: No Motion Bit 6: Any Motion Bit 7: Step Detector **Read**: Byte 1: Enable Bit Mask                                                                                                                                                                                                                                                                                                                                            |
| Feature Int Enable          | 0x07    | RW   | 2    | 1    | Turn the different feature interrupt ON/OFF. **Write**: Byte 1: Enable Bit Mask Byte 2: Disable Bit Mask Bit 0: Sig Motion Bit 1: Step Counter Bit 2: Activity Out Bit 3: Wrist Wakeup Bit 4: Wrist Gesture Bit 5: No Motion Bit 6: Any Motion Bit 7: Step Detector **Read**: Byte 1: Enable Bit Mask                                                                                                                                                                                                                                                                                                                                                  |
| Feature Config              | 0x08    | RW   | 17   | 1    | Feature Config Bytes **Write**: Byte 1: Config Bank Index Bit 0: Axis Remap Bit 1: Any Motion Bit 2: No Motion Bit 3: Sig Motion Bit 4: Step Counter 0 Bit 5: Step Counter 1 Bit 6: Step Counter 2 Bit 7: Step Counter 3 Bit 8: Wrist Gesture Bit 9: Wrist Wakeup Byte 2-16: Config Data (See datasheet for BMI270 registers:) Byte 2 \- 3: *GEN\_SET\_1* Byte 4 \- 5: *ANYMO\_1* Byte 6 \- 7: *NOMO\_1* Byte 8: *SIGMO*\_1 Byte 9 \- 10: *STEP\_COUNTER\_1* Byte 11 \- 12: *STEP\_COUNTER\_2* Byte 13 \- 14: *STEP\_COUNTER\_3* Byte 15 \- 16: *STEP\_COUNTER\_4* Byte 17: *WR\_GEST\_1* Byte 17: *WR\_WAKEUP\_1* **Read**: Byte 1: Config Bank Index |
| Motion Interrupt            | 0x09    | N    | 1    | 1    | Motion data. **Notify**:  Byte 1 Bit 0: Sig Motion Bit 1: No Motion Bit 2: Any Motion                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Wrist Interrupt             | 0x0a    | N    | 1    | 1    | Wrist data. **Notify**:  Byte 1 Bit 0: Wrist Wear Wakeup Bit 1: Wrist Gesture Bit 2 \- 4: Gesture Code 0 \= Unknown 1 \=  Push Arm Down 2 \=  Pivot Up 3 \=  Wrist Shake/Jiggle 4 \=  Arm Flick In 5 \=  Arm Flick Out                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Step Count Interrupt        | 0x0b    | RN   | 2    | 2    | Step Count data. **Notify/Read**: Bytes 1 \- 2: Step Count uint16\_t                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Activity Interrupt          | 0x0c    | N    | 1    | 1    | Activity data. **Notify**:  Byte 1 Bit 0: Activity Bit 1-2: Activity Code 0 \= Still 1 \= Walking 2 \= Running 3 \= Unknown                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Temp Enable                 | 0x0d    | W    | 1    |      | Temp Sensor Enable. **Write**: Byte 1: Input Values: 0 \= Off 1 \= On                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Temperature                 | 0x0e    | R    | 2    | 2    | Sensor Temperature. temp \= (1C/512)\*value+23C **Read**: Bytes 0 \- 1: int16\_t in units of (1./512 degC), offset by 23C                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Offset                      | 0x0f    | RW   | 4    | 4    | Offset compensation for accel. Corresponding to BMI270 Registers below (see datasheet). **Read/Write**: Byte 1: NV\_CONF Byte 2: *OFFSET0* Bit 0 \- 7: off acc x Byte 3: *OFFSET1* Bit 0 \- 7: off acc y Byte 4: *OFFSET2* Bit 0 \- 7: off acc z                                                                                                                                                                                                                                                                                                                                                                                                       |
| Downsampling                | 0x10    | RW   | 1    | 1    | Configure gyro/acell downsampling rates for FIFO. Corresponding to BMI270 Registers. **Read/Write**: Byte 1: *FIFO\_DOWNS* (see the datasheet) Bit 0 \- 2: gyro fifo down Bit 3: gyro fifo filt data Bit 4 \- 6: acc fifo down Bit 7: acc fifo filt data                                                                                                                                                                                                                                                                                                                                                                                               |

Linear acceleration is represented with the MblMwCartesianFloat struct and the values are in units of Gs. The x, y, and z fields contain the acceleration in that direction.

Sample data:

## 0x04 \- Temperature Module

The **Module Opcode** is `0x04`.

The **Temperature Module** allows users to read temperature data from up to three different sources on the MetaWear: an on-die sensor (inside the chip), an on-board thermistor, and an external thermistor that can be connected to the GPIO pins.

Temperature is returned as a signed 16-bit integer (int16\_t) in units of 0.125°C.

The **Temperature** register (`0x01`) requires an index byte in the **Value** field to select which sensor to read from:

* Index `0x00` \= On-die temperature sensor (nRF)
* Index `0x01` \= On-board thermistor
* Index `0x02` \= External thermistor (connected via GPIO)
* Index `0x03` \= BMP280 temperature sensor (Barometer on-chip)

The **Thermistor Mode** register (`0x05`) is used to configure the external thermistor GPIO pin, data pin, and whether the sensor is active high or active low.

| Setting          | Address | Mode | Wlen | Rlen | Value                                                                                                                                     |
| :--------------- | :------ | :--- | :--- | :--- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info      | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                     |
| Temperature      | `0x01`  | RN   |      | 2    | **Read**: Byte 0: Sensor Index (uint8\_t) **Response**: Byte 0: Sensor Index Byte 1-2: Temperature int16\_t in units of 0.125°C           |
| Mode             | `0x02`  | RW   | 8    | 8    | Byte 0-7: Temperature sensor mode configuration                                                                                           |
| Delta Temp       | `0x03`  | N    |      | 4    | Byte 0-3: Temperature delta notification data                                                                                             |
| Threshold Detect | `0x04`  | N    |      | 4    | Byte 0-3: Temperature threshold detection notification data                                                                               |
| Thermistor Mode  | `0x05`  | RW   | 3    | 3    | Byte 0: GPIO Analog Pin (uint8\_t) Byte 1: GPIO Data Pin (uint8\_t) Byte 2: Active High/Low (uint8\_t, 0 \= active low, 1 \= active high) |

For example, to read the on-die temperature, the command is \[`0x04, 0x81, 0x00`\]:

* `0x04` is the **Opcode** for the Temperature module
* `0x81` is the **Address** `0x01` OR'd with `0x80` for a read
* `0x00` is the sensor index for the on-die sensor

The response might be \[`0x04, 0x81, 0x00, 0xC8, 0x00`\]:

* `0x04` is the **Opcode**
* `0x81` is the **Address** with read bit set
* `0x00` is the sensor index
* `0xC8 0x00` is the temperature value (200 in decimal \= 200 × 0.125°C \= 25.0°C)

## 0x05 \- GPIO Module

The **Module Opcode** is `0x05`.

The **GPIO Module** allows users to interact with the General Purpose Input/Output pins on the MetaWear. Pins can be configured as digital outputs (set high or low), digital inputs (with pull-up, pull-down, or no-pull resistors), or analog inputs (read as an absolute voltage or as a supply ratio).

Pin change notifications can be configured to send a notification when a digital input pin changes state. The pin change type determines which transitions to monitor (rising edge, falling edge, or both).

| Setting                              | Address | Mode | Wlen | Rlen | Value                                                                                                                                         |
| :----------------------------------- | :------ | :--- | :--- | :--- | :-------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info                          | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                         |
| Set Digital Output                   | `0x01`  | W    | 1    |      | Byte 0: Pin Number (uint8\_t) \- Sets the pin HIGH                                                                                            |
| Clear Digital Output                 | `0x02`  | W    | 1    |      | Byte 0: Pin Number (uint8\_t) \- Sets the pin LOW                                                                                             |
| Digital In Pull Up                   | `0x03`  | W    | 1    |      | Byte 0: Pin Number (uint8\_t) \- Configures pin as digital input with pull-up resistor                                                        |
| Digital In Pull Down                 | `0x04`  | W    | 1    |      | Byte 0: Pin Number (uint8\_t) \- Configures pin as digital input with pull-down resistor                                                      |
| Digital In No Pull                   | `0x05`  | W    | 1    |      | Byte 0: Pin Number (uint8\_t) \- Configures pin as digital input with no pull resistor                                                        |
| Read Analog Input Absolute Reference | `0x06`  | R    | 1    | 3    | **Read**: Byte 0: Pin Number (uint8\_t) **Response**: Byte 0: Pin Number Byte 1-2: Analog value uint16\_t in mV                               |
| Read Analog Input Supply Ratio       | `0x07`  | R    | 1    | 3    | **Read**: Byte 0: Pin Number (uint8\_t) **Response**: Byte 0: Pin Number Byte 1-2: Analog value uint16\_t as ratio of supply voltage (0-4095) |
| Read Digital Input                   | `0x08`  | R    | 1    | 2    | **Read**: Byte 0: Pin Number (uint8\_t) **Response**: Byte 0: Pin Number Byte 1: Digital Value (0 or 1)                                       |
| Set Pin Change                       | `0x09`  | RW   | 2    | 2    | Byte 0: Pin Number (uint8\_t) Byte 1: Change Type (uint8\_t, 1 \= Rising, 2 \= Falling, 3 \= Any)                                             |
| Pin Change Notification              | `0x0A`  | N    |      | 2    | Byte 0: Pin Number (uint8\_t) Byte 1: Pin State (uint8\_t)                                                                                    |
| Pin Change Notification Enable       | `0x0B`  | W    | 2    |      | Byte 0: Pin Number (uint8\_t) Byte 1: Enable (0 \= disable, 1 \= enable)                                                                      |

## 0x08 \- Haptic Module

The **Module Opcode** is `0x08`.

The **Haptic Module** drives the buzzer motor on MetaWear boards that have one (MMS+ or MMRL+). It uses a simple pulse command to vibrate the buzzer for a specified duration at a specified duty cycle.

| Setting     | Address | Mode | Wlen | Rlen | Value                                                                                                                           |
| :---------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------ |
| Module Info | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                           |
| Pulse       | `0x01`  | W    | 4    |      | Byte 0: Duty Cycle (uint8\_t, 0-248) Byte 1-2: Pulse Width in ms (uint16\_t) Byte 3: Buzzer (uint8\_t, 0 \= Motor, 1 \= Buzzer) |

For example, to pulse the buzzer for 500ms at full duty cycle, the command is \[`0x08, 0x01, 0xF8, 0xF4, 0x01, 0x01`\]:

* `0x08` is the **Opcode** for the Haptic module
* `0x01` is the **Address** for the Pulse register
* `0xF8` is the duty cycle (248 \= max)
* `0xF4 0x01` is 500 in uint16\_t little-endian (500ms)
* `0x01` selects the buzzer mode

## 0x09 \- Data Processor Module

The **Module Opcode** is `0x09`.

The **Data Processor Module** provides on-board data processing filters and transformations. Filters can be chained together to create complex data pipelines that run entirely on the MetaWear without requiring a Bluetooth connection.

A filter is created by writing a filter configuration to the **Add** register. The filter receives input from a data source (a sensor register or another filter) and outputs processed data. Filters are identified by a filter ID returned when the filter is created.

The filter notification system allows the host to receive processed data. The **Notify Enable** register turns on notifications for a specific filter, and processed data is received via the **Notify** register.

| Setting             | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                  |
| :------------------ | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info         | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                                                  |
| Enable              | `0x01`  | RW   | 1    | 1    | Byte 0: Enable (0 \= disable, 1 \= enable global data processor)                                                                                                                                                                       |
| Add / Create Filter | `0x02`  | RW   | 18   | 18   | **Write**: Byte 0: Source Module ID Byte 1: Source Register ID Byte 2: Source Data Offset Byte 3: Source Data Length Byte 4: Filter Type ID Byte 5-17: Filter-specific configuration **Read (Response)**: Byte 0: Filter ID (uint8\_t) |
| Filter Notification | `0x03`  | N    |      | 18   | Byte 0: Filter ID (uint8\_t) Byte 1-17: Processed data (format depends on filter type)                                                                                                                                                 |
| Filter State        | `0x04`  | RW   | 18   | 18   | **Write**: Byte 0: Filter ID Byte 1-17: State data **Read**: Byte 0: Filter ID (input) Byte 1-17: Current state (output)                                                                                                               |
| Filter Param Modify | `0x05`  | W    | 18   |      | Byte 0: Filter ID (uint8\_t) Byte 1-17: Updated filter parameters                                                                                                                                                                      |
| Filter Remove       | `0x06`  | W    | 1    |      | Byte 0: Filter ID (uint8\_t) \- Removes the specified filter                                                                                                                                                                           |
| Notify Enable       | `0x07`  | W    | 1    |      | Byte 0: Filter ID (uint8\_t) Byte 1: Enable (0 \= disable, 1 \= enable notifications)                                                                                                                                                  |
| Remove All          | `0x08`  | W    |      |      | No value required \- Removes all filters                                                                                                                                                                                               |

The available filter types are:

| Filter Type ID | Name                  | Description                                        |
| :------------- | :-------------------- | :------------------------------------------------- |
| `0x01`         | Passthrough           | Passes data through, optionally with a count limit |
| `0x02`         | Counter / Accumulator | Counts events or accumulates values                |
| `0x03`         | Averaging             | Low memory recursive average / low pass filter     |
| `0x04`         | Comparator            | Compares input against a reference value           |
| `0x05`         | RMS                   | Root mean square of multi-component data           |
| `0x06`         | Time Delay            | Delays or downsamples data by time                 |
| `0x07`         | Math                  | Performs arithmetic operations on data             |
| `0x08`         | Sample Delay          | Delays data by a fixed number of samples           |
| `0x09`         | Data Packer           | Packs multiple data samples into one notification  |
| `0x0A`         | Account               | Adds timestamps to data                            |
| `0x0B`         | Threshold             | Detects when data crosses a threshold              |
| `0x0C`         | Delta                 | Detects when data changes by a specified amount    |
| `0x0D`         | Fuser                 | Combines data from multiple sources                |
| `0x0E`         | Buffer                | Stores the latest data sample for later retrieval  |

## 0x0A \- Event Module

The **Module Opcode** is `0x0A`.

The **Event Module** allows users to program automatic responses to data events on the MetaWear. When an event source fires (such as a sensor data ready interrupt, a button press, or a timer tick), the MetaWear can automatically execute one or more commands without host intervention.

Events are created by recording a command sequence: the host begins recording, sends the commands that should be executed when the event fires, and then ends recording. The MetaWear stores these commands and replays them each time the event triggers.

| Setting        | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                           |
| :------------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Module Info    | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                           |
| Entry          | `0x02`  | RW   | 18   | 18   | **Write**: Byte 0: Source Module ID Byte 1: Source Register ID Byte 2: Source Data Offset Byte 3-17: Event entry configuration **Read (Response)**: Byte 0: Event ID (uint8\_t) |
| Cmd Parameters | `0x03`  | W    | 18   |      | Byte 0: Event ID (uint8\_t) Byte 1-17: Command bytes to execute when event fires                                                                                                |
| Remove         | `0x04`  | W    | 1    |      | Byte 0: Event ID (uint8\_t) \- Removes the specified event                                                                                                                      |
| Remove All     | `0x05`  | W    |      |      | No value required \- Removes all events                                                                                                                                         |

## 0x0B \- Logging Module

The **Module Opcode** is `0x0B`.

The **Logging Module** allows users to record sensor data directly to the on-board flash memory of the MetaWear. This is useful when the host device (phone or computer) is not connected or when continuous streaming is not practical.

Logging works by adding triggers that specify which data sources to log. Once logging is enabled, data from those sources is written to flash with timestamps. The logged data can later be downloaded via the **Readout** register.

The MetaWear has limited flash storage. When the log is full, behavior depends on the **Circular Buffer Mode**: if enabled, the oldest entries are overwritten; if disabled, logging stops.

| Setting              | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                   |
| :------------------- | :------ | :--- | :--- | :--- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info          | `0x00`  | R    |      | 5    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t) Byte 2: Max Log Triggers (uint8\_t) Byte 3-4: Max Log Time (uint16\_t)            |
| Enable               | `0x01`  | RW   | 1    | 1    | Byte 0: Enable (0 \= disable, 1 \= enable logging)                                                                                                                      |
| Add Trigger          | `0x02`  | RW   | 4    | 4    | **Write**: Byte 0: Source Module ID Byte 1: Source Register ID Byte 2: Source Data Offset Byte 3: Source Data Length **Read (Response)**: Byte 0: Trigger ID (uint8\_t) |
| Remove Trigger       | `0x03`  | W    | 1    |      | Byte 0: Trigger ID (uint8\_t)                                                                                                                                           |
| Time                 | `0x04`  | R    |      | 4    | Byte 0-3: Current logging timestamp (uint32\_t)                                                                                                                         |
| Length               | `0x05`  | R    |      | 2    | Byte 0-1: Number of log entries (uint16\_t)                                                                                                                             |
| Readout              | `0x06`  | RW   | 4    | 4    | **Write**: Byte 0-3: Number of entries to read out (uint32\_t) **Read**: Byte 0-3: Readout status                                                                       |
| Readout Notify       | `0x07`  | N    |      | 18   | Byte 0: Trigger ID (uint8\_t) Byte 1-4: Timestamp (uint32\_t) Byte 5-17: Logged data                                                                                    |
| Readout Progress     | `0x08`  | N    |      | 2    | Byte 0-1: Entries remaining to be read (uint16\_t)                                                                                                                      |
| Drop Entries         | `0x09`  | W    | 2    |      | Byte 0-1: Number of entries to drop (uint16\_t)                                                                                                                         |
| Remove All Triggers  | `0x0A`  | W    |      |      | No value required \- Removes all log triggers                                                                                                                           |
| Circular Buffer Mode | `0x0B`  | RW   | 1    | 1    | Byte 0: Enable (0 \= stop when full, 1 \= overwrite oldest)                                                                                                             |
| Recycled Page Count  | `0x0C`  | R    |      | 2    | Byte 0-1: Number of recycled pages (uint16\_t)                                                                                                                          |
| Page Completed       | `0x0D`  | N    |      |      | Notification sent when a readout page is completed                                                                                                                      |
| Page Confirm         | `0x0E`  | W    |      |      | Confirms receipt of a readout page, triggers next page                                                                                                                  |
| Page Flush           | `0x10`  | W    |      |      | Flushes the current page to flash                                                                                                                                       |

## 0x0C \- Timer Module

The **Module Opcode** is `0x0C`.

The **Timer Module** provides a built-in timer that fires at a configurable interval. Timers are used in conjunction with the **Event Module** to periodically execute commands such as reading a sensor, toggling an LED, or streaming data at a fixed rate.

Multiple timers can be created and run simultaneously. Each timer is assigned a Timer ID when created.

| Setting       | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                 |
| :------------ | :------ | :--- | :--- | :--- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info   | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                 |
| Enable        | `0x01`  | RW   | 1    | 1    | Byte 0: Timer ID (uint8\_t) \- Enables the specified timer                                                                                                                                            |
| Timer Entry   | `0x02`  | RW   | 7    | 7    | **Write**: Byte 0-3: Period in ms (uint32\_t) Byte 4-5: Repeat Count (uint16\_t, 0xFFFF \= indefinite) Byte 6: Delay Start (uint8\_t, 0 \= no delay) **Read (Response)**: Byte 0: Timer ID (uint8\_t) |
| Start         | `0x03`  | W    | 1    |      | Byte 0: Timer ID (uint8\_t) \- Starts the timer                                                                                                                                                       |
| Stop          | `0x04`  | W    | 1    |      | Byte 0: Timer ID (uint8\_t) \- Stops the timer                                                                                                                                                        |
| Remove        | `0x05`  | W    | 1    |      | Byte 0: Timer ID (uint8\_t) \- Removes the timer                                                                                                                                                      |
| Notify        | `0x06`  | N    |      | 1    | Byte 0: Timer ID (uint8\_t) \- Notification sent when timer fires                                                                                                                                     |
| Notify Enable | `0x07`  | W    | 2    |      | Byte 0: Timer ID (uint8\_t) Byte 1: Enable (0 \= disable, 1 \= enable notifications)                                                                                                                  |

## 0x0D \- Serial Passthrough Module

The **Module Opcode** is `0x0D`.

The **Serial Passthrough Module** provides direct access to the I2C and SPI buses on the MetaWear. This allows users to communicate with external sensors and peripherals connected to the MetaWear board.

For I2C, the host specifies the device address, register address, and the data to read or write. For SPI, the host specifies the chip select pin, clock rate, and data.

| Setting        | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                                                                                        |
| :------------- | :------ | :--- | :--- | :--- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info    | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                                                                                                                        |
| I2C Read/Write | `0x01`  | RW   | 18   | 18   | **Write**: Byte 0: Device Address (uint8\_t, 7-bit I2C address) Byte 1: Register Address (uint8\_t) Byte 2: ID (uint8\_t, identifier for tracking) Byte 3: Data Length (uint8\_t) Byte 4+: Data to write **Read**: Byte 0: ID Byte 1: Device Address Byte 2: Register Address Byte 3+: Data read from device |
| SPI Read/Write | `0x02`  | RW   | 18   | 18   | **Write**: Byte 0: SPI Mode / LSB First Byte 1: CS Pin Byte 2: CLK Pin Byte 3: MOSI Pin Byte 4: MISO Pin Byte 5: Data Length Byte 6: ID Byte 7+: Data to send **Read**: Byte 0: ID Byte 1+: Data received from device                                                                                        |

## 0x0F \- Macro Module

The **Module Opcode** is `0x0F`.

The **Macro Module** allows users to store a sequence of commands in the MetaWear's non-volatile memory. These stored commands are executed on boot or on demand, allowing the MetaWear to operate autonomously without a host connection.

Macros are recorded by sending a **Begin** command, followed by the individual commands to store, and then an **End** command. The MetaWear assigns each macro a Macro ID. Macros persist across resets and power cycles.

| Setting                    | Address | Mode | Wlen | Rlen | Value                                                                                                                          |
| :------------------------- | :------ | :--- | :--- | :--- | :----------------------------------------------------------------------------------------------------------------------------- |
| Module Info                | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                          |
| Enable                     | `0x01`  | RW   | 1    | 1    | Byte 0: Macro ID \- Enables execution of a macro on boot                                                                       |
| Begin Macro                | `0x02`  | RW   | 1    | 3    | **Write**: Byte 0: Execute on boot (0 \= no, 1 \= yes) **Read (Response)**: Byte 0: Macro ID (uint8\_t)                        |
| Add Command                | `0x03`  | W    | 18   |      | Byte 0-17: Command bytes to add to the current macro (same format as any command sent to the Command Characteristic)           |
| End Macro                  | `0x04`  | W    |      |      | No value required \- Ends macro recording                                                                                      |
| Execute                    | `0x05`  | W    | 1    |      | Byte 0: Macro ID (uint8\_t) \- Executes the specified macro                                                                    |
| Finish Notification Enable | `0x06`  | W    | 2    |      | Byte 0: Macro ID (uint8\_t) Byte 1: Enable (0 \= disable, 1 \= enable)                                                         |
| Macro Finished             | `0x07`  | N    |      | 1    | Byte 0: Macro ID (uint8\_t) \- Notification sent when macro execution completes                                                |
| Erase All                  | `0x08`  | W    |      |      | No value required \- Erases all stored macros                                                                                  |
| Add Partial Command        | `0x09`  | W    | 18   |      | Byte 0-17: Partial command data for commands that exceed 18 bytes. Send partial first, then full command with **Add Command**. |

## 0x11 \- Settings Module

The **Module Opcode** is `0x11`.

The **Settings Module** is the most important module because it is used to determine which MetaWear device is connected. It is also used to control how the MetaWear advertises and how strong the Bluetooth connection is.

The **Module Info** determines the module revision:

* Revision `0x01` \= MMRL (MetaMotionRL)
* Revision `0x05` \= MMS (MetaMotionS)

The MMS has the additional capability of being able to indicate when it is charging.

The **Device Name** is “MetaWear” by default but can be changed.

Bluetooth devices typically advertise every 100ms to 1sec. The advertising only takes 1 or 2 milliseconds so that most of the time the device is not advertising. This conserves power and is one of the strategies behind Bluetooth LE's low power. The MetaWear **Advertising Interval** is configurable.

Bluetooth power is expressed in Decibel-milliwatt (dBm), the unit used to measure radio frequency (RF) power level. For a Bluetooth device, 0dBm is the standard power level and \+4dBm is just over a doubling of power. \-3dBm will be half power, \-6dBm will be a 1/4 power and \-9dBm 1/8 power. Each change in ±3dBM is a doubling/halving of power. The MetaWear **TX Power** is configurable.

**Bonding** is not currently supported.

Custom advertising packets are possible on MetaSensors using the **Partial Scan Response Packet** and **Scan Response Packet** which you can turn on and off using **Start Advertisement**.

The **Connection Parameters** for a BLE connection are a set of parameters that determine when and how the central and peripheral in a link transmit data. The parameters are exchanged when the central and peripheral initially connect. It is always the central that actually sets the connection parameters used, but the peripheral can send a Connection Parameter Update Request, that the central can then accept or reject.

The three different parameters are:

1. Connection Interval: Determines how often the central will ask for data from the peripheral. When the peripheral requests an update, it supplies a maximum and a minimum wanted interval. The connection interval must be between 7.5 ms and 4 s.
2. Slave Latency: By setting a non-zero slave latency, the peripheral can choose not to answer when the central asks for data up to the slave latency number of times. However, if the peripheral has data to send, it can choose to send data at any time. This enables a peripheral to stay sleeping for a longer time if it doesn't have data to send but still send data fast if needed.
3. Connection Supervision Timeout: This timeout determines the timeout from the last data exchange until a link is considered lost. The central will not try to reconnect before the timeout has passed, so if you have a device that goes in and out of range often and you need to notice when that happens, it might make sense to have a short timeout.

The MMS is also able to provide users with the ability to read the charge of the battery and determine if the battery is plugged in or not using the **Battery State, Charger Status** and **Power Status.** The power status determines if a USB cable is plugged into the MMS. The charger status determines if the battery finished charging or not.

| Setting               | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                         |
| :-------------------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Module Info           | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: see above (uint8\_t)                                                                                                                  |
| Device Name           | `0x01`  | RW   | 8    | 8    | Byte 0-7: Device name as ASCII characters (up to 8 bytes, null terminated)                                                                                                                                    |
| Advertising Interval  | `0x02`  | RW   | 3    | 3    | Byte 0-1: Interval in units of 0.625ms (uint16\_t) Byte 2: Timeout in seconds (uint8\_t, 0 \= no timeout)                                                                                                     |
| TX Power              | `0x03`  | RW   | 1    | 1    | Byte 0: TX Power Level in dBm (int8\_t). Valid values: \-40, \-20, \-16, \-12, \-8, \-4, 0, 4                                                                                                                 |
| Start Advertisement   | `0x05`  | W    |      |      | No value required \- Starts advertising with current settings                                                                                                                                                 |
| Scan Response Packet  | `0x07`  | RW   | 18   | 18   | Byte 0-17: Custom scan response data (up to 18 bytes)                                                                                                                                                         |
| Partial Scan Response | `0x08`  | W    | 13   |      | Byte 0-12: Partial scan response data (for packets exceeding 18 bytes)                                                                                                                                        |
| Connection Parameters | `0x09`  | RW   | 8    | 8    | Byte 0-1: Min Connection Interval in units of 1.25ms (uint16\_t) Byte 2-3: Max Connection Interval (uint16\_t) Byte 4-5: Slave Latency (uint16\_t) Byte 6-7: Supervision Timeout in units of 10ms (uint16\_t) |
| Disconnect Event      | `0x0A`  | N    |      | 1    | Byte 0: Disconnect reason code (uint8\_t) \- Notification sent on BLE disconnect                                                                                                                              |
| MAC Address           | `0x0B`  | R    |      | 6    | Byte 0-5: Bluetooth MAC Address (6 bytes, little-endian)                                                                                                                                                      |
| Battery State         | `0x0C`  | RN   |      | 3    | Byte 0: Battery charge percentage (uint8\_t, 0-100) Byte 1-2: Battery voltage in mV (uint16\_t)                                                                                                               |
| Power Status          | `0x11`  | RN   |      | 1    | Byte 0: Power Status (uint8\_t, 1 \= USB plugged in, 0 \= not plugged in)                                                                                                                                     |
| Charge Status         | `0x12`  | RN   |      | 1    | Byte 0: Charge Status (uint8\_t, 1 \= charging complete, 0 \= charging)                                                                                                                                       |
| Whitelist Filter Mode | `0x13`  | RW   | 1    | 1    | Byte 0: Filter mode (uint8\_t)                                                                                                                                                                                |
| Whitelist Addresses   | `0x14`  | RW   | 7    | 7    | Byte 0: Address entry index (uint8\_t) Byte 1: Address type (uint8\_t) Byte 2-7: BLE Address (6 bytes)                                                                                                        |
| 3V Power              | `0x1C`  | RW   | 1    | 1    | Byte 0: Enable 3.3V regulator (uint8\_t, 0 \= off, 1 \= on)                                                                                                                                                   |
| Force 1M PHY          | `0x1D`  | RW   | 1    | 1    | Byte 0: Force 1M PHY (uint8\_t, 0 \= allow 2M PHY, 1 \= force 1M PHY)                                                                                                                                         |

## 0x12 \- Barometer Module

The **Module Opcode** is `0x12`.

The **Barometer Module** provides access to the BOSCH BMP280 pressure sensor on the MetaWear. It can read barometric pressure and estimate altitude based on the pressure reading.

Pressure is returned as an unsigned 32-bit integer in units of Pa (Pascals) with a scaling factor of 256 (divide by 256 to get the actual pressure in Pa). Altitude is returned as a signed 32-bit integer in units of cm.

The barometer can be configured to run in single-shot or cyclic mode. In cyclic mode, the sensor continuously measures at the configured rate. The configuration register maps to the BMP280 registers for oversampling, filtering, and standby time.

Refer to the [BOSCH BMP280 Datasheet](https://www.bosch-sensortec.com/products/environmental-sensors/pressure-sensors/bmp280/) for detailed register configuration.

| Setting     | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                                  |
| :---------- | :------ | :--- | :--- | :--- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                                                                  |
| Pressure    | `0x01`  | RN   |      | 4    | Byte 0-3: Pressure uint32\_t in units of Pa/256 (divide by 256.0 to get Pa)                                                                                                                                                                            |
| Altitude    | `0x02`  | RN   |      | 4    | Byte 0-3: Altitude int32\_t in units of cm                                                                                                                                                                                                             |
| Config      | `0x03`  | RW   | 7    | 7    | Byte 0: Oversampling for pressure (uint8\_t, see BMP280 datasheet) Byte 1: Oversampling for temperature (uint8\_t) Byte 2: IIR Filter Coefficient (uint8\_t) Byte 3-4: Standby Time in ms (uint16\_t) Byte 5: Output Data Rate / Mode Byte 6: Reserved |
| Cyclic      | `0x04`  | RW   | 2    | 1    | **Write**: Byte 0: Enable Bit Mask (Bit 0: Pressure, Bit 1: Altitude) Byte 1: Disable Bit Mask **Read**: Byte 0: Enable Bit Mask                                                                                                                       |

## 0x13 \- Gyroscope Module

The **Module Opcode** is `0x13`.

The **Gyroscope Module** provides access to the 3-axis gyroscope on the MetaWear. The gyroscope measures angular velocity (rotational speed) in degrees per second (°/s).

Because the MMS and the MMRL have different IMUs, the **Module Info** determines which gyroscope is available:

* Implementation ID `0x00` \= BOSCH BMI160
* Implementation ID `0x01` \= BOSCH BMI270

Angular velocity is returned as three signed 16-bit integers (int16\_t) for X, Y, and Z axes. The raw values must be converted using the configured range. The gyroscope supports configurable output data rate (ODR) and full-scale range.

Refer to the datasheets for detailed register configuration:

* [BOSCH BMI160 DATASHEET](https://www.bosch-sensortec.com/bst/products/all_products/bmi160)
* [BOSCH BMI270 DATASHEET](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi270/)

For the BMI160:

| Setting                    | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                         |
| :------------------------- | :------ | :--- | :--- | :--- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info                | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: `0x00` (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                                    |
| Gyro Power Mode            | `0x01`  | W    | 1    |      | Switch Gyroscope Power modes. **Write**: Byte 0: Value `0x00` \= Suspend Value `0x01` \= Normal                                                                                                                               |
| Gyro Data Interrupt Enable | `0x02`  | RW   | 2    | 1    | Turn the data ready interrupt ON/OFF. **Write**: Byte 0: Enable Bit Mask (Bit 0: enable gyro data ready int) Byte 1: Disable Bit Mask **Read**: Byte 0: Enable Bit Mask                                                       |
| Gyro Data Config           | `0x03`  | RW   | 2    | 2    | Sets ODR and Range. **Read/Write**: Byte 0: *GYR\_CONF* (BMI160) Bit 0-3: uint8\_t odr Bit 4-5: uint8\_t bwp Byte 1: *GYR\_RANGE* Bit 0-2: uint8\_t range (0 \= 2000°/s, 1 \= 1000°/s, 2 \= 500°/s, 3 \= 250°/s, 4 \= 125°/s) |
| Gyro Data Interrupt        | `0x05`  | RN   |      | 6    | Gyroscope data. **Read/Notify**: Byte 0-1: X: int16\_t Byte 2-3: Y: int16\_t Byte 4-5: Z: int16\_t                                                                                                                            |
| Packed Gyro Data           | `0x07`  | N    |      | 18   | Accumulated Vector Output of Register 0x05. **Notify**: int16\_t (X, Y, Z)\[3\]: Byte 0-1: X Byte 2-3: Y Byte 4-5: Z Byte 6-7: X Byte 8-9: Y Byte 10-11: Z Byte 12-13: X Byte 14-15: Y Byte 16-17: Z                          |

For the BMI270:

| Setting                    | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                                                |
| :------------------------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info                | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: `0x01` (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                                                                           |
| Gyro Power Mode            | `0x01`  | W    | 1    |      | Switch Gyroscope Power modes. **Write**: Byte 0: Value `0x00` \= Suspend Value `0x01` \= Normal                                                                                                                                                                      |
| Gyro Data Interrupt Enable | `0x02`  | RW   | 2    | 1    | Turn the data ready interrupt ON/OFF. **Write**: Byte 0: Enable Bit Mask (Bit 0: enable gyro data ready int) Byte 1: Disable Bit Mask **Read**: Byte 0: Enable Bit Mask                                                                                              |
| Gyro Data Config           | `0x03`  | RW   | 2    | 2    | Sets ODR and Range. **Read/Write**: Byte 0: *GYR\_CONF* (BMI270) Bit 0-3: uint8\_t odr Bit 4-5: uint8\_t bwp Bit 6: noise\_perf Bit 7: filter\_perf Byte 1: *GYR\_RANGE* Bit 0-2: uint8\_t range (0 \= 2000°/s, 1 \= 1000°/s, 2 \= 500°/s, 3 \= 250°/s, 4 \= 125°/s) |
| Gyro Data Interrupt        | `0x04`  | RN   |      | 6    | Gyroscope data. **Read/Notify**: Byte 0-1: X: int16\_t Byte 2-3: Y: int16\_t Byte 4-5: Z: int16\_t                                                                                                                                                                   |
| Packed Gyro Data           | `0x05`  | N    |      | 18   | Accumulated Vector Output of Register 0x04. **Notify**: int16\_t (X, Y, Z)\[3\]: Byte 0-1: X Byte 2-3: Y Byte 4-5: Z Byte 6-7: X Byte 8-9: Y Byte 10-11: Z Byte 12-13: X Byte 14-15: Y Byte 16-17: Z                                                                 |
| Offset                     | `0x06`  | RW   | 6    | 6    | Offset compensation for gyro. **Read/Write**: Byte 0-1: gyr\_off\_x (int16\_t) Byte 2-3: gyr\_off\_y (int16\_t) Byte 4-5: gyr\_off\_z (int16\_t)                                                                                                                     |

## 0x15 \- Magnetometer Module

The **Module Opcode** is `0x15`.

The **Magnetometer Module** provides access to the BOSCH BMM150 3-axis magnetometer on the MetaWear. The magnetometer measures the Earth's magnetic field and is commonly used for compass heading and orientation.

Magnetic field strength is returned as three signed 16-bit integers (int16\_t) for X, Y, and Z axes in units of 0.0625 µT (micro-Tesla). The magnetometer supports configurable output data rate and repetition settings that affect accuracy and power consumption.

Refer to the [BOSCH BMM150 DATASHEET](https://www.bosch-sensortec.com/products/motion-sensors/magnetometers/bmm150/) for detailed register configuration.

| Setting                   | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                |
| :------------------------ | :------ | :--- | :--- | :--- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Module Info               | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                |
| Mag Power Mode            | `0x01`  | W    | 1    |      | Switch Magnetometer Power modes. **Write**: Byte 0: Value `0x00` \= Suspend Value `0x01` \= Active / Normal                                                                                          |
| Mag Data Interrupt Enable | `0x02`  | RW   | 2    | 1    | Turn the data ready interrupt ON/OFF. **Write**: Byte 0: Enable Bit Mask (Bit 0: enable mag data ready int) Byte 1: Disable Bit Mask **Read**: Byte 0: Enable Bit Mask                               |
| Mag Data Rate             | `0x03`  | RW   | 1    | 1    | Output data rate. **Read/Write**: Byte 0: ODR value (uint8\_t, 0 \= 10Hz, 1 \= 2Hz, 2 \= 6Hz, 3 \= 8Hz, 4 \= 15Hz, 5 \= 20Hz, 6 \= 25Hz, 7 \= 30Hz)                                                  |
| Mag Data Repetitions      | `0x04`  | RW   | 2    | 2    | Repetition settings for XY and Z axes. **Read/Write**: Byte 0: XY Repetitions (uint8\_t, see BMM150 datasheet) Byte 1: Z Repetitions (uint8\_t)                                                      |
| Mag Data                  | `0x05`  | RN   |      | 6    | Magnetometer data. **Read/Notify**: Byte 0-1: X: int16\_t Byte 2-3: Y: int16\_t Byte 4-5: Z: int16\_t (in units of 0.0625 µT)                                                                        |
| Packed Mag Data           | `0x09`  | N    |      | 18   | Accumulated Vector Output of Register 0x05. **Notify**: int16\_t (X, Y, Z)\[3\]: Byte 0-1: X Byte 2-3: Y Byte 4-5: Z Byte 6-7: X Byte 8-9: Y Byte 10-11: Z Byte 12-13: X Byte 14-15: Y Byte 16-17: Z |

## 0x19 \- Sensor Fusion Module

The **Module Opcode** is `0x19`.

The **Sensor Fusion Module** uses the BOSCH sensor fusion algorithm to combine data from the accelerometer, gyroscope, and magnetometer into meaningful orientation data. It provides several output types including quaternions, Euler angles, gravity vectors, and linear acceleration (acceleration without gravity).

The sensor fusion algorithm runs entirely on the MetaWear and outputs calibrated, processed orientation data. The module must be configured with a fusion mode that determines which sensors are used and which outputs are available.

The fusion modes are:

* Mode `0x01`: NDOF (Nine Degrees of Freedom) \- Uses accelerometer, gyroscope, and magnetometer. All outputs available.
* Mode `0x02`: IMU Plus \- Uses accelerometer and gyroscope only. Quaternion, Euler, gravity, and linear acceleration outputs available.
* Mode `0x03`: Compass \- Uses accelerometer and magnetometer. Heading output available.
* Mode `0x04`: M4G \- Uses accelerometer and magnetometer with gyroscope simulation. Lower power than NDOF.

| Setting           | Address | Mode | Wlen | Rlen | Value                                                                                                                                                                                                                                 |
| :---------------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Module Info       | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t)                                                                                                                                                 |
| Enable            | `0x01`  | RW   | 1    | 1    | Byte 0: Enable (0 \= disable, 1 \= enable sensor fusion)                                                                                                                                                                              |
| Mode              | `0x02`  | RW   | 4    | 4    | Byte 0: Fusion Mode (uint8\_t, 1 \= NDOF, 2 \= IMU Plus, 3 \= Compass, 4 \= M4G) Byte 1: Acc Range (uint8\_t) Byte 2: Gyro Range (uint8\_t) Byte 3: Reserved                                                                          |
| Output Enable     | `0x03`  | RW   | 2    | 1    | **Write**: Byte 0: Enable Bit Mask Byte 1: Disable Bit Mask (Bit 0: Corrected Acc, Bit 1: Corrected Gyro, Bit 2: Corrected Mag, Bit 3: Quaternion, Bit 4: Euler, Bit 5: Gravity, Bit 6: Linear Acc) **Read**: Byte 0: Enable Bit Mask |
| Corrected Acc     | `0x04`  | N    |      | 12   | Corrected accelerometer data. **Notify**: Byte 0-3: X: float Byte 4-7: Y: float Byte 8-11: Z: float (in units of g)                                                                                                                   |
| Corrected Gyro    | `0x05`  | N    |      | 12   | Corrected gyroscope data. **Notify**: Byte 0-3: X: float Byte 4-7: Y: float Byte 8-11: Z: float (in units of °/s)                                                                                                                     |
| Corrected Mag     | `0x06`  | N    |      | 12   | Corrected magnetometer data. **Notify**: Byte 0-3: X: float Byte 4-7: Y: float Byte 8-11: Z: float (in units of µT)                                                                                                                   |
| Quaternion        | `0x07`  | N    |      | 16   | Orientation as quaternion. **Notify**: Byte 0-3: W: float Byte 4-7: X: float Byte 8-11: Y: float Byte 12-15: Z: float                                                                                                                 |
| Euler Angles      | `0x08`  | N    |      | 12   | Orientation as Euler angles. **Notify**: Byte 0-3: Heading / Yaw: float (°) Byte 4-7: Pitch: float (°) Byte 8-11: Roll: float (°)                                                                                                     |
| Gravity Vector    | `0x09`  | N    |      | 12   | Gravity direction vector. **Notify**: Byte 0-3: X: float Byte 4-7: Y: float Byte 8-11: Z: float (in units of g)                                                                                                                       |
| Linear Acc        | `0x0A`  | N    |      | 12   | Linear acceleration (without gravity). **Notify**: Byte 0-3: X: float Byte 4-7: Y: float Byte 8-11: Z: float (in units of g)                                                                                                          |
| Calibration State | `0x0B`  | R    |      | 3    | Byte 0: Accelerometer Calibration Accuracy (uint8\_t, 0 \= unreliable, 3 \= high) Byte 1: Gyroscope Calibration Accuracy (uint8\_t) Byte 2: Magnetometer Calibration Accuracy (uint8\_t)                                              |
| Acc Cal Data      | `0x0C`  | RW   | 18   | 18   | Accelerometer calibration data for save/restore across sessions                                                                                                                                                                       |
| Gyro Cal Data     | `0x0D`  | RW   | 18   | 18   | Gyroscope calibration data for save/restore across sessions                                                                                                                                                                           |
| Mag Cal Data      | `0x0E`  | RW   | 18   | 18   | Magnetometer calibration data for save/restore across sessions                                                                                                                                                                        |
| Reset Orientation | `0x0F`  | W    |      |      | No value required \- Resets the orientation to the current position                                                                                                                                                                   |

### Fusion Mode Details

Each fusion mode uses a different combination of sensors and produces data at different rates:

| Mode     | Enum | Sensors Used               | Acc Rate | Gyro Rate | Mag Rate | Orientation Type |
| :------- | :--- | :------------------------- | :------- | :-------- | :------- | :--------------- |
| NDoF     | 0x01 | Accelerometer, Gyro, Mag   | 100 Hz   | 100 Hz    | 25 Hz    | Absolute         |
| IMU Plus | 0x02 | Accelerometer, Gyro        | 100 Hz   | 100 Hz    | N/A      | Relative         |
| Compass  | 0x03 | Accelerometer, Mag         | 25 Hz    | N/A       | 25 Hz    | Absolute         |
| M4G      | 0x04 | Accelerometer, Mag         | 50 Hz    | N/A       | 50 Hz    | Relative         |

**NDoF** (Nine Degrees of Freedom) calculates absolute orientation from all three sensors. It provides fast output with high robustness against magnetic field distortions. Fast Magnetometer calibration is enabled in this mode. After magnetic calibration is complete, heading reads 0° when facing magnetic north.

**IMU Plus** calculates relative orientation from the accelerometer and gyroscope. It provides high output data rate but heading will drift over time because no magnetometer is used for correction. Best for short\-duration applications.

**Compass** calculates geographic direction by combining gravity sensing (accelerometer) with magnetic field measurement. Accuracy depends on the stability of the surrounding magnetic field. Requires calibration.

**M4G** (Magnet for Gyroscope) is similar to IMU Plus but detects rotation via magnetometer instead of gyroscope. This results in lower power consumption and no gyroscope drift, but accuracy depends on the surrounding magnetic field. No magnetometer calibration is required in this mode.

When using the sensor fusion module, the underlying accelerometer, gyroscope, and magnetometer are automatically configured by the fusion algorithm. Do not configure these sensors separately via their individual module registers while sensor fusion is active.

### Calibration

The sensor fusion algorithm performs automatic background calibration of all sensors. This cannot be disabled. The **Calibration State** register (`0x0B`) reports the accuracy of each sensor's calibration on a scale from 0 (unreliable) to 3 (high accuracy). A device is considered fully calibrated when all three sensors report accuracy 3.

To calibrate: lay the device on a flat surface and rotate it in 45\-degree intervals while monitoring the calibration state values. Once calibrated, save the calibration offsets via the **Cal Data** registers (`0x0C`, `0x0D`, `0x0E`) and restore them on subsequent connections using boot macros to avoid repeating the calibration.

Nearby magnets, batteries, motors, electronics, jewelry, and steel structures will distort the magnetometer. If the environment cannot be guaranteed magnet\-free, avoid NDoF and M4G modes and use IMU Plus instead.

### Output Data Guidance

**Euler Angles** (Heading, Pitch, Roll) are suited for applications where tilt angles remain below ~30° and complete rotations are not required, such as digital compasses or pitch/roll monitoring. When pitch reaches 90°, heading becomes undetermined (gimbal lock).

**Quaternions** (W, X, Y, Z) are a four\-component representation that avoids gimbal lock and can describe any arbitrary 3D orientation. Use quaternions when the device may be oriented freely in space or when measuring complete rotations.

### Drift Behavior

In IMU Plus mode, pitch and roll drift are compensated by the accelerometer sensing gravity, but heading drifts over time due to gyroscope integration bias. When the device is stationary, the algorithm detects this and stops integrating gyroscope data to prevent additional drift. For applications requiring stable heading over extended periods (multiple hours), use a mode with magnetometer correction (NDoF or Compass).

The sensor fusion algorithm is optimized for tracking human motion. Under sustained large accelerations (e.g., vehicle cornering), the algorithm may incorrectly interpret the acceleration as gravity. Test the device for your specific use case if high\-acceleration environments are expected.

## 0xFE \- Debug Module

The **Module Opcode** is `0xFE`.

The **Debug Module** provides diagnostic and maintenance commands for the MetaWear. It can be used to reset the device, jump to the bootloader for firmware updates, or trigger a watchdog reset.

| Setting            | Address | Mode | Wlen | Rlen | Value                                                                                 |
| :----------------- | :------ | :--- | :--- | :--- | :------------------------------------------------------------------------------------ |
| Module Info        | `0x00`  | R    |      | 2    | Byte 0: Module Implementation ID: 0 (uint8\_t) Byte 1: Module Revision: 0+ (uint8\_t) |
| Reset              | `0x01`  | W    |      |      | No value required \- Performs a soft reset of the device                              |
| Jump to Bootloader | `0x02`  | W    |      |      | No value required \- Jumps to the bootloader for firmware update (DFU mode)           |
| Disconnect         | `0x06`  | W    |      |      | No value required \- Forces a BLE disconnect                                          |
| Reset After GC     | `0x05`  | W    |      |      | No value required \- Resets the device after performing garbage collection on flash   |
| Stack Overflow     | `0x03`  | W    |      |      | No value required \- For testing: triggers a stack overflow (debug only)              |

## Board Models

MetaWear boards come in several models, each with different sensor configurations. The board model is determined by reading the **Module Info** from the Settings Module (`0x11`) and the Accelerometer Module (`0x03`) during the discovery process.

| Model              | Enum Value | Accelerometer | Gyroscope | Magnetometer | Barometer | Temperature | Sensor Fusion |
| :----------------- | :--------- | :------------ | :-------- | :----------- | :-------- | :---------- | :------------ |
| MetaWear R         | 0          | MMA8452Q      | No        | No           | No        | On-die      | No            |
| MetaWear RG        | 1          | BMI160        | BMI160    | No           | No        | On-die      | No            |
| MetaWear RPro      | 2          | BMI160        | BMI160    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | No |
| MetaWear C         | 3          | MMA8452Q      | No        | No           | No        | On-die      | No            |
| MetaWear CPro      | 4          | BMI160        | BMI160    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | No |
| MetaEnvironment    | 5          | BMI160        | BMI160    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | No |
| MetaDetector       | 6          | BMI160        | BMI160    | No           | No        | On-die      | No            |
| MetaHealth         | 7          | BMI160        | BMI160    | No           | No        | On-die      | No            |
| MetaTracker        | 8          | BMI160        | BMI160    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | No |
| MetaMotion R       | 9          | BMI160        | BMI160    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | Yes |
| MetaMotion RL      | 10         | BMI160        | BMI160    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | Yes |
| MetaMotion C       | 11         | BMI160        | BMI160    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | Yes |
| MetaMotion S       | 12         | BMI270        | BMI270    | BMM150       | BMP280    | On-die, Thermistor, BMP280 | Yes |

The MetaMotion S (MMS) is the newest board and uses the BMI270 IMU. The MetaMotion RL (MMRL) and earlier boards use the BMI160. Older boards (MetaWear R, MetaWear C) use the MMA8452Q accelerometer which has a completely different register set (see the MMA8452Q section below).

### Active Board Hardware Specifications

The two currently supported MetaSensor boards are:

| Spec             | MetaMotion RL (MMRL)                          | MetaMotion S (MMS)                                         |
| :--------------- | :-------------------------------------------- | :--------------------------------------------------------- |
| Battery          | 100mAh rechargeable Li-Po                     | 100mAh rechargeable Li-Po                                  |
| Battery Life     | 1 \- 14 days (usage dependent)                | 8 \- 24 days (usage dependent)                             |
| Physical Size    | 17mm x 25mm x 5mm                             | 17mm x 25mm x 5mm                                         |
| Log Memory       | 8MB (~0.5M log entries)                        | 512MB (~30M log entries)                                   |
| Accelerometer    | BMI160                                         | BMI270                                                     |
| Gyroscope        | BMI160                                         | BMI270                                                     |
| Magnetometer     | BMM150                                         | BMM150                                                     |
| Barometer / Temp | BMP280                                         | BMP280                                                     |
| Ambient Light    | None                                           | LTR\-329ALS\-01                                            |
| Charging         | Micro\-USB                                     | Micro\-USB                                                 |
| MCU              | ARM (Nordic nRF52)                             | ARM (Nordic nRF52)                                         |

Streaming sensor data consumes more battery than logging because the active BLE connection draws additional power. When logging, the device can operate disconnected; however, downloading large logs after reconnection can take hours depending on the amount of data stored.

The proximity of the battery to the PCB when the board is enclosed in a case may reduce Bluetooth range.

**Datasheets**: [MetaMotion RL (MMRL)](https://mbientlab.com/docs/MetaMotionR-PS5.pdf) | [MetaMotion S (MMS)](https://mbientlab.com/docs/MetaMotionS-PS5.pdf)

**Sensor Datasheets**: [BMI160](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi160) | [BMI270](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi270) | [BMM150](https://www.bosch-sensortec.com/products/motion-sensors/magnetometers-bmm150/) | [BMP280](https://www.bosch-sensortec.com/products/environmental-sensors/pressure-sensors/bmp280/) | [LTR\-329ALS\-01](http://www.mouser.com/ds/2/239/Lite-On_LTR-329ALS-01%20DS_ver1.1-348647.pdf)

### Logging Memory Capacity

Each log entry occupies 8 bytes in flash memory. Sensors that produce multi\-axis data (e.g., accelerometer XYZ) require multiple log entries per sample. The number of log entries per sample determines how quickly memory fills at a given data rate.

| Sensor Output        | Log Entries Per Sample | Notes                                |
| :------------------- | :--------------------- | :----------------------------------- |
| Accelerometer (XYZ)  | 2                      | 6 bytes of data split across 2 entries |
| Gyroscope (XYZ)      | 2                      | 6 bytes of data split across 2 entries |
| Magnetometer (XYZ)   | 2                      | 6 bytes of data split across 2 entries |
| Temperature          | 1                      | Single value                          |
| Pressure             | 1                      | Single value                          |
| Humidity             | 1                      | Single value                          |
| Ambient Light        | 1                      | Single value                          |
| Color                | 2                      | Multi\-channel value                  |
| Proximity            | 1                      | Single value                          |
| Euler Angles         | 4                      | Heading, Pitch, Roll as floats        |
| Quaternion           | 4                      | W, X, Y, Z as floats                  |
| Gravity Vector       | 3                      | X, Y, Z as floats                     |
| Linear Acceleration  | 3                      | X, Y, Z as floats                     |

**Log capacity per board model** (total log entries):

| Model           | Log Entries  | Approx. Memory | Battery (mAh) |
| :-------------- | :----------- | :------------- | :------------ |
| MetaMotion S    | 67,108,864   | 512 MB         | 100           |
| MetaMotion C    | 1,048,576    | 8 MB           | 230           |
| MetaMotion R    | 1,048,576    | 8 MB           | 100           |
| MetaMotion RL   | 1,048,576    | 8 MB           | 100           |
| MetaTracker     | 262,144      | 2 MB           | 600           |
| MetaWear C      | 7,552        | ~59 KB         | 230           |
| MetaWear CPro   | 7,552        | ~59 KB         | 230           |
| MetaEnvironment | 7,552        | ~59 KB         | 230           |
| MetaDetector    | 7,552        | ~59 KB         | 230           |
| MetaWear R      | 7,552        | ~59 KB         | 100           |
| MetaWear RG     | 7,552        | ~59 KB         | 100           |
| MetaWear RPro   | 7,552        | ~59 KB         | 100           |

**Estimating time to fill log**: Time (seconds) = Log Entries / (sum of entries\-per\-second across all active sensors). For example, logging accelerometer at 100 Hz on an MMS uses 200 entries/sec (100 samples × 2 entries), giving 67,108,864 / 200 = 335,544 seconds ≈ 3.9 days.

### Sensor Data Rates

The available output data rates (ODR) for each sensor, configurable via the sensor's Data Config register:

**Accelerometer**:

| Chip                | Available ODR (Hz)                                    |
| :------------------ | :---------------------------------------------------- |
| BMI160 (MMRL)       | 12.5, 25, 50, 100, 200, 400, 800                     |
| BMI270 (MMS)        | 12.5, 25, 50, 100, 200, 400, 800                     |
| BMA255 (MetaEnv/Det)| 15.62, 31.26, 62.5, 125, 250, 500                    |
| MMA8452Q (MW R/C)   | 1.56, 6.25, 12.5, 50, 100, 200, 400, 800             |

**Gyroscope** (BMI160 / BMI270): 25, 50, 100, 200, 400, 800 Hz

**Magnetometer** (BMM150): 10, 15, 20, 25 Hz

**Barometer** (BMP280): 0.25, 0.50, 0.99, 1.96, 3.82, 7.33, 13.5, 83.3 Hz

**Barometer** (BME280, MetaTracker): 0.99, 1.96, 3.82, 7.33, 13.5, 31.8, 46.5, 83.3 Hz

**Ambient Light** (LTR\-329ALS\-01, MMS only): 0.5, 1, 2, 5, 10 Hz

**Temperature**: No fixed ODR. Temperature is read on demand or via a timer. Common polling intervals: 1s, 15s, 30s, 1m, 15m, 30m, 1hr.

**Sensor Fusion Outputs** (Euler, Quaternion, Gravity, Linear Acc): Fixed at 100 Hz (determined by the fusion mode, not independently configurable).

### Sensor Power Consumption

Approximate current draw per sensor when active (excluding BLE overhead):

| Sensor                | Current Draw (mA) |
| :-------------------- | :----------------- |
| Accelerometer BMI160  | 0.18               |
| Accelerometer BMI270  | 0.21               |
| Accelerometer BMA255  | 0.13               |
| Accelerometer MMA8452Q| 0.165              |
| Gyroscope             | 0.5                |
| Magnetometer          | 0.5                |
| Temperature           | 0.01               |
| Pressure (BMP280)     | 0.0034             |
| Pressure (BME280)     | 0.0028             |
| Humidity              | 0.0018             |
| Ambient Light         | 0.22               |
| Color / Proximity     | 0.065              |
| Sensor Fusion (any)   | 0.925              |

**BLE streaming overhead**: ~7.5 mA when actively streaming data over Bluetooth. When logging without a BLE connection, this overhead is zero.

**Battery life estimate**: Battery capacity (mAh) / (BLE overhead + sum of active sensor currents) = hours of operation. For example, an MMS (100 mAh) streaming accelerometer BMI270 (0.21 mA) over BLE: 100 / (7.5 + 0.21) ≈ 13 hours. The same configuration logging (no BLE): 100 / 0.21 ≈ 476 hours ≈ 19.8 days.

## Data Processor Filter Configuration

When creating a filter via the **Data Processor Module** (0x09) **Add** register (0x02), the filter configuration bytes (bytes 4+) are structured differently for each filter type. This section details the byte layout for each filter.

### Passthrough (Type 0x01)

Passes data through, optionally with a count limit or conditional gate.

| Byte | Field | Size   | Description                                                                       |
| :--- | :---- | :----- | :-------------------------------------------------------------------------------- |
| 0    | Mode  | 3 bits | 0 \= Pass All, 1 \= Pass Conditional (block until count > 1), 2 \= Pass Count (pass X samples then stop) |
| 1-2  | Value | uint16\_t | Count or condition value                                                       |

**State (Set/Reset)**: Value \- uint16\_t count or condition.

### Counter / Accumulator (Type 0x02)

Counts events or accumulates data values.

| Byte     | Field        | Size   | Description                                                    |
| :------- | :----------- | :----- | :------------------------------------------------------------- |
| 0 [0-1]  | Output Length| 2 bits | Output data length (0 \= 1 byte, 3 \= 4 bytes)                |
| 0 [2-3]  | Input Length | 2 bits | Input value length (0 \= 1 byte, 3 \= 4 bytes)                |
| 0 [4-6]  | Mode         | 3 bits | 0 \= Accumulate Values (sum), 1 \= Count Events (increment)   |

**State (Set/Reset)**: Count Value \- uint32\_t (4 bytes).

### Low Memory Average / Low Pass (Type 0x03)

Computes a running average using a recursive algorithm that uses minimal memory.

| Byte     | Field        | Size   | Description                                                                           |
| :------- | :----------- | :----- | :------------------------------------------------------------------------------------ |
| 0 [0-1]  | Output Length| 2 bits | Output data length (0 \= 1 byte, 3 \= 4 bytes)                                       |
| 0 [2-3]  | Input Length | 2 bits | Input value length (0 \= 1 byte, 3 \= 4 bytes)                                       |
| 0 [5]    | Mode         | 1 bit  | 0 \= Low pass filter, 1 \= High pass filter (firmware >= 1.2.2)                       |
| 1        | Size         | uint8\_t | Internal buffer size                                                                 |
| 2        | Count        | uint8\_t | Depth of average. Non-power-of-2 results in slower software divide.                  |

**State (Set/Reset)**: No parameters \- resets the running average.

### Pulse Detector (Type 0x04)

Detects pulses (peaks above a threshold for a minimum width) in the input data.

| Byte | Field                 | Size     | Description                                                        |
| :--- | :-------------------- | :------- | :----------------------------------------------------------------- |
| 0    | Length                | uint8\_t | Length of input data in bytes                                       |
| 1    | Trigger Mode          | uint8\_t | Trigger output mode                                                |
| 2    | Output Mode           | uint8\_t | What to output when pulse detected                                 |
| 3-6  | Threshold             | 4 bytes  | Minimum value for pulse detection                                  |
| 7-8  | Width                 | uint16\_t | Minimum pulse width in samples                                    |

### Peak Detector (Type 0x05)

Detects local maxima or minima in the input data stream.

| Byte | Field           | Size     | Description                                                                      |
| :--- | :-------------- | :------- | :------------------------------------------------------------------------------- |
| 0    | Look Ahead      | uint8\_t | Number of samples to look ahead for a bigger peak                                |
| 1    | Looking For Max | uint8\_t | 0 \= detect min peaks, 1 \= detect max peaks                                    |
| 2    | Extra Peak Width| uint8\_t | Number of samples to sum on both sides of the peak (becomes output value)        |
| 3-6  | Delta           | 4 bytes  | Minimum delta that must occur before a peak is considered                         |
| 7    | Offset          | uint8\_t | Offset in bytes from beginning of input data                                     |
| 8    | Length          | uint8\_t | Length of input data in bytes                                                     |
| 9    | Is Signed       | uint8\_t | 0 \= unsigned input, 1 \= signed input                                          |

### Comparator (Type 0x06)

Compares input data against one or more reference values and only passes data that satisfies the comparison.

**Single Comparator** (firmware < 1.2.3):

| Byte | Field      | Size     | Description                                                                         |
| :--- | :--------- | :------- | :---------------------------------------------------------------------------------- |
| 0    | Is Signed  | uint8\_t | 0 \= unsigned comparison, 1 \= signed comparison                                   |
| 1    | Operation  | uint8\_t | 0 \= EQ, 1 \= NEQ, 2 \= LT, 3 \= LTE, 4 \= GT, 5 \= GTE                         |
| 2    | (reserved) | uint8\_t | Unused                                                                              |
| 3-6  | Reference  | 4 bytes  | Reference value to compare against                                                  |

**Multi-Value Comparator** (firmware >= 1.2.3):

| Byte     | Field      | Size    | Description                                                                                              |
| :------- | :--------- | :------ | :------------------------------------------------------------------------------------------------------- |
| 0 [0]    | Is Signed  | 1 bit   | 0 \= unsigned, 1 \= signed                                                                               |
| 0 [1-2]  | Length     | 2 bits  | Data length encoding                                                                                      |
| 0 [3-5]  | Operation  | 3 bits  | 0 \= EQ, 1 \= NEQ, 2 \= LT, 3 \= LTE, 4 \= GT, 5 \= GTE                                               |
| 0 [6-7]  | Mode       | 2 bits  | 0 \= Absolute (return data as-is), 1 \= Reference (return matched ref), 2 \= Zone (return ref index), 3 \= Binary (return 1/0) |
| 1-12     | References | 12 bytes| Up to 3 reference values (4 bytes each)                                                                   |

### RMS / RSS (Type 0x07 / 0x08)

Computes the Root Mean Square (RMS) or Root Sum Square (RSS) of multi-component data (e.g., XYZ accelerometer data into a single magnitude value).

| Byte     | Field       | Size    | Description                                                     |
| :------- | :---------- | :------ | :-------------------------------------------------------------- |
| 0 [0-1]  | Output Length| 2 bits | Output data length                                               |
| 0 [2-3]  | Input Length | 2 bits | Input component length                                           |
| 0 [4-6]  | Count       | 3 bits | Number of input components (e.g., 3 for XYZ)                    |
| 0 [7]    | Is Signed   | 1 bit  | 0 \= unsigned, 1 \= signed                                      |
| 1        | Mode        | uint8\_t| 0 \= RMS, 1 \= RSS                                             |

### Math (Type 0x09)

Performs arithmetic operations on the input data.

| Byte     | Field       | Size     | Description                                                  |
| :------- | :---------- | :------- | :----------------------------------------------------------- |
| 0 [0-1]  | Output Length| 2 bits  | Output data length                                            |
| 0 [2-3]  | Input Length | 2 bits  | Input data length                                             |
| 0 [4]    | Is Signed   | 1 bit   | 0 \= unsigned, 1 \= signed                                   |
| 1        | Operation   | uint8\_t | See math operations table below                              |
| 2-5      | RHS         | 4 bytes  | Right-hand side operand (little-endian)                       |
| 6        | N Channels  | uint8\_t | Number of input channels (firmware >= 1.1.0)                 |

The math operations are:

| Value | Operation | Formula             |
| :---- | :-------- | :------------------ |
| 1     | Add       | input + rhs         |
| 2     | Multiply  | input × rhs        |
| 3     | Divide    | input / rhs         |
| 4     | Modulus   | input % rhs         |
| 5     | Exponent  | input ^ rhs         |
| 6     | Sqrt      | sqrt(input)         |
| 7     | Left Shift| input << rhs        |
| 8     | Right Shift| input >> rhs       |
| 9     | Subtract  | input \- rhs        |
| 10    | Abs Value | \|input\|           |
| 11    | Constant  | rhs (replaces input)|

### Sample Delay (Type 0x0A)

Delays data by a fixed number of samples (FIFO buffer).

| Byte | Field    | Size     | Description                   |
| :--- | :------- | :------- | :---------------------------- |
| 0    | Length   | uint8\_t | Length of each sample in bytes |
| 1    | Bin Size | uint8\_t | Number of samples to buffer    |

### Threshold (Type 0x0B)

Detects when data crosses a threshold boundary.

| Byte     | Field      | Size    | Description                                                                                                    |
| :------- | :--------- | :------ | :------------------------------------------------------------------------------------------------------------- |
| 0 [0-1]  | Length     | 2 bits  | Data length encoding                                                                                            |
| 0 [2]    | Is Signed  | 1 bit   | 0 \= unsigned, 1 \= signed                                                                                     |
| 0 [3-5]  | Mode       | 3 bits  | 0 \= Absolute (output data as-is), 1 \= Binary (output 1 if rising, \-1 if falling)                           |
| 1-4      | Boundary   | 4 bytes | Threshold value                                                                                                 |
| 5-6      | Hysteresis | uint16\_t | Hysteresis value to prevent rapid toggling                                                                    |

### Delta (Type 0x0C)

Detects when data changes by a specified magnitude from the last reported value.

| Byte     | Field      | Size    | Description                                                                                                     |
| :------- | :--------- | :------ | :-------------------------------------------------------------------------------------------------------------- |
| 0 [0-1]  | Length     | 2 bits  | Data length encoding                                                                                             |
| 0 [2]    | Is Signed  | 1 bit   | 0 \= unsigned, 1 \= signed                                                                                      |
| 0 [3-5]  | Mode       | 3 bits  | 0 \= Absolute (output when delta exceeded), 1 \= Differential (output the difference), 2 \= Binary (output 1/\-1) |
| 1-4      | Magnitude  | 4 bytes | Minimum change required to trigger output                                                                        |

### Time Delay (Type 0x0E)

Periodically passes the latest data sample, effectively downsampling by time.

| Byte     | Field  | Size    | Description                                                  |
| :------- | :----- | :------ | :----------------------------------------------------------- |
| 0 [0-2]  | Length | 3 bits  | Data length encoding                                          |
| 0 [3-5]  | Mode   | 3 bits  | Output mode                                                   |
| 1-4      | Period | uint32\_t | Time period in ms between outputs                           |

### Buffer (Type 0x0F)

Stores the latest sample for on-demand retrieval via the **State** register.

| Byte     | Field  | Size   | Description          |
| :------- | :----- | :----- | :------------------- |
| 0 [0-4]  | Length | 5 bits | Data length in bytes |

### Packer (Type 0x10)

Packs multiple data samples into a single BLE notification to reduce overhead.

| Byte     | Field  | Size   | Description                     |
| :------- | :----- | :----- | :------------------------------ |
| 0 [0-4]  | Length | 5 bits | Length of each sample in bytes  |
| 1 [0-4]  | Count  | 5 bits | Number of samples to pack       |

### Accounter (Type 0x11)

Adds a timestamp or counter to each data sample.

| Byte     | Field    | Size   | Description                                                  |
| :------- | :------- | :----- | :----------------------------------------------------------- |
| 0 [0-3]  | Mode     | 4 bits | 0 \= Add counter, 1 \= Add timestamp                        |
| 0 [4-5]  | Length   | 2 bits | Account data length                                           |
| 1 [0-3]  | Prescale | 4 bits | Prescaler for the counter/timer                               |

### Type 0x12 - Fuser

Combines data from multiple sources. Waits until all inputs have new data, then outputs the combined result.

| Byte     | Field      | Size    | Description                                               |
| :------- | :--------- | :------ | :-------------------------------------------------------- |
| 0 [0-3]  | Count      | 4 bits  | Number of input sources                                    |
| 1-12     | References | 12 bytes| Filter IDs of the input sources (up to 3, 4 bytes each)   |

## Workflow Examples

### Example: Periodic Temperature Reading

This example uses the **Timer Module** (0x0C) and the **Event Module** (0x0A) to read the temperature every 5 seconds.

**Step 1**: Create a timer with a 5000ms period.

Send: \[`0x0C, 0x02, 0x88, 0x13, 0x00, 0x00, 0xFF, 0xFF, 0x00`\]

* `0x0C` \- Timer module
* `0x02` \- Timer Entry register
* `0x88 0x13 0x00 0x00` \- Period: 5000ms (uint32\_t little-endian)
* `0xFF 0xFF` \- Repeat count: indefinite
* `0x00` \- No delay

The MetaWear responds with the Timer ID (e.g., `0x00`).

**Step 2**: Begin recording an event for this timer.

Send: \[`0x0A, 0x02, 0x0C, 0x06, 0x00`\]

* `0x0A` \- Event module
* `0x02` \- Entry register
* `0x0C 0x06 0x00` \- Source: Timer Module (0x0C), Notify register (0x06), Timer ID 0

**Step 3**: Record the command to execute (read temperature).

Send: \[`0x0A, 0x03, 0x04, 0x81, 0x00`\]

* `0x0A` \- Event module
* `0x03` \- Cmd Parameters register
* `0x04 0x81 0x00` \- The command: read temperature sensor index 0

**Step 4**: Start the timer.

Send: \[`0x0C, 0x03, 0x00`\]

* `0x0C` \- Timer module, `0x03` \- Start register, `0x00` \- Timer ID

Now the MetaWear will automatically read and send back the temperature every 5 seconds.

### Example: Logging Readout Protocol

The logging readout uses a page-based protocol to reliably transfer large amounts of data over BLE.

**Step 1**: Enable logging and add a trigger for accelerometer data.

Send: \[`0x0B, 0x02, 0x03, 0x04, 0x00, 0x06`\]

* `0x0B` \- Logging module
* `0x02` \- Add Trigger register
* `0x03 0x04` \- Source: Accelerometer (0x03), Data Interrupt (0x04)
* `0x00 0x06` \- Data offset 0, length 6 bytes

Send: \[`0x0B, 0x01, 0x01`\] to enable logging.

**Step 2**: After some time, initiate readout.

Send: \[`0x0B, 0x06, 0xFF, 0xFF, 0xFF, 0xFF`\] to read all entries.

**Step 3**: Process the page-based readout.

The MetaWear sends log entries via the **Readout Notify** register (0x07). Each notification contains: Trigger ID, Timestamp (uint32\_t), and logged data.

Progress updates arrive via the **Readout Progress** register (0x08) with the number of entries remaining.

When a page is complete, the **Page Completed** register (0x0D) fires. The host must confirm receipt by writing to the **Page Confirm** register (0x0E), which triggers the next page.

This continues until all entries have been transferred and the progress reaches 0.

### Example: LED Blink on Button Press

This example uses the **Event Module** to flash the LED when the button is pressed.

**Step 1**: Record an event triggered by the switch.

Send: \[`0x0A, 0x02, 0x01, 0x01, 0x00`\]

* Source: Switch Module (0x01), Switch State register (0x01)

**Step 2**: Record the LED play command.

Send: \[`0x0A, 0x03, 0x02, 0x01, 0x01`\]

* Command: LED Module (0x02), Play register (0x01), Value 0x01 (play)

Now pressing the button will automatically play the LED pattern.

## Settings Module Firmware Revisions

The **Settings Module** (0x11) has expanded over firmware versions. Not all registers are available on all boards. The module revision (from **Module Info**) determines which registers are supported:

| Register                | Address | Min Revision | Notes                                       |
| :---------------------- | :------ | :----------- | :------------------------------------------ |
| Device Name             | `0x01`  | All          | Available on all firmware versions           |
| Advertising Interval    | `0x02`  | All          | Available on all firmware versions           |
| TX Power                | `0x03`  | All          | Available on all firmware versions           |
| Start Advertisement     | `0x05`  | All          | Available on all firmware versions           |
| Scan Response Packet    | `0x07`  | All          | Available on all firmware versions           |
| Partial Scan Response   | `0x08`  | All          | Available on all firmware versions           |
| Connection Parameters   | `0x09`  | Rev 1+       | MMRL and later                               |
| Disconnect Event        | `0x0A`  | Rev 1+       | MMRL and later                               |
| MAC Address             | `0x0B`  | Rev 1+       | MMRL and later                               |
| Battery State           | `0x0C`  | Rev 1+       | MMRL and later; MMS has charger support      |
| Power Status            | `0x11`  | Rev 5+       | MMS only \- USB cable detection              |
| Charge Status           | `0x12`  | Rev 5+       | MMS only \- charging state                   |
| Whitelist Filter Mode   | `0x13`  | Rev 5+       | MMS only                                     |
| Whitelist Addresses     | `0x14`  | Rev 5+       | MMS only                                     |
| 3V Power                | `0x1C`  | Rev 5+       | MMS only \- 3.3V regulator control           |
| Force 1M PHY            | `0x1D`  | Rev 5+       | MMS only \- force Bluetooth 1M PHY           |

Key firmware version thresholds:

* **Revision 1** (MMRL): Added connection parameters, disconnect events, MAC address, and battery state
* **Revision 5** (MMS): Added power/charge monitoring, whitelist management, 3V regulator control, and PHY selection

The multi-value comparator in the Data Processor Module also has a firmware dependency: the **Multi-Value Comparator** mode (with Zone and Binary output modes) requires firmware >= 1.2.3.

## Error and Status Codes

The MetaWear SDK defines the following status codes. While these are SDK-level codes (not sent over BLE), they indicate common error conditions that hosts should handle:

| Code | Name                                    | Value | Description                                                                      |
| :--- | :-------------------------------------- | :---- | :------------------------------------------------------------------------------- |
| OK   | `STATUS_OK`                             | 0     | Operation completed successfully                                                 |
| WARN | `WARNING_UNEXPECTED_SENSOR_DATA`        | 1     | Received sensor data that does not match any known subscription                  |
| WARN | `WARNING_INVALID_PROCESSOR_TYPE`        | 2     | Attempted to modify a processor with the wrong type-specific function            |
| ERR  | `ERROR_UNSUPPORTED_PROCESSOR`           | 4     | Attempted to create a processor type not supported by the firmware                |
| WARN | `WARNING_INVALID_RESPONSE`              | 8     | Received a response that does not match any pending request                      |
| ERR  | `ERROR_TIMEOUT`                         | 16    | A command or read operation timed out waiting for a response                      |
| ERR  | `ERROR_SERIALIZATION_FORMAT`            | 32    | Board state deserialization failed due to format mismatch                         |
| ERR  | `ERROR_ENABLE_NOTIFY`                   | 64    | Failed to enable BLE notifications on the Notification Characteristic            |

At the BLE protocol level, error conditions typically manifest as:

* **No response**: The module opcode or register address is invalid, or the module is not present on the board. Always use the Module Discovery process to verify module availability before sending commands.
* **Unexpected data length**: The response contains fewer or more bytes than expected. This can happen when reading a register that requires specific input parameters (e.g., temperature sensor index).
* **Stale data**: Notifications may arrive for previously subscribed data sources after a reconnect. Clear subscriptions and re-subscribe after each connection.

## XYZ Axis Orientation

When interpreting accelerometer, gyroscope, and magnetometer data, the axis orientation depends on the IMU chip mounted on the board. With the board positioned face\-up and the USB port pointing away from you:

For **BMI160** (MMRL and earlier boards) and **BMI270** (MMS):

* **X\-axis**: Points to the right (positive direction)
* **Y\-axis**: Points upward / away from you (positive direction)
* **Z\-axis**: Points out of the board face, toward you (positive direction)

This follows a right\-hand coordinate system. When the board is lying flat on a table face\-up, the Z\-axis accelerometer reading should be approximately +1g (9.81 m/s²) due to gravity.

The magnetometer (BMM150) follows the same axis convention as the IMU.

Refer to the [BMI160 datasheet](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi160/) or [BMI270 datasheet](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi270/) for the full axis orientation diagrams.

## MAC Address and BLE Advertising

Each MetaSensor has a unique 6\-byte MAC address (e.g., `F1:4A:45:90:AC:9D`) that serves as its identifier. The MAC address is available in three ways:

1. **BLE Advertising Packet**: The MAC is included in the device's advertising data and can be read during scanning before establishing a connection.
2. **Settings Module Register**: On MMRL and later boards (Settings Module revision 1+), the MAC address can be read via register `0x0B` of the Settings Module (`0x11`).
3. **Physical Label**: The MAC is printed on a sticker on the back of the device.

**Platform Note**: On Android and Windows, the MAC address is directly accessible from the BLE scan callback. On iOS, Apple obfuscates the MAC address in advertising packets and replaces it with an auto\-generated `CBUUID`. To retrieve the actual MAC on iOS, use the Settings Module register `0x0B` after establishing a connection, or read it from the custom advertising data in the MetaWear advertising packet.