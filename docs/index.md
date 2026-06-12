# MetaSensors

We currently have two MetaSensor types that are supported today, the **MMS** and the **MMRL**.

All MetaSensors include:

- an ARM MCU (Nordic nRF52)
- Bluetooth Low Energy
- flash memory for data logging
- an RGB LED
- a push button
- a rechargeable lithium-polymer battery

---

## Supported Boards

Select your MetaSensor from the table below to continue:

| Name                        | Battery             | Size              | Memory              | Acc | Gyro | Mag | Temp     | Baro | Light |
| :-------------------------- | :------------------ | :---------------- | :------------------ | :-- | :--- | :-- | :------- | :--- | :---- |
| [MetaMotionRL](metamotion-rl.md) | 100mAh rechargeable | 17mm x 25mm x 5mm | 8MB (~1M entries) | X   | X    | X   | on-board |      |       |
| [MetaMotionS](metamotion-s.md)   | 100mAh rechargeable | 17mm x 25mm x 5mm | 512MB (~67M entries) | X   | X    | X   | X        | X    | X     |

---

## Resources

- **[API Specification](api-specification.md)** — The complete MetaWear BLE serial protocol: module registers, byte-level wire reference, data processor filters, sensor fusion modes, and board model reference.
- **[MbientLab Store](https://mbientlab.com)** — Purchase MetaSensors and accessories.
