# HC-05 Bluetooth Module Setup

Connect an HC-05 classic Bluetooth module to an Arduino, pair it with Lubuntu, and stream serial temperature data from DS18B20 over the wireless link.

## Hardware Setup
HC-05 logic pins expect 3.3 V. When using a 5 V Arduino, level-shift the Arduino TX line (going into HC-05 RX) with a simple voltage divider (e.g., 1.8 kΩ + 3.3 kΩ) is required. Alternatively, bi-directional logic level converter can be used to convert 5V and 3.3V logic.

**Wiring (Arduino Uno example)**

- HC-05 `TXD` → Arduino `pin 0` (software RX)
- HC-05 `RXD` → Arduino `pin 1` (software TX) **through divider**

- HC-05 `VCC` → 5 V (Arduino `5V pin`/ or other 5V devices)
- HC-05 `GND` → (Arduino `GND pin`)
- Optional: `KEY` pin → 3.3 V to enter AT mode (not needed for this data-mode example)

## Pair HC-05 with Lubuntu
1. Open `Menu → Preferences → Bluetooth Manager` (or `blueman-manager`).
2. Enable Bluetooth, then search for devices. Select `HC-05` (or similar name).
3. Click **Pair** and enter PIN `1234` (fallback `0000`).
4. After pairing, note the MAC address (format `XX:XX:XX:XX:XX:XX`).
5. In a terminal, list RFComm ports:
   ```bash
   ls /dev/rfcomm*
   ```
   If no device appears, bind one manually:
   ```bash
   sudo rfcomm connect hci0 XX:XX:XX:XX:XX:XX 1
   ```
   This opens `/dev/rfcomm0` and keeps it active until you press `Ctrl+C`.

=== "C++"
```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

#define ONE_WIRE_BUS 2
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
Serial.begin(9600);  // hardware serial for HC-05
sensors.begin();
}

void loop() {
sensors.requestTemperatures();
float tempC = sensors.getTempCByIndex(0);
Serial.println(String(tempC, 1));
delay(1000);
}
```

=== "Python"
```python
import serial
import time

port = "/dev/rfcomm0"
baudrate = 9600

bt = serial.Serial(port, baudrate, timeout=1)
time.sleep(2)  # allow Arduino to reset

print("Connected to HC-05! Reading temperatures:")

while True:
    try:
        line = bt.readline()            # read one line from HC-05
        if line:
            # decode safely, ignoring errors
            decoded = line.decode('utf-8', errors='ignore').strip()
            if decoded:
                print(f"Temperature: {decoded} °C")
    except Exception as e:
        print("Error:", e)

```


## Troubleshooting
- **No `/dev/rfcomm0`:** Use `sudo rfcomm bind 0 XX:XX:XX:XX:XX:XX 1` to create a persistent device, then connect to `/dev/rfcomm0`.
- **Wrong baud:** HC-05 defaults to 9600 baud in data mode. Ensure both Arduino and Python scripts use the same value.
- **Module stuck in AT mode:** Ensure the `KEY` pin is low (disconnect it) so the module enters data mode.
- **Acts like serial cable:** Remember that once paired, Bluetooth simply emulates a serial port—anything you send through `/dev/rfcomm0` arrives at the HC-05 RX pin.
