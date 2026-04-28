# MetaWear BLE Protocol Reference

Every byte-level detail needed to implement the MetaWear BLE serial protocol
from scratch in any language.

---

## BLE Transport Layer

All commands are written to a single GATT characteristic. All responses (notifications)
arrive on a single notify characteristic.

```
Service UUID:        0x326a9000-85cb-9195-d9dd-464cfbbae75a
Command char UUID:   0x326a9001-85cb-9195-d9dd-464cfbbae75a   (write without response, or with response for MACRO commands)
Notify char UUID:    0x326a9006-85cb-9195-d9dd-464cfbbae75a   (subscribe for notifications)
```

Device Info service (standard BLE 0x180A):
```
Firmware revision:   0x2A26
Model number:        0x2A24
Hardware revision:   0x2A27
Manufacturer:        0x2A29
Serial number:       0x2A25
```

### Packet Format

Every command and every notification follows the same two- or three-byte header:

```
Byte 0: module_id
Byte 1: register_id  (bit 7 = 0x80 set means "READ" request/response)
Byte 2: data_id      (only present for signals that have an ID, e.g. timer, logger entries)
Bytes 3+: payload
```

Macros are the only commands written **with** response; everything else uses
write-without-response.

The READ modifier:
```
#define READ_REGISTER(x)   (0x80 | x)
#define CLEAR_READ(x)      (0x3f & x)
```

---

## Module IDs

```
MBL_MW_MODULE_SWITCH          = 0x01
MBL_MW_MODULE_LED             = 0x02
MBL_MW_MODULE_ACCELEROMETER   = 0x03
MBL_MW_MODULE_TEMPERATURE     = 0x04
MBL_MW_MODULE_GPIO            = 0x05
MBL_MW_MODULE_NEO_PIXEL       = 0x06
MBL_MW_MODULE_IBEACON         = 0x07
MBL_MW_MODULE_HAPTIC          = 0x08
MBL_MW_MODULE_DATA_PROCESSOR  = 0x09
MBL_MW_MODULE_EVENT           = 0x0A
MBL_MW_MODULE_LOGGING         = 0x0B
MBL_MW_MODULE_TIMER           = 0x0C
MBL_MW_MODULE_I2C             = 0x0D   (Serial Passthrough: I2C + SPI)
MBL_MW_MODULE_MACRO           = 0x0F
MBL_MW_MODULE_CONDUCTANCE     = 0x10
MBL_MW_MODULE_SETTINGS        = 0x11
MBL_MW_MODULE_BAROMETER       = 0x12
MBL_MW_MODULE_GYRO            = 0x13
MBL_MW_MODULE_AMBIENT_LIGHT   = 0x14
MBL_MW_MODULE_MAGNETOMETER    = 0x15
MBL_MW_MODULE_HUMIDITY        = 0x16
MBL_MW_MODULE_COLOR_DETECTOR  = 0x17
MBL_MW_MODULE_PROXIMITY       = 0x18
MBL_MW_MODULE_SENSOR_FUSION   = 0x19
MBL_MW_MODULE_DEBUG           = 0xFE
```

---

## Board Initialization Sequence

1. Enable notifications on the notify characteristic.
2. Read each Device Info GATT characteristic in order (firmware, model, hardware, manufacturer, serial).
3. For each module in `MODULE_DISCOVERY_CMDS` list, send:
   ```
   [module_id, 0x80]   <- READ_REGISTER(0x00) = info register
   ```
   The board responds with `[module_id, 0x80, implementation_byte, revision_byte, ...]`.
4. After all modules respond, call `init_*_module()` for each present module.
5. Read logging time signal (module 0x0B, register 0x84) and set reference epoch.

Module discovery order:
```
SWITCH, LED, ACCELEROMETER, TEMPERATURE, GPIO, IBEACON, HAPTIC,
DATA_PROCESSOR, EVENT, LOGGING, TIMER, I2C, MACRO, SETTINGS,
BAROMETER, GYRO, AMBIENT_LIGHT, MAGNETOMETER, HUMIDITY,
SENSOR_FUSION, DEBUG
```

Model numbers (from `module_number` string in Device Info "model number" characteristic):
```
"0" -> MetaWear R
"1" -> MetaWear RG  (or RPro if barometer + ambient light present)
"2" -> MetaWear C   (or CPro if magnetometer present, or MetaEnv if humidity present)
"3" -> MetaHealth
"4" -> MetaTracker
"5" -> MetaMotion R (or RL if no ambient light)
"6" -> MetaMotion C
"8" -> MetaMotion S
```

Hardware revisions (from `hardware_revision` string, characteristic `0x2A27`) shipped per model:
```
"5" -> r0.1, r0.2, r0.3, r0.4, r0.5      (MetaMotion R / RL)
"8" -> r0.1                              (MetaMotion S)
```
Some firmware reports the bare `0.X` form without the leading `r`; treat both forms as equivalent.

---

## Module 0x01 — Switch

No register header file found in the public SDK. Module responds to info query only.
The switch button state is a data signal; response is a 1-byte value (0=released, 1=pressed).

---

## Module 0x02 — LED

Register opcodes:
```
LED_PLAY   = 0x01
LED_STOP   = 0x02
LED_CONFIG = 0x03
```

### Write LED Pattern
```
[0x02, 0x03, color, 0x02, <13 bytes MblMwLedPattern>]
```
`color` is an enum: 0=Green, 1=Red, 2=Blue (from `MblMwLedColor`).

`MblMwLedPattern` (13 bytes, packed):
```
uint8_t  high_intensity      // 0..31
uint8_t  low_intensity       // 0..31
uint16_t rise_time_ms
uint16_t high_time_ms
uint16_t fall_time_ms
uint16_t pulse_duration_ms
uint16_t delay_time_ms       // only if revision >= 1 (DELAYED_REVISION)
uint8_t  repeat_count        // 0xFF = indefinite (use 0xFF, not 0; 0 causes undefined behaviour on firmware)
```
Total command: 17 bytes.

### Play / Pause / Stop
```
Play:          [0x02, 0x01, 0x01]
Autoplay:      [0x02, 0x01, 0x02]
Pause:         [0x02, 0x01, 0x00]
Stop:          [0x02, 0x02, 0x00]
Stop+Clear:    [0x02, 0x02, 0x01]
```

### Required sequence
Always send **Stop+Clear before writing a new pattern**. The firmware does not reset LED state on BLE reconnection; stale patterns from a previous session persist until explicitly cleared.

```
Stop+Clear  →  WritePattern (per channel)  →  Play
```

---

## Module 0x03 — Accelerometer

### Implementation Types
```
MBL_MW_MODULE_ACC_TYPE_BMI160 = 1
MBL_MW_MODULE_ACC_TYPE_BMI270 = 4
```

### BMI160 Register Opcodes
```
POWER_MODE                = 0x01
DATA_INTERRUPT_ENABLE     = 0x02
DATA_CONFIG               = 0x03
DATA_INTERRUPT            = 0x04
DATA_INTERRUPT_CONFIG     = 0x05
MOTION_INTERRUPT_ENABLE   = 0x09
MOTION_CONFIG             = 0x0A
MOTION_INTERRUPT          = 0x0B
TAP_INTERRUPT_ENABLE      = 0x0C
TAP_CONFIG                = 0x0D
TAP_INTERRUPT             = 0x0E
ORIENT_INTERRUPT_ENABLE   = 0x0F
ORIENT_CONFIG             = 0x10
ORIENT_INTERRUPT          = 0x11
STEP_DETECTOR_INTERRUPT_EN= 0x17
STEP_DETECTOR_CONFIG      = 0x18
STEP_DETECTOR_INTERRUPT   = 0x19
STEP_COUNTER_DATA         = 0x1A
STEP_COUNTER_RESET        = 0x1B
PACKED_ACC_DATA           = 0x1C
```

### BMI270 Register Opcodes
```
POWER_MODE                = 0x01
DATA_INTERRUPT_ENABLE     = 0x02
DATA_CONFIG               = 0x03
DATA_INTERRUPT            = 0x04
PACKED_ACC_DATA           = 0x05
FEATURE_ENABLE            = 0x06
FEATURE_INTERRUPT_ENABLE  = 0x07
FEATURE_CONFIG            = 0x08
MOTION_INTERRUPT          = 0x09
WRIST_INTERRUPT           = 0x0A
STEP_COUNT_INTERRUPT      = 0x0B
ACTIVITY_INTERRUPT        = 0x0C
TEMP_INTERRUPT            = 0x0D
TEMP_ENABLE               = 0x0E
TEMP                      = 0x0F
OFFSET                    = 0x10
DOWNSAMPLING              = 0x11
```

### Commands

**Start / Stop sampling:**
```
Start:  [0x03, 0x01, 0x01]
Stop:   [0x03, 0x01, 0x00]
```

**Enable / Disable data stream (BMI160):**
```
Enable:  [0x03, 0x02, 0x01, 0x00]
Disable: [0x03, 0x02, 0x00, 0x01]
```

**Write acceleration config (BMI160 / BMI270):**
```
[0x03, 0x03, acc_conf_byte, acc_range_byte]
```
`acc_conf_byte` layout — **BMI160**:
```
bits 0-3: odr  (MblMwAccBmi160Odr + 1, i.e. 1-indexed)
bits 4-6: bwp  (2 = normal for ODR >= 12.5 Hz; must be 0 when acc_us is set)
bit  7:   us   (under-sampling: 1 for ODR < 12.5 Hz, 0 otherwise)
```
Reference values:
```
0.78125 Hz -> 0x81    12.5 Hz -> 0x25    200 Hz -> 0x29
  1.5625 Hz -> 0x82    25   Hz -> 0x26    400 Hz -> 0x2A
   3.125 Hz -> 0x83    50   Hz -> 0x27    800 Hz -> 0x2B
    6.25 Hz -> 0x84   100   Hz -> 0x28   1600 Hz -> 0x2C
```

`acc_conf_byte` layout — **BMI270** (different from BMI160):
```
bits 0-3: acc_odr          (same 1-indexed codes as BMI160)
bits 4-6: acc_bwp          (always 2 = normal averaging)
bit  7:   acc_filter_perf  (1 for ODR >= 12.5 Hz, 0 for ODR < 12.5 Hz)
```
Note: bit 7 is **inverted** vs BMI160. BMI270 uses it as a high-performance filter
enable (not an under-sampling flag). Reference values:
```
0.78125 Hz -> 0x21    12.5 Hz -> 0xA5    200 Hz -> 0xA9
  1.5625 Hz -> 0x22    25   Hz -> 0xA6    400 Hz -> 0xAA
   3.125 Hz -> 0x23    50   Hz -> 0xA7    800 Hz -> 0xAB
    6.25 Hz -> 0x24   100   Hz -> 0xA8   1600 Hz -> 0xAC
```

`acc_range_byte` (BMI160 FSR bitmasks):
```
+/-2g  -> 0x03    scale = 16384 LSB/g
+/-4g  -> 0x05    scale = 8192  LSB/g
+/-8g  -> 0x08    scale = 4096  LSB/g
+/-16g -> 0x0C    scale = 2048  LSB/g
```
`acc_range_byte` (BMI270 FSR bitmasks):
```
+/-2g  -> 0x00    scale = 16384 LSB/g
+/-4g  -> 0x01    scale = 8192  LSB/g
+/-8g  -> 0x02    scale = 4096  LSB/g
+/-16g -> 0x03    scale = 2048  LSB/g
```

**Enable / Disable motion interrupt (BMI160):**
```
Enable:  [0x03, 0x09, enable_mask, 0x00]
Disable: [0x03, 0x09, 0x00, disable_mask]
```

**Enable / Disable tap detection (BMI160):**
```
Enable single/double: [0x03, 0x0C, mask, 0x00]
  mask bit 0 = double tap, mask bit 1 = single tap
Disable: [0x03, 0x0C, 0x00, 0x03]
```

**Enable / Disable orientation detection (BMI160):**
```
Enable:  [0x03, 0x0F, 0x01, 0x00]
Disable: [0x03, 0x0F, 0x00, 0x01]
```

**BMI160 Step detector enable/disable:**
```
Enable:  [0x03, 0x17, 0x01, 0x00]
Disable: [0x03, 0x17, 0x00, 0x01]
Reset:   [0x03, 0x1B]
```

**BMI270 Feature enable/disable (using FEATURE_ENABLE and FEATURE_INTERRUPT_ENABLE):**
```
Step counter enable:
  [0x03, 0x07, 0x02, 0x00]   <- interrupt enable
  [0x03, 0x06, 0x02, 0x00]   <- feature enable
Step counter disable:
  [0x03, 0x07, 0x00, 0x02]
  [0x03, 0x06, 0x00, 0x02]

Step detector enable:
  [0x03, 0x07, 0x80, 0x00]
  [0x03, 0x06, 0x80, 0x00]
Step detector disable:
  [0x03, 0x07, 0x00, 0x80]
  [0x03, 0x06, 0x00, 0x80]
```

**BMI270 Feature config (FEATURE_CONFIG = 0x08):**
```
[0x03, 0x08, feature_index, ...config_bytes...]
```
Feature indices used:
```
axis_remap  = 0
any_motion  = 1
no_motion   = 2
sig_motion  = 3
step_counter_0..3 = 4..7
wrist_gesture = 8
wrist_wakeup  = 9
```

### Notification Headers (what the board sends back)

| Register | Description |
|---|---|
| [0x03, 0x04] | Accelerometer XYZ data (no ID byte) |
| [0x03, 0x05] | BMI270 Packed accelerometer data |
| [0x03, 0x0B] | BMI160 Any/Slow/No-motion interrupt |
| [0x03, 0x09] | BMI270 Motion interrupt |
| [0x03, 0x0E] | BMI160 Tap interrupt |
| [0x03, 0x11] | BMI160 Orientation interrupt |
| [0x03, 0x19] | BMI160 Step detector |
| [0x03, 0x1C] | BMI160 Packed accelerometer data |
| [0x03, 0x0B] | BMI270 Step count interrupt |
| [0x03, 0x0A] | BMI270 Wrist gesture interrupt |
| [0x03, 0x0C] | BMI270 Activity interrupt |

### Response Parsing

**Accelerometer XYZ data** (6 bytes after header):
```kotlin
val x = (response[2] or (response[3].toInt() shl 8)).toShort() / scale
val y = (response[4] or (response[5].toInt() shl 8)).toShort() / scale
val z = (response[6] or (response[7].toInt() shl 8)).toShort() / scale
// 'scale' from FSR lookup table above
```

**Any-motion response** (1 byte, `response[2]`):
```
bit 3: x-axis active    (0x1 << (0+3))
bit 4: y-axis active    (0x1 << (1+3))
bit 5: z-axis active    (0x1 << (2+3))
bit 6: sign             (0 = positive, 1 = negative)
```

**Tap response** (1 byte, `response[2]`):
```
bit 0: single tap
bit 1: double tap
bit 5: sign (1 = positive)
```

**Orientation response** (1 byte, `response[2]`):
```
MblMwSensorOrientation = ((byte & 0x06) >> 1) + 4 * ((byte & 0x08) >> 3)
```

**BMI270 Gesture response** (1 byte, `response[2]`):
```
type          = byte & 0x03
gesture_code  = byte >> 2
```

**BMI270 Activity response** (1 byte, `response[2]`):
```
activity = byte >> 1
```

**Packed accelerometer data** (multiple 6-byte XYZ triplets starting at `response[2]`):
```
Each 6-byte block: int16 x, int16 y, int16 z (little-endian)
```

---

## Module 0x13 — Gyroscope

### Implementation Types
```
MBL_MW_MODULE_GYRO_TYPE_BMI160 = 0
MBL_MW_MODULE_GYRO_TYPE_BMI270 = 1
```

### BMI160 Register Opcodes
```
POWER_MODE             = 0x01
DATA_INTERRUPT_ENABLE  = 0x02
CONFIG                 = 0x03
DATA                   = 0x05
PACKED_GYRO_DATA       = 0x07
```

### BMI270 Register Opcodes
```
POWER_MODE             = 0x01
DATA_INTERRUPT_ENABLE  = 0x02
CONFIG                 = 0x03
DATA                   = 0x04
PACKED_GYRO_DATA       = 0x05
OFFSET                 = 0x06
```

### Config Struct (`GyroBoschConfig`, 2 bytes)
```
byte 0:
  bits 0-3: gyr_odr  (use MblMwGyroBoschOdr enum value directly)
  bits 4-5: gyr_bwp  (2 = normal)
  bits 6-7: unused
byte 1:
  bits 0-2: gyr_range (0=2000dps, 1=1000dps, 2=500dps, 3=250dps, 4=125dps)
  bits 3-7: unused
```

FSR scales (index = `gyr_range`):
```
0 -> 16.4   (2000 dps, 1 LSB = 1/16.4 dps)
1 -> 32.8   (1000 dps)
2 -> 65.6   (500 dps)
3 -> 131.2  (250 dps)
4 -> 262.4  (125 dps)
```

### Commands

**Start / Stop:**
```
Start BMI160: [0x13, 0x01, 0x01]
Stop  BMI160: [0x13, 0x01, 0x00]
Start BMI270: [0x13, 0x01, 0x01]
Stop  BMI270: [0x13, 0x01, 0x00]
```

**Enable / Disable data stream:**
```
Enable BMI160:  [0x13, 0x02, 0x01, 0x00]
Disable BMI160: [0x13, 0x02, 0x00, 0x01]
Enable BMI270:  [0x13, 0x02, 0x01, 0x00]
Disable BMI270: [0x13, 0x02, 0x00, 0x01]
```

**Write config (BMI160):**
```
[0x13, 0x03, gyr_odr_bwp_byte, gyr_range_byte]
```

**Write config (BMI270):**
```
[0x13, 0x03, gyr_odr_bwp_byte, gyr_range_byte]
```

**Write offsets (BMI270):**
```
[0x13, 0x06, x_offset, y_offset, z_offset]
```

### Notification Headers

| Register | Description |
|---|---|
| [0x13, 0x05] | BMI160 Rotation XYZ data |
| [0x13, 0x07] | BMI160 Packed rotation data |
| [0x13, 0x04] | BMI270 Rotation XYZ data |
| [0x13, 0x05] | BMI270 Packed rotation data |

### Response Parsing

**Rotation XYZ** (6 bytes after header, same as accelerometer):
```kotlin
val x = (response[2] or (response[3].toInt() shl 8)).toShort() / scale
val y = (response[4] or (response[5].toInt() shl 8)).toShort() / scale
val z = (response[6] or (response[7].toInt() shl 8)).toShort() / scale
```

---

## Module 0x15 — Magnetometer (BMM150)

### Register Opcodes
```
POWER_MODE             = 0x01
DATA_INTERRUPT_ENABLE  = 0x02
DATA_RATE              = 0x03
DATA_REPETITIONS       = 0x04
MAG_DATA               = 0x05
PACKED_MAG_DATA        = 0x09
```

Packed mag data available when module revision >= 1.
Suspend mode available when module revision >= 2.

### Commands

**Power / start / stop / suspend:**
```
Start:   [0x15, 0x01, 0x01]
Stop:    [0x15, 0x01, 0x00]
Suspend: [0x15, 0x01, 0x02]   <- only if revision >= 2
```

**Enable / Disable data stream:**
```
Enable:  [0x15, 0x02, 0x01, 0x00]
Disable: [0x15, 0x02, 0x00, 0x01]
```

**Configure (XY reps, Z reps, ODR):**
```
[0x15, 0x04, (xy_reps - 1) / 2, z_reps - 1]
[0x15, 0x03, odr_byte]
```

Presets:
```
LOW_POWER:         xy=3,  z=3,  ODR=10Hz
REGULAR:           xy=9,  z=15, ODR=10Hz
ENHANCED_REGULAR:  xy=15, z=27, ODR=10Hz
HIGH_ACCURACY:     xy=47, z=83, ODR=20Hz
```

### Response Parsing

**Mag data** (6 bytes after header):
```kotlin
val x = (response[2] or (response[3].toInt() shl 8)).toShort() / 16.0f  // uT
val y = (response[4] or (response[5].toInt() shl 8)).toShort() / 16.0f
val z = (response[6] or (response[7].toInt() shl 8)).toShort() / 16.0f
```
BMM150_SCALE = 16.0f

---

## Module 0x12 — Barometer (BMP280 / BME280)

### Implementation Types
```
MBL_MW_MODULE_BARO_TYPE_BMP280 = 0
MBL_MW_MODULE_BARO_TYPE_BME280 = 1
```

### Register Opcodes
```
PRESSURE  = 0x01
ALTITUDE  = 0x02
CONFIG    = 0x03
CYCLIC    = 0x04
```

### Config Struct (`BoschBaroConfig`, 2 bytes)
```
byte 0:
  bits 0-1: unused
  bits 2-4: pressure_oversampling
  bits 5-7: temperature_oversampling
byte 1:
  bits 0-1: unused
  bits 2-4: iir_filter
  bits 5-7: standby_time
```

### Commands

**Start / Stop cyclic measurement:**
```
Start: [0x12, 0x04, 0x01, 0x01]
Stop:  [0x12, 0x04, 0x00, 0x00]
```

**Write config:**
```
[0x12, 0x03, config_byte_0, config_byte_1]
```

### Response Parsing

**Pressure** (4 bytes, unsigned, `response[2..5]`):
```kotlin
val rawPressure = (response[2].toLong() and 0xFF) or
                  ((response[3].toLong() and 0xFF) shl 8) or
                  ((response[4].toLong() and 0xFF) shl 16) or
                  ((response[5].toLong() and 0xFF) shl 24)
val pressurePa = rawPressure / 256.0f   // BOSCH_BARO_SCALE = 256.0
```

**Altitude** (4 bytes, signed):
```kotlin
val rawAlt = ByteBuffer.wrap(response, 2, 4).order(ByteOrder.LITTLE_ENDIAN).int
val altitudeM = rawAlt / 256.0f
```

---

## Module 0x19 — Sensor Fusion

### Register Opcodes
```
ENABLE             = 0x01
MODE               = 0x02
OUTPUT_ENABLE      = 0x03
CORRECTED_ACC      = 0x04
CORRECTED_GYRO     = 0x05
CORRECTED_MAG      = 0x06
QUATERNION         = 0x07
EULER_ANGLES       = 0x08
GRAVITY_VECTOR     = 0x09
LINEAR_ACC         = 0x0A
CALIBRATION_STATE  = 0x0B
ACC_CAL_DATA       = 0x0C
GYRO_CAL_DATA      = 0x0D
MAG_CAL_DATA       = 0x0E
RESET_ORIENTATION  = 0x0F
```

### Config Struct (`SensorFusionState.config`, 2 bytes)
```
byte 0: mode      (MblMwSensorFusionMode: 0=SLEEP, 1=NDOF, 2=IMU_PLUS, 3=COMPASS, 4=M4G)
byte 1:
  bits 0-3: acc_range  (0=2g, 1=4g, 2=8g, 3=16g)
  bits 4-7: gyro_range (enum+1: 1=2000dps, 2=1000dps, 3=500dps, 4=250dps, 5=125dps)
```

### Enable mask bits
```
bit 0: CORRECTED_ACC
bit 1: CORRECTED_GYRO
bit 2: CORRECTED_MAG
bit 3: QUATERNION
bit 4: EULER_ANGLES
bit 5: GRAVITY_VECTOR
bit 6: LINEAR_ACC
```

### Underlying-sensor requirements per mode

The fusion algorithm is fed by the on-board acc / gyro / mag modules. Each mode dictates which sensors must be running and at what rate; the host must configure and start each underlying sensor in addition to the fusion module itself.

| Mode      | Acc                | Gyro          | Mag             |
|-----------|--------------------|---------------|-----------------|
| `SLEEP`   | —                  | —             | —               |
| `NDOF`    | 100 Hz, host range | 100 Hz, host range | 25 Hz, xy=9, z=15 |
| `IMU_PLUS`| 100 Hz, host range | 100 Hz, host range | —               |
| `COMPASS` | 25 Hz,  host range | —             | 25 Hz, xy=9, z=15 |
| `M4G`     | 50 Hz,  host range | —             | 25 Hz, xy=9, z=15 |

Mag preset is fixed by the firmware: `xy_reps = 9`, `z_reps = 15`, ODR = 25 Hz → `[0x15, 0x04, 0x04, 0x0E]` followed by `[0x15, 0x03, 0x06]`.

### Commands

The lifecycle for configuring, starting, and stopping sensor fusion follows this sequence of BLE commands:

**Write fusion config:**
```
[0x19, 0x02, mode_byte, range_byte]
```
where `range_byte = acc_range | ((gyro_range + 1) << 4)`.

**Enable output mask:**
```
[0x19, 0x03, enable_mask, 0x00]
```
`enable_mask` is the per-signal bit (see *Enable mask bits* above). Multiple bits may be set to subscribe to several outputs from the same fusion run.

**Start / Stop the fusion algorithm:**
```
Start: [0x19, 0x01, 0x01]
Stop:  [0x19, 0x01, 0x00]
```

**Clear output mask** (issued during stop, before stopping the underlying sensors):
```
[0x19, 0x03, 0x00, 0x7F]
```

#### Full configure sequence (NDOF, BMI160, ±2g / ±2000 dps shown)

```
[0x19, 0x02, 0x01, 0x10]   # fusion mode = NDOF, ranges packed
[0x03, 0x03, 0x28, 0x03]   # acc 100 Hz, ±2g (BMI160: conf=0x28, range bitmask=0x03)
[0x13, 0x03, 0x28, 0x00]   # gyro 100 Hz, ±2000 dps
[0x15, 0x04, 0x04, 0x0E]   # mag repetitions: xy=9, z=15
[0x15, 0x03, 0x06]         # mag ODR = 25 Hz
```

For **BMI270** the acc command differs: `[0x03, 0x03, 0xA8, 0x00]` (filter_perf bit 7 set, 0-based range).

For `IMU_PLUS` the two mag commands are omitted. For `COMPASS` and `M4G` the gyro command is omitted (and the acc ODR is 25 Hz / 50 Hz respectively → conf = `0x26` / `0x27` on BMI160, `0xA6` / `0xA7` on BMI270).

#### Full start sequence (NDOF + quaternion shown)

```
# Underlying sensors first — interrupt enables, then start
[0x03, 0x02, 0x01, 0x00]   # acc data interrupt enable
[0x13, 0x02, 0x01, 0x00]   # gyro data interrupt enable
[0x15, 0x02, 0x01, 0x00]   # mag data interrupt enable
[0x03, 0x01, 0x01]         # acc start
[0x13, 0x01, 0x01]         # gyro start
[0x15, 0x01, 0x01]         # mag start

# Fusion last — enable mask then start the algorithm
[0x19, 0x03, 0x08, 0x00]   # output_enable: bit 3 = QUATERNION
[0x19, 0x01, 0x01]         # fusion start
```

`IMU_PLUS` skips the mag enable + mag start. `COMPASS` and `M4G` skip the gyro enable + gyro start.

#### Full stop sequence (NDOF shown)

Reverses the start: fusion off + mask cleared first, then underlying sensors are stopped + interrupts disabled.

```
[0x19, 0x01, 0x00]         # fusion stop
[0x19, 0x03, 0x00, 0x7F]   # output_enable: clear all bits

[0x03, 0x01, 0x00]         # acc stop
[0x13, 0x01, 0x00]         # gyro stop
[0x15, 0x01, 0x00]         # mag stop

[0x03, 0x02, 0x00, 0x01]   # acc data interrupt disable
[0x13, 0x02, 0x00, 0x01]   # gyro data interrupt disable
[0x15, 0x02, 0x00, 0x01]   # mag data interrupt disable
```

A common host bug is to send only the fusion config + start (`[0x19, 0x02, …]` and `[0x19, 0x01, 0x01]`) without configuring or starting the underlying acc / gyro / mag — the fusion algorithm runs but produces no output because it never sees any input samples.

**Read config:**
```
[0x19, 0x82]   <- READ_REGISTER(MODE) = 0x82
```

**Read calibration state:**
```
[0x19, 0x8B]   <- READ_REGISTER(CALIBRATION_STATE) = 0x8B
```

**Read calibration data (chain of 3 reads):**
```
[0x19, 0x8C]   <- READ_REGISTER(ACC_CAL_DATA)
Response -> [0x19, 0x8C, <10 bytes acc cal>]
Then send:
[0x19, 0x8D]   <- READ_REGISTER(GYRO_CAL_DATA)
Response -> [0x19, 0x8D, <10 bytes gyro cal>]
Then send:
[0x19, 0x8E]   <- READ_REGISTER(MAG_CAL_DATA)
Response -> [0x19, 0x8E, <10 bytes mag cal>]
```

### Response Parsing

**Corrected Accelerometer** (13 bytes after header):
```kotlin
// First 12 bytes = 3x float32 (x, y, z in m/s^2), last byte = accuracy
val x = ByteBuffer.wrap(response, 2, 4).order(LITTLE_ENDIAN).float / 1000f  // SENSOR_FUSION_ACC_SCALE
val y = ByteBuffer.wrap(response, 6, 4).order(LITTLE_ENDIAN).float / 1000f
val z = ByteBuffer.wrap(response, 10, 4).order(LITTLE_ENDIAN).float / 1000f
val accuracy = response[14]
```

**Corrected Gyro / Corrected Mag** (13 bytes after header):
```kotlin
// 3x float32, same layout; no divide (direct float), last byte = accuracy
val x = ByteBuffer.wrap(response, 2, 4).order(LITTLE_ENDIAN).float
val y = ByteBuffer.wrap(response, 6, 4).order(LITTLE_ENDIAN).float
val z = ByteBuffer.wrap(response, 10, 4).order(LITTLE_ENDIAN).float
val accuracy = response[14]
```

**Quaternion** (16 bytes after header = 4x float32):
```kotlin
val w = ByteBuffer.wrap(response, 2, 4).order(LITTLE_ENDIAN).float
val x = ByteBuffer.wrap(response, 6, 4).order(LITTLE_ENDIAN).float
val y = ByteBuffer.wrap(response, 10, 4).order(LITTLE_ENDIAN).float
val z = ByteBuffer.wrap(response, 14, 4).order(LITTLE_ENDIAN).float
```

**Euler Angles** (16 bytes = 4x float32):
```kotlin
val heading = ByteBuffer.wrap(response, 2, 4).order(LITTLE_ENDIAN).float
val pitch   = ByteBuffer.wrap(response, 6, 4).order(LITTLE_ENDIAN).float
val roll    = ByteBuffer.wrap(response, 10, 4).order(LITTLE_ENDIAN).float
val yaw     = ByteBuffer.wrap(response, 14, 4).order(LITTLE_ENDIAN).float
```

**Gravity Vector / Linear Acceleration** (12 bytes = 3x float32):
```kotlin
val x = ByteBuffer.wrap(response, 2, 4).order(LITTLE_ENDIAN).float / 9.80665f  // MSS_TO_G_SCALE
val y = ByteBuffer.wrap(response, 6, 4).order(LITTLE_ENDIAN).float / 9.80665f
val z = ByteBuffer.wrap(response, 10, 4).order(LITTLE_ENDIAN).float / 9.80665f
```

**Calibration state** (3 bytes after header):
```kotlin
val acc_cal  = response[2]
val gyro_cal = response[3]
val mag_cal  = response[4]
// Values 0-3 (0=uncalibrated, 3=fully calibrated)
```

---

## Module 0x04 — Temperature

Multichannel temperature. Read via the data signal mechanism. Temperature values are
signed int32 divided by 8.0 (TEMPERATURE_SCALE = 8.0f).

---

## Module 0x0B — Logging

### Register Opcodes
```
ENABLE                  = 0x01
TRIGGER                 = 0x02
REMOVE                  = 0x03
TIME                    = 0x04
LENGTH                  = 0x05
READOUT                 = 0x06
READOUT_NOTIFY          = 0x07
READOUT_PROGRESS        = 0x08
REMOVE_ENTRIES          = 0x09
REMOVE_ALL              = 0x0A
CIRCULAR_BUFFER         = 0x0B
READOUT_PAGE_COMPLETED  = 0x0D
READOUT_PAGE_CONFIRM    = 0x0E
PAGE_FLUSH              = 0x10
```

Revisions:
```
REVISION_EXTENDED_LOGGING = 2
MMS_REVISION              = 3
```

### Tick-to-ms Conversion
```
TICK_TIME_STEP = (48.0 / 32768.0) * 1000.0 = 1.46484375 ms/tick
```

### Key Constants
```
ENTRY_ID_MASK = 0x1F   (lower 5 bits of byte)
RESET_UID_MASK = 0x07  (next 3 bits: bits 5-7)
LOG_ENTRY_SIZE = 8 bytes total (1 id/reset + 3 tick + 4 data)
LOG_ENTRY_DATA_SIZE = 4 bytes (uint32_t payload per entry)
```

### Commands

**Create a logger for a signal:**
```
[0x0B, 0x02, module_id, register_id, data_id, (offset<<5 | length-1)]
```
Response: `[0x0B, 0x02, assigned_entry_id]`

**Start logging (with optional overwrite):**
```
[0x0B, 0x0B, overwrite]   <- set circular buffer
[0x0B, 0x01, 0x01]        <- enable logging
```

**Stop logging:**
```
[0x0B, 0x01, 0x00]
```

**Read time signal (get reference epoch):**
```
[0x0B, 0x84]   <- READ_REGISTER(TIME) = 0x84
```
Response: `[0x0B, 0x84, tick_byte0, tick_byte1, tick_byte2, tick_byte3, reset_uid]`
```kotlin
val tick = ByteBuffer.wrap(response, 2, 4).order(LITTLE_ENDIAN).int.toLong() and 0xFFFFFFFFL
val epoch = System.currentTimeMillis() - (tick * TICK_TIME_STEP).toLong()
val resetUid = response[6]
```

**Download sequence:**
1. Enable readout notify: `[0x0B, 0x07, 0x01]`
2. For extended logging (revision 2): `[0x0B, 0x0D, 0x01]`
3. Enable progress: `[0x0B, 0x08, 0x01]`
4. Read length: `[0x0B, 0x85]` (READ_REGISTER(LENGTH))
5. On length response, send readout: `[0x0B, 0x06, n_entries(4 bytes LE), n_notify(4 bytes LE)]`
6. For each page-completed notification, confirm with: `[0x0B, 0x0E]`

**Clear log entries:**
```
[0x0B, 0x09, 0xFF, 0xFF, 0xFF, 0xFF]
```

**Flush page (MMS only, revision 3):**
```
[0x0B, 0x10, 0x01]
```

### Log Entry Format (from READOUT_NOTIFY = 0x07)
Each notification packet is `[0x0B, 0x07, entry...]` and carries 1 or 2 log
entries of **8 bytes each** (so payload = 8 or 16 bytes, packet = 10 or 18 bytes
including the 2-byte header):
```
Entry at offset 2 (always present):
  byte[offset+0]:    (reset_uid << 5) | entry_id     (reset_uid: bits 5-7, entry_id: bits 0-4)
  byte[offset+1..3]: uint24 tick     (little-endian, 3 bytes)
  byte[offset+4..7]: uint32 data     (little-endian, 4 bytes)

Entry at offset 10 (present if packet length == 18):
  same format as above
```

Wall-clock time of an entry: `logReferenceDate + tick * TICK_TIME_STEP`.

To convert to a signal value, reassemble 4-byte chunks from consecutive entry IDs.

---

## Module 0x0C — Timer

### Register Opcodes
```
ENABLE        = 0x01
TIMER_ENTRY   = 0x02
START         = 0x03
STOP          = 0x04
REMOVE        = 0x05
NOTIFY        = 0x06
NOTIFY_ENABLE = 0x07
```

### Commands

**Create timer:**
```
[0x0C, 0x02, period(4 bytes LE), repetitions(2 bytes LE), immediate_flag]
```
- `period` in milliseconds
- `repetitions` = 0xFFFF for indefinite
- `immediate_flag` = 1 for immediate first fire, 0 for delayed

Response: `[0x0C, 0x02, timer_id]`

**Start / Stop / Remove:**
```
Start:  [0x0C, 0x03, timer_id]
Stop:   [0x0C, 0x04, timer_id]
Remove: [0x0C, 0x05, timer_id]
```

**Timer fires notification:** `[0x0C, 0x06, timer_id]`

---

## Module 0x0A — Event

### Register Opcodes
```
ENTRY          = 0x02
CMD_PARAMETERS = 0x03
REMOVE         = 0x04
REMOVE_ALL     = 0x05
```

### Event Entry Command Format
Events bind a source signal to a command that fires when the signal fires.

```
[0x0A, 0x02, src_module_id, src_register_id, src_data_id, dst_module_id, dst_register_id, param_length]
```
Optionally followed by a data token:
```
[data_length_and_offset_byte, dest_offset_byte]
  where byte0 = 0x01 | (data_length << 1) | (data_offset << 4)
```

Then parameters:
```
[0x0A, 0x03, ...param_bytes...]
```

**Remove specific event commands:**
```
[0x0A, 0x04, command_id]
```

**Remove all events:**
```
[0x0A, 0x05]
```

---

## Module 0x09 — Data Processor

### Register Opcodes
```
ADD            = 0x02
NOTIFY         = 0x03
STATE          = 0x04
PARAMETER      = 0x05
REMOVE         = 0x06
NOTIFY_ENABLE  = 0x07
REMOVE_ALL     = 0x08
```

### Overview

The data processor chains on-device signal transformations. Processors are created one at a time;
each ADD response assigns an ID that can be used as the source for subsequent processors.

### ADD command format

```
[0x09, 0x02, src_module, src_reg, src_data_id, src_config, proc_type, config_bytes...]
```

| Byte | Field | Notes |
|------|-------|-------|
| 0 | module | 0x09 |
| 1 | register | 0x02 (ADD) |
| 2 | src_module | Source module ID |
| 3 | src_reg | Source register ID (not OR'd with 0x80) |
| 4 | src_data_id | Source data ID, or 0xFF for "any" |
| 5 | src_config | Encodes sample length and offset (see below) |
| 6 | proc_type | Processor type ID |
| 7+ | config_bytes | Per-processor config (see Processor Types) |

**Response** (plain notification on (0x09, 0x02)):
```
[0x09, 0x02, assigned_proc_id]
```
Note: this is a plain notification, not a read-response (bit 7 is NOT set).

### Source config byte formula

```
src_config = ((n_channels * channel_size - 1) << 5) | offset
```

This encodes the total sample length minus 1 in the upper 3 bits, and the byte offset within
the sample in the lower 5 bits.

**Common source signals:**

| Signal | Module | Reg | ID | Channels | Ch size | src_config |
|--------|--------|-----|----|----------|---------|------------|
| Switch | 0x01 | 0x01 | 0xFF | 1 | 1B | 0x00 |
| GPIO ADC | 0x05 | 0x07 | pin | 1 | 2B | 0x20 |
| GPIO absolute | 0x05 | 0x06 | pin | 1 | 2B | 0x20 |
| Accelerometer | 0x03 | 0x04 | 0xFF | 3 | 2B | 0xA0 |
| Gyroscope | 0x13 | 0x05 | 0xFF | 3 | 2B | 0xA0 |
| Temperature | 0x04 | 0xC1 | ch | 1 | 2B | 0x20 |
| Processor output | 0x09 | 0x03 | proc_id | varies | varies | computed |

### Processor streaming

**Enable notifications:**
```
[0x09, 0x07, proc_id, 0x01]
```

**Disable notifications:**
```
[0x09, 0x07, proc_id, 0x00]
```

**Data notification format:**
```
[0x09, 0x03, proc_id, data_bytes...]
```
Multiple processors all share the same (0x09, 0x03) notification; demultiplex by proc_id at byte[2].

### Remove processors

**Remove one:**
```
[0x09, 0x06, proc_id]
```

**Remove all:**
```
[0x09, 0x08]
```

---

### Processor Types

#### 0x01 — Passthrough

Gates data flow.

Config bytes (3): `[mode, count_lo, count_hi]`

| Mode | Value |
|------|-------|
| ALL | 0 |
| CONDITIONAL | 1 |
| COUNT | 2 |

#### 0x02 — Accumulator / Counter

Config byte (1): `{output_size-1 : 2, input_size-1 : 2, mode : 3}`

| mode | Meaning |
|------|---------|
| 0 | Accumulate (SUM) |
| 1 | Count events |

For Counter, input_size field is 0 (ignored). Output is always 1 channel.

**Reference test (test_led_controller step 1, Counter outputSize=1):**
```
[0x09, 0x02, 0x01, 0x01, 0xFF, 0x00, 0x02, 0x10]
```
Config byte 0x10 = `(0 & 0x3) | (1 << 4)` — outputSize=1, mode=COUNT.

#### 0x03 — Average (Low-pass filter)

Config bytes (2): `[byte0, sample_size]`

`byte0 = (output_unit-1 & 0x3) | ((input_unit-1 & 0x3) << 2)`  (output == input size, mode=0=LPF)

**Reference test (test_freefall step 2, Average of 2-byte RSS output, sampleSize=4):**
```
[0x09, 0x02, 0x09, 0x03, 0x00, 0x20, 0x03, 0x05, 0x04]
```
Config bytes `[0x05, 0x04]` — unit=2, s=1 → 1|(1<<2)=0x05; sample_size=4.

#### 0x06 — Comparator

Config bytes (7): `[is_signed, operation, padding, ref_b0, ref_b1, ref_b2, ref_b3]`

Reference is a signed Int32 in little-endian byte order.

| Operation | Value |
|-----------|-------|
| EQ | 0 |
| NEQ | 1 |
| LT | 2 |
| LTE | 3 |
| GT | 4 |
| GTE | 5 |

**Reference test (test_freefall step 4, EQ -1 signed):**
```
config: [0x01, 0x00, 0x00, 0xFF, 0xFF, 0xFF, 0xFF]
```

#### 0x07 — RMS / RSS Combiner

Reduces a multi-axis signal to a scalar magnitude.

Config bytes (2): `[byte0, mode]`

`byte0 = (unit-1 & 0x3) | ((unit-1 & 0x3) << 2) | ((channels-1 & 0x7) << 4) | (is_signed << 7)`

| Mode | Value |
|------|-------|
| RMS | 0 |
| RSS | 1 |

Output: 1 channel, same byte width as one input channel, unsigned.

**Reference test (test_freefall step 1, RSS of accelerometer 3ch×2B signed):**
```
[0x09, 0x02, 0x03, 0x04, 0xFF, 0xA0, 0x07, 0xA5, 0x01]
```
Config bytes `[0xA5, 0x01]` — unit=2, s=1, ch=3, signed: `1|(1<<2)|(2<<4)|0x80 = 0xA5`, mode=RSS=1.

#### 0x08 — Time Delay

Passes one sample per period.

Config bytes (5): `[byte0, period_b0, period_b1, period_b2, period_b3]`

`byte0 = ((data_length-1) & 0x7) | ((mode & 0x7) << 3)`

Period is in milliseconds, little-endian UInt32.

| Mode | Value |
|------|-------|
| ABSOLUTE | 0 |
| DIFFERENTIAL | 1 |

#### 0x09 — Math

Arithmetic transform applied per sample.

Config bytes (7): `[byte0, operation, rhs_b0, rhs_b1, rhs_b2, rhs_b3, n_channels]`

`byte0 = (output_unit-1 & 0x3) | ((input_unit-1 & 0x3) << 2) | (is_signed << 4)`

`n_channels = inputChannels - 1` when multichannel, else 0.

| Operation | Value |
|-----------|-------|
| ADD | 0 |
| SUBTRACT | 1 |
| MULTIPLY | 2 |
| DIVIDE | 3 |
| MODULO | 4 |
| EXPONENT | 5 |
| SQRT | 6 |
| LSHIFT | 7 |
| RSHIFT | 8 |
| ABS | 9 |
| CONSTANT | 10 |
| NEGATE | 11 |
| FLOOR | 12 |
| CEIL | 13 |
| ROUND | 14 |

**Reference test (test_led_controller step 2, counter % 2, unsigned, output=4):**
```
config: [0x03, 0x04, 0x02, 0x00, 0x00, 0x00, 0x00]
```
byte0 = `(4-1 & 0x3) | ((1-1 & 0x3) << 2) | (0 << 4) = 0x03`; op=MODULO=4; rhs=2 LE32; nch=0.

#### 0x0A — Sample Delay

Buffers N samples before emitting.

Config bytes (2): `[data_length - 1, bin_size]`

#### 0x0D — Threshold

Emits a value when the input crosses a boundary.

Config bytes (7): `[byte0, boundary_b0, boundary_b1, boundary_b2, boundary_b3, hyst_b0, hyst_b1]`

`byte0 = (unit_size-1 & 0x3) | (is_signed << 2) | ((mode & 0x7) << 3)`

Boundary is a signed Int32 in little-endian. Hysteresis is an unsigned UInt16 in little-endian.

| Mode | Value | Output |
|------|-------|--------|
| ABSOLUTE | 0 | raw value (only when crossing) |
| BINARY | 1 | Int32: +1 when above, –1 when below |

**Reference test (test_freefall step 3, BINARY boundary=8192 unsigned):**
```
config: [0x09, 0x00, 0x20, 0x00, 0x00, 0x00, 0x00]
```
byte0 = `(2-1 & 0x3) | (0 << 2) | (1 << 3) = 0x09`; boundary=8192=0x2000 LE32; hyst=0.

---

## Module 0x0F — Macro

### Register Opcodes
```
BEGIN       = 0x02
ADD_COMMAND = 0x03
END         = 0x04
EXECUTE     = 0x05
ERASE_ALL   = 0x08
ADD_PARTIAL = 0x09
```

### Protocol

Commands are written with response (unlike all others).

**Begin macro recording:**
```
[0x0F, 0x02, exec_on_boot]   <- exec_on_boot: 1=run on boot, 0=manual only
```
Response: `[0x0F, 0x02, macro_id]`

**Add command to macro:**
For commands <= 13 bytes (MW_CMD_MAX_LENGTH - 2):
```
[0x0F, 0x03, ...command_bytes...]
```

For commands >= 14 bytes:
```
[0x0F, 0x09, cmd_byte0, cmd_byte1]       <- ADD_PARTIAL (first 2 bytes)
[0x0F, 0x03, cmd_byte2..cmd_byteN-2]     <- ADD_COMMAND (remaining)
```

**End macro:**
```
[0x0F, 0x04]
```

**Execute macro:**
```
[0x0F, 0x05, macro_id]
```

**Erase all macros:**
```
[0x0F, 0x08]
```

---

## Module 0x11 — Settings

### Register Opcodes
```
DEVICE_NAME             = 0x01
AD_INTERVAL             = 0x02
TX_POWER                = 0x03
START_ADVERTISING       = 0x05
SCAN_RESPONSE           = 0x07
PARTIAL_SCAN_RESPONSE   = 0x08
CONNECTION_PARAMS       = 0x09
DISCONNECT_EVENT        = 0x0A
MAC                     = 0x0B
BATTERY_STATE           = 0x0C
POWER_STATUS            = 0x11
CHARGE_STATUS           = 0x12
WHITELIST_FILTER_MODE   = 0x13
WHITELIST_ADDRESSES     = 0x14
THREE_VOLT_POWER        = 0x1C
FORCE_1M_PHY            = 0x1D
```

Revisions:
```
CONN_PARAMS_REVISION      = 1
DISCONNECTED_EVENT_REVISION = 2
BATTERY_REVISION          = 3
CHARGE_STATUS_REVISION    = 5
WHITELIST_REVISION        = 6
MMS_REVISION              = 9
MMS_PHY_REVISION          = 10
```

### Commands

**Set device name:**
```
[0x11, 0x01, ...ascii_bytes...]
```
Constraints (enforced by firmware for BLE advertising):
- Maximum **26 ASCII bytes** (not UTF-8). Longer strings are silently truncated by the board.
- Allowed characters: `A-Z`, `a-z`, `0-9`, `_`, `-`, and space. Writing other code points will be rejected or mangled by the advertising layer.

**Set advertising interval:**
```
[0x11, 0x02, interval_low, interval_high, timeout]
```
If revision >= 1: interval is divided by 0.625 (AD_INTERVAL_STEP) before encoding.
If revision >= 6 (WHITELIST): append `ad_type` byte.

**Set TX power:**
```
[0x11, 0x03, tx_power_signed_byte]
```

**Start advertising:**
```
[0x11, 0x05]
```

**Set connection parameters:**
```
[0x11, 0x09, min_ci_lo, min_ci_hi, max_ci_lo, max_ci_hi, latency_lo, latency_hi, timeout_lo, timeout_hi]
```
All values divided by CONN_INTERVAL_STEP (1.25) or TIMEOUT_STEP (10) respectively.

**Read battery:**
```
[0x11, 0x8C]   <- READ_REGISTER(BATTERY_STATE)
```
Response: `[0x11, 0x8C, charge_percent, voltage_lo, voltage_hi]`
```kotlin
val charge  = response[2]
val voltage = ((response[3].toInt() and 0xFF) or ((response[4].toInt() and 0xFF) shl 8)).toFloat()
```

**Read MAC address:**
```
[0x11, 0x8B]   <- READ_REGISTER(MAC)
```
Response: `[0x11, 0x8B, b0, b1, b2, b3, b4, b5]`
Format: `"%02X:%02X:%02X:%02X:%02X:%02X", b5, b4, b3, b2, b1, b0`

**Read power/charge status:**
```
[0x11, 0x91]   <- READ_REGISTER(POWER_STATUS)
[0x11, 0x92]   <- READ_REGISTER(CHARGE_STATUS)
```
Response byte[2] contains the status value.

**3V regulator (MMS only, revision 9):**
```
[0x11, 0x1C, enable_byte]
```

**Force 1M PHY (MMS only, revision 10):**
```
[0x11, 0x1D, enable_byte]
```

---

## Module 0x05 — GPIO

### Register Opcodes
```
SET_DO                  = 0x01
CLEAR_DO                = 0x02
PULL_UP_DI              = 0x03
PULL_DOWN_DI            = 0x04
NO_PULL_DI              = 0x05
READ_AI_ABS_REF         = 0x06
READ_AI_ADC             = 0x07
READ_DI                 = 0x08
PIN_CHANGE              = 0x09
PIN_CHANGE_NOTIFY       = 0x0A
PIN_CHANGE_NOTIFY_ENABLE= 0x0B
```

### Commands
```
Set digital output high:  [0x05, 0x01, pin]
Set digital output low:   [0x05, 0x02, pin]
Enable pull-up on DI:     [0x05, 0x03, pin]
Enable pull-down on DI:   [0x05, 0x04, pin]
No pull on DI:            [0x05, 0x05, pin]
Read analog (abs ref):    [0x05, 0x86, pin]   <- READ_REGISTER(0x06)
Read analog (ADC):        [0x05, 0x87, pin]   <- READ_REGISTER(0x07)
Read digital input:       [0x05, 0x88, pin]   <- READ_REGISTER(0x08)
Configure pin change:     [0x05, 0x09, pin, change_type]
Enable pin change notify: [0x05, 0x0B, pin, 0x01]
Disable pin change notify:[0x05, 0x0B, pin, 0x00]
```

---

## Module 0x07 — iBeacon

### Register Opcodes
```
ENABLE    = 0x01
UUID      = 0x02
MAJOR     = 0x03
MINOR     = 0x04
RX_POWER  = 0x05
TX_POWER  = 0x06
PERIOD    = 0x07
```

### Commands
```
Enable iBeacon:          [0x07, 0x01, 0x01]
Disable iBeacon:         [0x07, 0x01, 0x00]
Set UUID (16 bytes BE):  [0x07, 0x02, b0, b1, ..., b15]
Set major (UInt16 LE):   [0x07, 0x03, major_lo, major_hi]
Set minor (UInt16 LE):   [0x07, 0x04, minor_lo, minor_hi]
Set RX power (Int8):     [0x07, 0x05, power_byte]
  RX power = signal strength at 1m, broadcast in advert for ranging.
  Typical: –55 dBm → 0xC9.
Set TX power (Int8):     [0x07, 0x06, power_byte]
  TX power = actual BLE transmission power.
  Typical: –4 dBm → 0xFC, 0 dBm → 0x00.
Set period (UInt16 LE):  [0x07, 0x07, period_lo, period_hi]
  Period is in milliseconds. Typical: 700 ms → [0xBC, 0x02].
```

### Reference test vectors
```
Enable:       [0x07, 0x01, 0x01]
Disable:      [0x07, 0x01, 0x00]
SetMajor(78): [0x07, 0x03, 0x4E, 0x00]
SetMinor(0x1D1D): [0x07, 0x04, 0x1D, 0x1D]
SetRXPower(-55):  [0x07, 0x05, 0xC9]
SetTXPower(-12):  [0x07, 0x06, 0xF4]
SetPeriod(0x3AB3):[0x07, 0x07, 0xB3, 0x3A]
```

---

## Module 0x08 — Haptic

### Register Opcodes
```
PULSE = 0x01
```

### Command
```
[0x08, 0x01, duty_cycle_byte, pulse_width_lo, pulse_width_hi, mode]
```

| Field | Description |
|---|---|
| `duty_cycle_byte` | Motor: `floor(dutyCycle% × 248 / 100)`, clamped to 0–248. Buzzer: always `0x7F`. |
| `pulse_width_lo/hi` | Pulse duration in milliseconds, UInt16 little-endian. |
| `mode` | `0x00` = ERM haptic motor, `0x01` = piezo buzzer. |

### Reference test vectors
```
Motor 100%, 5000 ms: [0x08, 0x01, 0xF8, 0x88, 0x13, 0x00]
  0xF8 = 248 (100% duty cycle), 0x1388 = 5000 ms, mode=0x00
Buzzer, 7500 ms:     [0x08, 0x01, 0x7F, 0x4C, 0x1D, 0x01]
  0x7F always for buzzer, 0x1D4C = 7500 ms, mode=0x01
```

---

## Module 0x0D — Serial Passthrough (I2C / SPI)

### Register Opcodes
```
I2C_READ_WRITE = 0x01
SPI_READ_WRITE = 0x02
```

### I2C Write
```
[0x0D, 0x01, device_addr, reg_addr, data_len, id, data...]
```

| Field | Description |
|---|---|
| `device_addr` | 7-bit I2C address of the peripheral. |
| `reg_addr` | Register (sub-address) to write to. |
| `data_len` | Number of payload bytes that follow. |
| `id` | Caller-assigned identifier (0–9); echoed in read responses. |
| `data...` | Payload bytes. |

### I2C Read
Send:
```
[0x0D, 0xC1, device_addr, reg_addr, read_len, id]
```
`0xC1 = 0x01 | 0x80 (read bit) | 0x40 (data_id bit)` — the data_id bit tells the board to include `id` as byte[2] in its response.

Board responds with a plain notification (bit 7 NOT set):
```
[0x0D, 0x01, id, byte0, byte1, ...]
```

### Reference test vector
```
Read 10 bytes from device 0x1C, register 0x0D, id=1:
  Send:    [0x0D, 0xC1, 0x1C, 0x0D, 0x0A, 0x01]
  Receive: [0x0D, 0x01, 0x01, data...]
```

### SPI Write
```
[0x0D, 0x02, slave_select, clock, mode, data_len, msb_first, nrf_pins, id, data...]
```

### SPI Read
Send:
```
[0x0D, 0xC2, slave_select, clock, mode, read_len, msb_first, nrf_pins, id]
```
`0xC2 = 0x02 | 0x80 | 0x40` — same read+data_id bit pattern as I2C.

Board responds:
```
[0x0D, 0x02, id, byte0, byte1, ...]
```

**SPI clock enum values:**
```
0 = 125 kHz
1 = 250 kHz
2 = 500 kHz
3 = 1 MHz
4 = 2 MHz
5 = 4 MHz
6 = 8 MHz
```

**SPI mode (CPOL/CPHA):**
```
0 = mode 0 (CPOL=0, CPHA=0)
1 = mode 1 (CPOL=0, CPHA=1)
2 = mode 2 (CPOL=1, CPHA=0)
3 = mode 3 (CPOL=1, CPHA=1)
```

**`msb_first`:** `1` = MSB transmitted first (typical), `0` = LSB first.
**`nrf_pins`:** `1` = use nRF internal SPI pins, `0` = use board expansion header pins.

---

## Module 0xFE — Debug

### Register Opcodes
```
RESET             = 0x01
BOOTLOADER        = 0x02
NOTIFICATION_SPOOF= 0x03
KEY_REGISTER      = 0x04
RESET_GC          = 0x05
DISCONNECT        = 0x06
POWER_SAVE        = 0x07
STACK_OVERFLOW    = 0x09
SCHEDULE_QUEUE    = 0x0A
```

### Commands
```
Reset:                        [0xFE, 0x01]
Jump to bootloader:           [0xFE, 0x02]
Spoof notification:           [0xFE, 0x03, ...bytes...]
Set key register:             [0xFE, 0x04, val_byte0, val_byte1, val_byte2, val_byte3]
Reset after GC:               [0xFE, 0x05]
Disconnect:                   [0xFE, 0x06]
Enable power save:            [0xFE, 0x07]
Set stack overflow assertion: [0xFE, 0x09, enable_byte]
Read stack overflow:          [0xFE, 0x89]   <- READ_REGISTER(0x09)
Read schedule queue:          [0xFE, 0x8A]   <- READ_REGISTER(0x0A)
```

**Fake button event:**
```
[0xFE, 0x03, 0x01, 0x01, 0x00, value]
```

---

## Data Scales Summary

| Data type | Scale constant | Notes |
|---|---|---|
| Accelerometer (all Bosch) | see FSR table | 16384, 8192, 4096, or 2048 LSB/g |
| Gyroscope (all Bosch) | see FSR table | 16.4, 32.8, 65.6, 131.2, 262.4 LSB/dps |
| Magnetometer BMM150 | 16.0 | LSB/uT |
| Barometer pressure | 256.0 | raw int / 256.0 = Pa |
| Barometer altitude | 256.0 | raw int / 256.0 = m |
| Temperature | 8.0 | raw int / 8.0 = °C |
| BME280 humidity | 1024.0 | raw / 1024.0 = % |
| Q16.16 fixed point | 65536.0 (0x10000) | raw / 65536 = value |
| Sensor fusion corrected acc | 1000.0 | raw float / 1000.0 = g |
| Sensor fusion gravity / linear acc | 9.80665 | raw float (m/s²) / g_scale = g |
| Battery voltage | direct uint16 | millivolts (raw bytes 1-2 of response) |
| Battery charge | direct uint8 | percent |

---

## Response Dispatch

The SDK uses a two-key map `(module_id, register_id)` or three-key `(module_id, register_id, data_id)` to route incoming BLE notifications.

```kotlin
// Two-byte header (no ID): data starts at response[2]
fun responseHandlerDataNoId(response: ByteArray) {
    val header = ResponseHeader(response[0], response[1])
    forwardResponse(header, response.copyOfRange(2, response.size))
}

// Three-byte header (with ID): data starts at response[3]
fun responseHandlerDataWithId(response: ByteArray) {
    val header = ResponseHeader(response[0], response[1], response[2])
    forwardResponse(header, response.copyOfRange(3, response.size))
}

// Packed data (multiple samples per notification): response[2] onward in 6-byte chunks
fun responseHandlerPackedData(response: ByteArray) {
    val header = ResponseHeader(response[0], response[1])
    var i = 2
    while (i < response.size) {
        dispatchSample(header, response, i, 6)
        i += 6
    }
}
```

---

## Tear Down Sequence

```
[0x09, 0x08]   <- DataProcessor REMOVE_ALL
[0x0A, 0x05]   <- Event REMOVE_ALL
[0x0B, 0x0A]   <- Logging REMOVE_ALL
```

---

## GATT UUIDs (Canonical)

| Characteristic | UUID |
|---|---|
| MetaWear Service | 326a9000-85cb-9195-d9dd-464cfbbae75a |
| Command (write) | 326a9001-85cb-9195-d9dd-464cfbbae75a |
| Notify (notify) | 326a9006-85cb-9195-d9dd-464cfbbae75a |
| Device Info Svc | 0000180a-0000-1000-8000-00805f9b34fb |
| Firmware Rev    | 00002a26-0000-1000-8000-00805f9b34fb |
| Model Number    | 00002a24-0000-1000-8000-00805f9b34fb |
| Hardware Rev    | 00002a27-0000-1000-8000-00805f9b34fb |
| Manufacturer    | 00002a29-0000-1000-8000-00805f9b34fb |
| Serial Number   | 00002a25-0000-1000-8000-00805f9b34fb |
