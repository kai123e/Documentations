# Setting up Python on Lubuntu

Follow these steps to confirm Python is installed, add any missing components, and run a quick “Hello, World” test.

## 1. Check Existing Python
Open a terminal (`Ctrl+Alt+T`) and run:

=== "Bash"
    ```bash
    python3 --version
    ```

If Python is already present you will see something like `Python 3.10.12` (the number dependings on your version). No further action is required unless you also need `pip`.

## 2. Install Python (If Needed)
If the previous command reports “command not found,” install the interpreter via APT:

=== "Bash"
    ```bash
    sudo apt update
    sudo apt install python3
    ```

### Add pip (Python Package Manager)

=== "Bash"
    ```bash
    sudo apt install python3-pip
    ```
This provides the `pip3` command so you can install third-party libraries later.

## 3. Verify Installations
Double-check both tools:

=== "Bash"
    ```bash
    python3 --version
    pip3 --version
    ```
Seeing valid version numbers confirms the binaries are on your PATH.

## 4. Test the Python Setup
Create a simple script and execute it.

1. Open a new file:
	```bash
	nano hello.py
	```
2. Paste the following code:
	```python
	print("Hello, Python on Lubuntu!")
	```
3. Save (`Ctrl+O`, Enter) and exit (`Ctrl+X`).
4. Run the script:
	```bash
	python3 hello.py
	```
5. Expected output:
	```
	Hello, Python on Lubuntu!
	```

If you see the message, Python, pip, and your shell environment are working correctly.

