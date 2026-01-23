# Data Transfer Through Dweet

## Overview

## 
=== "Python"
```python
import time
import requests
import serial
from datetime import datetime

# ---- CONFIG ----
BASE_THING_NAME = "whittlesea_tech_school"
SERIAL_PORT = "/dev/rfcomm0" 
BAUD_RATE = 9600
POST_URL = f"https://dweet.cc/dweet/for/{BASE_THING_NAME}"
POLL_INTERVAL = 2  # seconds

def parse_temperature(line: str):
    # The Arduino prints a number like "24.73"
    try:
        return float(line.strip())
    except ValueError:
        return None

def main():
    print(f"Opening serial {SERIAL_PORT} @ {BAUD_RATE}")
    with serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=2) as bt:
        time.sleep(2)  # give Arduino time to reset on open
        print(f"Publishing to {POST_URL}")
        while True:
            try:
                raw = bt.readline().decode("utf-8", errors="ignore")
                temp_c = parse_temperature(raw)
                if temp_c is None:
                    continue  # skip malformed lines

                # Get current time
                current_time = datetime.now()
                time_string = current_time.isoformat()

                payload = {"temperature_c": temp_c,
                           "time": time_string
                           }
                r = requests.post(POST_URL, json=payload, timeout=5)
                r.raise_for_status()

                # dweet returns JSON with "with" metadata
                print(f"Sent: {payload} | Response: {r.status_code}")
                time.sleep(POLL_INTERVAL)

            except (serial.SerialException, requests.RequestException) as e:
                print(f"Warning: {e}. Retrying in 3s...")
                time.sleep(3)
            except KeyboardInterrupt:
                print("Exiting.")
                break

if __name__ == "__main__":
    current_time = datetime.now()
    time_string = current_time.isoformat()
    print(time_string)
    print(type(time_string))
    # main()
```