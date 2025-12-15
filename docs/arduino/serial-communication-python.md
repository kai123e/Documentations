# Serial Communication with Arduino via Python (Thonny on Lubuntu)

Use this guide to install the Thonny IDE, add the `pyserial` library, and prove that Python can talk to an Arduino using a simple echo demo.

## 1. Install Thonny IDE
=== "Discover (GUI)"
    1. Open `Menu → System Tools → Discover`.
    2. Search **Thonny** and click **Install**.
    3. Launch via `Menu → Applications → Programming → Thonny`.
=== "Terminal"
    ```bash
    sudo apt update
    sudo apt install thonny
    thonny
    ```

## 2. Install `pyserial`
`pyserial` lets Python scripts open `/dev/ttyUSB0` (or similar) and exchange bytes with your Arduino. Because global `pip` installs are locked down on modern Ubuntu, use the safe APT package:

=== "Bash"
    ```bash
    sudo apt install python3-serial
    ```

Alternative: create a virtual environment (`python3 -m venv venv && source venv/bin/activate`) and run `pip install pyserial` inside it if you prefer isolated dependencies.

## 3. Verify the Library
In Thonny (or any Python terminal), enter:

=== "Python"
    ````python
    import serial
    print(serial.__version__)
    ````

The Thonny shell should output the current version of pyserial, confirming successful installation.

## 4. Serial Communication Demo
This section demonstrates how to establish two-way communication between Arduino and Python using the Thonny IDE. You will upload a simple Arduino sketch that echoes messages and then write a Python script to send and receive data.

### Arduino Echo Sketch
=== "c++"
    ```cpp
        /*Serial Echo Test
        Type in the Serial Monitor and the Arduino will echo back.
        Works on UNO/Nano/Mega; also fine on Leonardo/Micro if you keep the wait for Serial.
        */
        
        void setup() {
        Serial.begin(9600);           // Initialize serial at 9600 bps
        while (!Serial) {}         
        Serial.println("Ready!"); // send initial message
        }
        
        void loop() {
        // Check if data is available
        if (Serial.available() > 0) {
            String msg = Serial.readStringUntil('\n');  // Read the message
            Serial.print("Recieved: ");       
        Serial.println(msg);
        }
        }
    ```

Upload the sketch by `sketch -> upload`. Now open the Thonny IDE. Create a new Python file and paste the following code:

=== "Python"
    ```python
    import serial
    import time
    
    # Set your COM port (update with your actual port!)
    PORT = '/dev/ttyUSB0'  # Adjust for your system (Linux)
    BAUD = 9600
    
    # Open serial connection to Arduino
    arduino = serial.Serial(PORT, BAUD, timeout=1)
    time.sleep(2)  # Give Arduino time to reset
    
    print("Connected to Arduino!")
    # Response of Arduino from initial setup
    response = arduino.readline().decode('utf-8').strip()  # Decode and strip extra whitespace
    print(f"Arduino replied: {response}")
    
    # Send a test message to Arduino
    message = "Hello Arduino!"
    arduino.write(message.encode())  # Convert string to bytes
    
    # Read Arduino's response
    time.sleep(0.1) # need to include to wait for response (may need to increase)
    response = arduino.readline().decode('utf-8').strip()  # Decode and strip extra whitespace
    print(f"Arduino replied: {response}")
    
    arduino.close()  # Always close the serial connection after use
    ```
Click run button in Thony. The expected output in the terminal should be:

=== "Terminal"
    ```bash
    Connected to Arduino!
    Arduino replied: Ready!
    Arduino replied: Received: Hello Arduino!
    ```