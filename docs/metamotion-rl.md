# MetaMotionRL (MMRL)

![MetaMotionRL](assets/MetaMotionR-t.png){ align=center }

MetaMotionRL is powered by a rechargeable lithium-polymer (LiPo) battery. It has a typical battery life of 1 to 14 days.

Since it is USB powered, you can add a battery pack to get even more life out of your MetaMotionRL.

The MMRL optionally comes with a plastic case. Please note that the proximity of the battery to the PCB when encased reduces the Bluetooth range.

## Details

| Name | Battery              | Size              | Memory              |
| :--- | :------------------- | :---------------- | :------------------ |
| MMRL | 100mAh recharge lipo | 17mm x 25mm x 5mm | 8MB (~1M entries) |

## Battery Life

How long the MMRL runs depends on which sensors are active, the sample rate, and whether you're **logging** (recording to onboard memory while disconnected) or **streaming** live over Bluetooth. Streaming is far more power-hungry — the radio alone draws roughly 7.5 mA.

For a logging session, two things run down and **whichever empties first ends the session**: the battery and the 8 MB onboard flash. On the MMRL the flash is small, so at high sample rates it — not the battery — is the limit. The table below shows both for common configurations at 100 Hz.

| Configuration (100 Hz)                            | Battery life | Flash fills  | Lasts about     |
| :------------------------------------------------ | :----------- | :----------- | :-------------- |
| Accelerometer only — logging                      | ~23 days     | ~1.5 hours   | **~1.5 hours**  |
| Full IMU (accel + gyro + mag) — logging           | ~2.6 days    | ~30 minutes  | **~30 minutes** |
| Sensor fusion → Euler angles — logging            | ~1.7 days    | ~45 minutes  | **~45 minutes** |
| Sensor fusion → Euler angles — streaming over BLE | ~10 hours    | n/a          | **~10 hours**   |

On the MMRL the 8 MB flash, not the battery, is what caps high-rate logging — at 100 Hz it fills in minutes to an hour or two. To capture for longer, lower the sample rate (flash time scales inversely with rate), stream over Bluetooth (~10 hours, but the device must stay in range of a connected phone), or use the [MetaMotion S](metamotion-s.md) (512 MB) for multi-day high-rate captures.

These are ballpark figures — real life shifts with temperature, battery age, Bluetooth settings, and exact rates. The per-sensor current draws and formulas behind them are in [Sensor Power Consumption](api-specification.md#sensor-power-consumption) and [Logging Memory Capacity](api-specification.md#logging-memory-capacity).

## Downloads

| Document     | Link |
| :----------- | :--- |
| Datasheet    | [MetaMotionRPS](assets/MetaMotionR-PS5.pdf) |
| Schematic    | [MetaMotionR0.5](assets/MetaMotionR_0.5.pdf) |
| CAD Model    | [STEP File](assets/MetaMotionR3-STEP.zip) |
| Case CAD     | [Rectangle Case STEP](assets/Rectangle-Case-STEP.zip) |
| Certificates | [MetaMotion Module](assets/MetaMotion-Certification.zip) |

## Sensor Datasheets

| Name | Accelerometer                                                          | Gyroscope                                                              | Magnetometer                                                                   | Temperature | Pressure | Light |
| :--- | :--------------------------------------------------------------------- | :--------------------------------------------------------------------- | :----------------------------------------------------------------------------- | :---------- | :------- | :---- |
| MMRL | [BMI160](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi160) | [BMI160](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi160) | [BMM150](https://www.bosch-sensortec.com/products/motion-sensors/magnetometers-bmm150/) | on-board    | none     | none  |

## Steps to get started

### 1. Unbox

Unbox your MetaMotionRL.

![Unbox](assets/MMR-1.jpg)

### 2. Power

Plug in your MetaMotionRL using a micro USB cable to charge the battery.

![Power](assets/MMR-2.jpg)

### 3. Wake Up

The MMRL will blink when charging.

![Wake Up](assets/MMR-3.jpg)

### 4. Connect

Download the App of your choice to start working with your MMRL.

![Connect](assets/MMR-5.jpg)

## Troubleshooting

Recharge the battery with a micro-USB to USB cable plugged in to power (AC adapter or computer) when you are not using it.

Troubleshoot any issues by following the steps [here](https://mbientlab.com/troubleshooting/).
