# Arduino IDE Installation on Ubuntu/Lubuntu

This guide explains **three installation paths**, how to grant serial-port access, and how to confirm everything works by uploading a loopback sketch. Follow the method that best fits your environment.

## Installing Arduino IDE (Three Options)
=== "Discover Software Center (GUI)"
    1. Open `Menu → System Tools → Discover`.
    2. Search for **Arduino IDE** and click **Install**.
    3. Launch it from `Menu → Applications → Programming → Arduino IDE`.
    4. First launch prompts for dialout permissions—click **Add**, enter your password, then log out/in (or reboot) so group membership refreshes.
=== "APT Packages (Terminal)"
    Update package list to the latest version and install Arduino IDE.

    === "Bash"
        ```bash
        sudo apt update
        sudo apt install arduino
        ```
    By executing the above command lines the latest version of Arduino IDE on you Lubuntu's respository will be installed.
    
    !!! warning "Warning"
        If `apt install` fails because the distro mirrors are outdated, use Option 3- Official Arduino ZIP (Latest Version).
    

=== "Official Arduino ZIP (Latest Version)"
    Download and unzip the latest build:
    === "Bash"
        ```bash
        wget https://downloads.arduino.cc/arduino-ide/latest/arduino-ide_latest_Linux_64bit.zip
        unzip arduino-ide_latest_Linux_64bit.zip
        cd arduino-ide_*_Linux_64bit
        ```
    Fix sandbox permissions (required for the Chromium-based UI):
    === "Bash"
        ```bash
        sudo chown root:root chrome-sandbox
        sudo chmod 4755 chrome-sandbox
        ```
    Start the IDE:
    === "Bash"
        ```bash
        ./arduino-ide
        ```
    !!! note "Optional"
        Create a `.desktop` launcher for long-term use.

## Grant Serial-Port Access (dialout group)
By default, only users in the dialout group can access serial ports like /dev/ttyUSB0. Run the following code:

=== "Bash"
    ```bash
    sudo usermod -aG dialout $USER
    ```
!!! warning "Warning"
    Log out/in (or reboot). Without this, uploads will fail with `Permission denied` when opening `/dev/ttyUSB0`.

## Launching the IDE
- GUI: `Menu → Programming → Arduino IDE`
- Terminal: `arduino` (APT build) or `~/Downloads/arduino-ide_*/arduino-ide` (ZIP build)

## Verify the Installation (Two-Way Serial Test)
1. Connect your Arduino via USB.
2. In the IDE, go to **Tools → Board** and pick your model (e.g., *Arduino Uno*).
3. Go to **Tools → Port** and select the entry that appeared when you plugged the board in (for example `/dev/ttyUSB0`).

### Upload the Echo Sketch
=== "c++" 
```cpp
/*
  Serial Echo Test
  Type in the Serial Monitor and the Arduino will echo back.
*/

void setup() {
  Serial.begin(9600);      // Initialise serial at 9600 bps
  while (!Serial) { ; }    // Wait for serial port (needed on Leonardo/Micro)
  Serial.println("Ready!");
}

void loop() {
  if (Serial.available() > 0) {
    String msg = Serial.readStringUntil('\n');
    Serial.print("You sent: ");
    Serial.println(msg);
  }
}
```
Upload: **Sketch → Upload** (or click the right-arrow icon).

### Use the Serial Monitor
1. **Tools → Serial Monitor**
2. Set **Baud** to `9600` and **Line Ending** to `Newline`.
3. You should see:
   ```
   Ready!
   ```
4. Type a message (e.g. Hello class!), press **Send**. Expect a reply:
   ```
   You sent: Hello class!
   ```
This proves **full-duplex serial** is working: the PC sends characters → Arduino echoes them → both RX/TX LEDs blink.

??? info "Theory"
    ## Theory: What the Serial Test Demonstrates
    - **USB-to-Serial bridge:** The Arduino’s onboard USB chip appears as `/dev/ttyUSB0` (or `/dev/ttyACM0`). Every byte typed in the Serial Monitor travels over this virtual serial cable to the microcontroller.
    - **Baud rate agreement:** Both sides run at 9600 baud, meaning 9,600 bits per second. If the baud rates mismatch you would see random symbols instead of readable text.
    - **Full duplex:** USB serial exposes separate TX (transmit) and RX (receive) channels, so the computer and board can talk at the same time. The echo sketch uses `Serial.available()` to see inbound data while still free to send outbound messages.
    - **Line discipline:** `Serial.readStringUntil('\n')` waits for a newline character. That is why the Serial Monitor is set to “Newline”—it tells the sketch when a complete message has arrived.
    - **Feedback loop:** `Serial.println()` sends text plus a newline back to the PC. Seeing the expected response proves the IDE, USB drivers, permissions, and board firmware all agree on the protocol.

    ### Direction of Travel
    - **Arduino → PC:** The sketch calls `Serial.println()` to send the startup line and every “You sent:” response. Those characters ride the TX line through the USB interface and appear instantly in the Serial Monitor.
    - **PC → Arduino:** Whatever you type into the Serial Monitor is pushed across the RX line. The call to `Serial.readStringUntil('\n')` grabs the message, which the sketch then echoes back.

    Because TX and RX are active simultaneously on the same virtual cable, this setup exemplifies **two-way (full-duplex) serial communication**—both endpoints can transmit whenever they need, and acknowledgements arrive without waiting for the other side to finish talking.


## Troubleshooting Checklist
- **Board not detected:** Try another USB cable/port. Run `lsusb` to confirm the device shows up.
- **Port menu empty:** Ensure `dmesg | tail` shows the board attach event; install `usbutils` if `lsusb` is missing.
- **Permission denied:** Confirm group membership with `groups`—`dialout` must be listed. Reboot if you just added it.
- **Upload hangs:** Close any other apps using the serial port (screen, ModemManager) and retry.

With these steps, even new users can install the Arduino IDE on Lubuntu, gain serial access, and validate the toolchain with a self-contained test sketch.
