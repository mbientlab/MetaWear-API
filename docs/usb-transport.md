# USB Transport

The **MetaMotionS (MMS)** can be controlled over a USB cable as an alternative to Bluetooth Low Energy. The board enumerates as a USB CDC (communications device class) serial port and accepts the same MetaWear command packets that are normally written to the BLE Command Characteristic.

Everything above the transport layer is unchanged. Modules, registers, little-endian encoding, data processors, the logging readout sequence, and every byte-level detail in the [API Specification](api-specification.md) apply exactly as written. Only the way bytes reach the board is different.

!!! info "Hardware requirement"
    USB transport requires the **MetaMotionS**. The MMS uses the Nordic **nRF52840**, which has a native USB device peripheral. The MetaMotionRL and all earlier boards use the **nRF52832**, which has no USB peripheral, so their micro-USB port supplies power only.

---

## Transport Model

The USB transport is a drop-in replacement for the GATT layer. Every BLE concept has a direct USB equivalent, which is what allows a single SDK to speak either transport without changing any higher-level code.

| BLE concept | USB equivalent |
| :---------- | :------------- |
| Scan for advertising devices | Enumerate USB serial ports matching the MetaWear VID:PID |
| Device MAC address | USB serial-number descriptor (same MAC, hex, no separators) |
| Connect to GATT server | Open the serial port |
| Discover services and characteristics | Not required, the layout is fixed |
| Write to Command Characteristic `326A9001` | Write one framed packet to the port |
| Subscribe to Notification Characteristic `326A9006` | Not required, framed packets arrive on the port |
| Notification received | One decoded frame from the port |
| Read Device Information characteristics | Identification command (`?`) issued once at connect |
| GAP disconnect | Write Debug `0xFE 0x06`, then close the port |

---

## 1. Enumeration

A MetaMotionS attached over USB enumerates as a CDC serial device with the following USB descriptors.

| Descriptor | Value | Notes |
| :--------- | :---- | :---- |
| Vendor ID (VID) | `0x1915` | Nordic Semiconductor |
| Product ID (PID) | `0xD978` | MetaWear USB interface |
| Serial number | Board MAC address, 12 hex characters, no separators | For example `E8C98F527B07` |
| Product | Model name | For example `MetaMotionS`. Treat a missing value as `MetaMotionS`. |

Discovery is therefore a matter of listing the host's serial ports and filtering on `VID:PID=1915:D978`.

### Recovering the MAC address

The USB serial-number descriptor carries the board's BLE MAC address with the separators stripped. Insert a colon every two characters to recover the canonical form:

```
E8C98F527B07   ->   E8:C9:8F:52:7B:07
```

This matters because it lets a host address the same physical board by the same identifier over either transport. An SDK can accept a MAC from the user, check whether a USB port with that serial number is present, and silently prefer USB when it is.

### Device paths

The operating system assigns the port a path in its usual convention for CDC devices:

| Platform | Typical path |
| :------- | :----------- |
| Linux | `/dev/ttyACM0` |
| macOS | `/dev/cu.usbmodem*` |
| Windows | `COM3` |

On Linux the invoking user generally needs membership in the group that owns the device node (commonly `dialout`) or an equivalent udev rule.

---

## 2. Opening the Port

| Setting | Value |
| :------ | :---- |
| Baud rate | `1000000` |
| Read timeout | 100 ms |
| Data bits / parity / stop bits | 8 / none / 1 |
| Flow control | None |

After opening the port:

1. Wait approximately **100 ms** for the device to settle.
2. **Reset (flush) both the input and output buffers.** The port may hold stale bytes from a previous session, and a partial frame left in the input buffer will desynchronise the decoder on the very first read.

!!! warning "Do not skip the buffer reset"
    The frame decoder is a stream state machine with no checksum. A leftover fragment will be interpreted as frame data and can corrupt the first several packets of a session.

---

## 3. Identification Handshake

Over BLE, an SDK reads the Device Information Service characteristics individually. Over USB there is a single identification command that returns all of them at once. Issue it immediately after opening the port, before sending any MetaWear command frames.

**Request.** Write the ASCII string `?` followed by a newline, then flush:

```
3F 0A          ("?\n")
```

**Response.** One newline-terminated ASCII line containing six space-separated fields, in this fixed order:

| Position | Field | Corresponding DIS characteristic |
| :------- | :---- | :------------------------------- |
| 1 | Manufacturer | `00002A29-0000-1000-8000-00805F9B34FB` |
| 2 | Model name | *(no DIS equivalent, human-readable name)* |
| 3 | Model number | `00002A24-0000-1000-8000-00805F9B34FB` |
| 4 | Hardware revision | `00002A27-0000-1000-8000-00805F9B34FB` |
| 5 | Firmware revision | `00002A26-0000-1000-8000-00805F9B34FB` |
| 6 | Serial number | `00002A25-0000-1000-8000-00805F9B34FB` |

Split the line on spaces and cache the six values. Any subsequent request for a Device Information characteristic is served from this cache rather than from a live read. Model number `8` identifies the MetaMotionS, matching the value returned over BLE.

!!! note
    The identification command is plain ASCII and is **not** wrapped in the binary frame format described below. It is the only exception. Every byte exchanged after this handshake is framed.

---

## 4. Frame Format

All MetaWear traffic in both directions is wrapped in a simple length-prefixed frame.

```
+--------+--------+---------------------------+--------+
|  0x1F  | length |      payload (N bytes)    |  0x0A  |
+--------+--------+---------------------------+--------+
   start    N=len      MetaWear command or        stop
    byte              notification payload        byte
```

| Field | Size | Description |
| :---- | :--- | :---------- |
| Start byte | 1 | Always `0x1F` |
| Length | 1 | Number of payload bytes that follow, `0x01` to `0xFF` |
| Payload | *length* | The MetaWear packet, byte for byte |
| Stop byte | 1 | Always `0x0A` |

The payload is **exactly** the byte array that would otherwise be written to Command Characteristic `326A9001`, or exactly the byte array that a notification from `326A9006` would carry. The first payload byte is the module opcode and the second is the register, precisely as documented in the [API Specification](api-specification.md).

### The length prefix is authoritative

There is **no escaping and no byte stuffing**. The values `0x1F` and `0x0A` may appear freely inside the payload, and often do. Framing is unambiguous because the decoder consumes exactly *length* bytes after reading the length field, without inspecting their values.

Implementations that scan for the stop byte instead of honouring the length field will fragment any packet containing `0x0A`.

### Worked examples

**Debug GAP Disconnect**, module `0xFE`, register `0x06`, no parameters. Payload is 2 bytes:

```
Payload:  FE 06
Frame:    1F 02 FE 06 0A
```

**Logging Flush Pending Writes**, module `0x0B`, register `0x10`, one parameter byte. Payload is 3 bytes:

```
Payload:  0B 10 01
Frame:    1F 03 0B 10 01 0A
```

**Logging Readout**, module `0x0B`, register `0x06`, requesting 1000 entries (`0x000003E8`) with a progress notification every 20 entries (`0x00000014`). Both counts are little-endian uint32. Payload is 10 bytes:

```
Payload:  0B 06 E8 03 00 00 14 00 00 00
Frame:    1F 0A 0B 06 E8 03 00 00 14 00 00 00 0A
```

Note the third example carefully. The length byte is `0x0A`, the same value as the stop byte, and the payload itself contains no stop byte at all before the real terminator. This is the clearest demonstration of why the length field, and not delimiter scanning, must drive the decoder.

### Constraints

- **Minimum payload length is 1.** A length byte of `0x00` is not representable and will desynchronise a decoder that treats `0` as "length not yet read". In practice this never occurs, since every MetaWear packet is at least 2 bytes (module opcode plus register).
- **Maximum payload length is 255**, bounded by the single length byte. MetaWear command packets are far shorter than this in normal operation.

---

## 5. Receive Decoder

Incoming bytes arrive as an unstructured stream and must be reassembled. Read in chunks, up to **1024 bytes** per read, taking whatever is currently available in the input buffer, then feed the bytes through a state machine one at a time.

```
state:
    started    = false
    length     = 0
    received   = 0
    buffer     = []

for each byte c from the port:

    if not started:
        if c == 0x1F:                    # start byte
            started  = true
            length   = 0
            received = 0
            buffer   = []
        # any other byte outside a frame is ignored
        continue

    if length == 0:                      # this byte is the length field
        length = c
        continue

    if received < length:                # this byte is payload
        buffer.append(c)
        received = received + 1
        continue

    if c == 0x0A:                        # this byte is the terminator
        started = false
        emit(buffer)                     # one complete MetaWear packet
    # a non-terminator here means the frame is malformed:
    # discard bytes until a valid terminator or restart on 0x1F
```

Each emitted buffer is one complete MetaWear packet. Hand it to the same dispatch logic used for BLE notifications, described under [Response Dispatch](api-specification.md#response-dispatch): key on `(module_id, register_id)`, or `(module_id, register_id, data_id)` where the register carries an index.

### Reading efficiently

Take `max(1, min(1024, bytes_available))` per read call rather than reading a single byte at a time. During a log download the board can deliver packets faster than a byte-at-a-time loop will drain them, and the input buffer will overflow. This chunked read is the single most important performance detail in the transport.

---

## 6. Mapping to an Existing SDK

If you are wrapping the MetaWear C++ SDK, the USB transport substitutes for the four `BtleConnection` callbacks. Nothing in the C++ library needs to change, it continues to believe it is speaking GATT.

| C++ SDK callback | USB implementation |
| :--------------- | :----------------- |
| `write_gatt_char` | Frame the supplied bytes and write them to the port |
| `read_gatt_char` | Return the cached value from the identification handshake |
| `enable_notifications` | Immediately invoke the completion callback with success, there is nothing to subscribe to |
| `on_disconnect` | Invoke when the port closes or the device is unplugged |

Two details are worth calling out:

- **Write-with-response and write-without-response are identical on the wire.** Both produce the same frame. The distinction exists only in when the SDK's completion callback fires.
- **Service and characteristic discovery must be stubbed.** Report the MetaWear Service `326A9000-85CB-9195-D9DD-464CFBBAE75A` and the Device Information Service `0000180A-0000-1000-8000-00805F9B34FB` as present. Report every other service as absent.

---

## 7. Log Download over USB

Log download is the most common reason to reach for USB, since a full 512 MB NAND readout over BLE is slow. The register sequence is identical to BLE and is documented in full under the [Logging Module](api-specification.md#0x0b-logging-module). The USB-specific parts are the framing and the throughput.

### Sequence

1. **Stop the sensor** producing the logged signal, and stop logging with Logging Enable `0x0B 0x01 0x00`.
2. **Flush pending writes.** Write `0x0B 0x10 0x01`. This is **required on the MMS** and is the single most commonly missed step. The MMS buffers log entries in a NAND page cache, and any entries still in that cache at download time are simply absent from the readout with no error reported. Skipping this step silently loses the most recent data.
3. **Query the log length** with Logging Length `0x0B 0x05` to learn how many entries are pending.
4. **Start the readout.** Write Logging Readout `0x0B 0x06` with a uint32 entry count and a uint32 progress-notification interval, both little-endian.
5. **Consume Readout Notify** `0x0B 0x07` packets. Each carries one or two 9-byte log entries. Each entry is: byte 0 with the Trigger UID in bits 0 to 4 and the Reset UID in bits 5 to 7, bytes 1 to 4 the uint32 timestamp in ticks, bytes 5 to 8 the uint32 data payload.
6. **Track progress** with Readout Progress `0x0B 0x08`, a uint32 count of entries remaining. The download is complete when this reaches zero.
7. **Acknowledge each page.** When a Readout Page Complete `0x0B 0x0D` notification arrives with an empty payload, reply with Readout Page Confirm `0x0B 0x0E`, also empty.
8. **Tear down** the data processors, events, and log triggers, per the [Tear Down Sequence](api-specification.md#tear-down-sequence).

!!! danger "Page confirm permanently erases entries"
    Sending Readout Page Confirm `0x0B 0x0E` nulls the confirmed entries in flash. Persist the received data before confirming the page, not after. There is no way to re-read a confirmed page.

### Timestamps

Log entry timestamps are in internal ticks where one tick is 48/32768 seconds, approximately **1.46484375 ms**. The Reset UID in the entry header distinguishes tick counts recorded across device resets, since the tick counter restarts at each reset. Convert to wall-clock time by correlating against Logging Time `0x0B 0x04`, which returns the current tick count and the current Reset UID.

---

## 8. Differences from BLE

| Property | BLE | USB |
| :------- | :-- | :-- |
| Packet size | 20-byte characteristic limit | Length-prefixed, up to 255 bytes per frame |
| Pacing | Bound by the connection interval, 7.5 ms to 4 s | Bound by the serial link only |
| Discovery | Advertising and scanning | USB enumeration |
| Bonding and pairing | Supported | Not applicable |
| Notification subscription | Required | Not required |
| Device Information reads | Individual characteristic reads | Single identification command |
| Range | Wireless | Cable length |
| Power | Battery | Bus-powered, and the battery charges concurrently |

### Limitations

- **Firmware update over USB is not supported.** MetaBoot (DFU) mode is detected by the presence of the DFU service `00001530-1212-EFDE-1523-785FEABCD123`, which the USB transport does not expose. Perform firmware updates over BLE.
- **The board must already be awake and enumerated.** A board in deep sleep will begin charging when plugged in but will not necessarily enumerate immediately.
- **One host connection at a time**, as with any serial port.

---

## 9. Implementation Checklist

- [ ] Enumerate serial ports and filter on VID `0x1915`, PID `0xD978`
- [ ] Parse the MAC address from the USB serial-number descriptor
- [ ] Open the port at 1000000 baud with a 100 ms read timeout
- [ ] Wait ~100 ms, then flush both input and output buffers
- [ ] Send `?\n` and parse the six space-separated identification fields
- [ ] Cache those values to serve Device Information reads
- [ ] Frame every outbound packet as `0x1F | length | payload | 0x0A`
- [ ] Decode inbound frames with a length-driven state machine, never by delimiter scanning
- [ ] Read up to 1024 bytes per call, sized to the available input
- [ ] Dispatch decoded payloads on `(module, register)` exactly as BLE notifications
- [ ] Treat notification subscription as an immediate success with no wire traffic
- [ ] Flush pending writes (`0x0B 0x10 0x01`) before every log download
- [ ] Persist log entries before sending Readout Page Confirm
- [ ] On disconnect, write Debug `0xFE 0x06` and then close the port
