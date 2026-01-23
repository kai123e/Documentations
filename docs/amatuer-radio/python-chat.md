# Replicating Linbpq Chat Behaviour Using Python (Port-Oriented Design)

## Overview
This section documents a **Python-based chat system** that conceptually replicates the chat and session behaviour of LinBPQ, using an object-oriented approach and persistent storage instead of AX.25 or TCP networking.
The implementation is intentionally simple and suitable for:

 - CLI environments
 - packet-style text interfaces
 - learning and prototyping LinBPQ-like behaviour
 - environments where RF or TCP ports are abstracted

Although this program does not implement AX.25, it mirrors LinBPQ’s **operational model** rather than its protocol stack.

## Overall Architecture
The system is built around a single-user session object.

User (Callsign) -> Chat Object -> Command Loop (chat_main) -> Topic Files (logical ports)

Each connected user:
has one Chat instance
joins one topic at a time
writes to shared text files
This mirrors how LinBPQ handles sessions, ports, and chat applications.

## Imports and Why They Matter

=== "Python"
```python
import sys
import os
from datetime import datetime
from typing import List
```

**Explanation:**

- `sys` → direct access to stdin/stdout (important for Telnet, packet radio, and pipes)
- `os` → file and directory management (chat logs, user tracking)
- `datetime` → timestamps for messages and logs
- `List` → type hints for readability and correctness

This design avoids GUI or network libraries so it works over:
- Telnet
- SSH
- AX.25 packet gateways
- redirected stdin/stdout (e.g. LinBPQ applications)

## Global Constants (System Configuration)

=== "Python"
```python
CHAT = [
    "/U - Show Users",
    "/T - Show Topics",
    "/T Name - Join/Create Topic",
    "/B - Return to Main",
    "/M - Show Current Topic Messages",
    "/bye",
]
CALLSIGN_LENGTH = 6
CHAT_DIR_NAME = "Group chats"
CONNECTED_USER_TEXT_LOG = "connected users.txt"
CONNECTED_USER_TEXT_LOG_RECORD = "connected user records.txt"
```

**Explanation:**

These values define system behaviour without modifying logic:
- `CHAT` → help menu displayed to users
- `CALLSIGN_LENGTH` → minimum callsign length (matches amateur radio conventions)
- `CHAT_DIR_NAME` → directory where chat rooms live
- `CONNECTED_USER_TEXT_LOG` → active users
- `CONNECTED_USER_TEXT_LOG_RECORD` → historical user log

## HelperFunctions Class (Utility Layer)

This class does not store state. It provides shared tools used everywhere else.

### Graceful Exit Handling

=== "Python"
```python
def _chat_exit(self, callSign: str, stripped: str, stringToExit: str) -> None:
    if stripped.lower() == stringToExit and (callSign != None):
        sys.stdout.write("Goodbye \r\n")
        sys.stdout.flush()
        Chat(callSign).delete_user()
        raise SystemExit(0)
```

**Purpose:** Handles clean program termination when the user types `/bye`.

**Step-by-step:**

1. Checks if user input matches the exit command
2. Confirms a callsign exists (user is logged in)
3. Prints a goodbye message
4. Removes the user from `connected users.txt`
5. Terminates the program safely

This prevents ghost users, a common issue in packet systems.

### Safe Input Reader (REPL Core)

=== "Python"
```python
def ask_for_input(self, callSign=None, stringToExit='/bye') -> str:
    line = sys.stdin.readline()
    if not line:
        return ''
    
    stripped = line.strip()
    self._chat_exit(callSign, stripped, stringToExit)
    
    return stripped
```

**Purpose:** Reads user input safely and checks for exit commands.

**What it does:**

- Reads from standard input (stdin)
- Strips newline characters
- Automatically calls `_chat_exit()` if `/bye` is entered

**Why this matters:** In real packet radio, every keystroke arrives as a stream. This function simulates that behaviour.

### Call Sign Registration

=== "Python"
```python
def record_call_sign(self, logName="connected users.txt", logRecordName="connected user records.txt")->str:
    while True:
        sys.stdout.write("Enter your call sign " + "\r\n")
        sys.stdout.flush()
        callSign = self.ask_for_input().strip()
        if len(callSign) < CALLSIGN_LENGTH:
            sys.stdout.write("The call sign that you've entered is incorrect " + "\r\n")
            sys.stdout.flush()
        else:
            chat = Chat(callSign)
            users_file = os.path.join(os.path.dirname(__file__), logName)
            chat.write_file(users_file)
            users_record_file = os.path.join(os.path.dirname(__file__), logRecordName)
            chat.write_file(users_record_file)
            return callSign
```

**Purpose:** Registers a user before allowing chat access.

**Detailed flow:**

1. Prompts user for a callsign
2. Validates minimum length
3. Writes callsign to:
   - active users file
   - historical record file
4. Returns the callsign for session use

This matches packet radio logging requirements.

### File and Directory Safety

=== "Python"
```python
def make_dir(self, line, type: str) -> str:
    name = (line or "").strip()
    if not name:
        sys.stdout.write("Invalid name\r\n")
        sys.stdout.flush()
        return ""
    
    safe_name = name.replace("/", "_").replace("\\", "_")
    base_dir = os.path.join(os.path.dirname(__file__), type)
    os.makedirs(base_dir, exist_ok=True)
    return os.path.join(base_dir, f"{safe_name}.txt")
```

**Purpose:** Creates a safe filename and directory for chat logs.

**Important behaviour:**

- Removes `/` and `\` to prevent path traversal
- Automatically creates directories

**Security note:** This prevents users from writing outside the chat directory.

### Chat Menu Selection

=== "Python"
```python
def choose_menu(self, names: List[str], menuType: str) -> None:
    lines = [f"{name}" for _, name in enumerate(names)]
    sys.stdout.write(f"{menuType}:\r\n" + "\r\n".join(lines) + "\r\n")
    sys.stdout.flush()
    return
```

**Purpose:** Prints formatted menus (help, command lists).

**Why separate this function:** Avoids repeating print logic and keeps menus consistent.

### Make/Append text to Files

=== "Python"
```python
def make_text_file(self, line:str, type:str) -> str:
    chat_file = self.make_dir(line, type)
    if not chat_file:
        return ""
    
    if not os.path.exists(chat_file):
        open(chat_file, "a", encoding="utf-8").close()
    
    return chat_file
```

**Purpose:** Ensures a text file exists for a given chat topic.

**Example:** Typing `/T Weather` creates:
```
Group chats/Weather.txt
```

### Get File Lines

=== "Python"
```python
def get_file_lines(self, path) -> List[str]:
    """All chat messages"""
    with open(path, "r", encoding="utf-8") as f:
        msg = []
        for row in f:
            msg.append(row)
    return msg
```

**Purpose:** Reads all messages from a chat log.

**Why not stream?** This keeps the system simple and easy for beginners to understand.

## Chat Class

This class models a single chat session for one connected user.

=== "Python"
```python
def __init__(self, callSign):
    self.callSign = callSign
```

**Purpose:** Stores the user's callsign for tagging messages.

Every message written includes this callsign.

### Write File

=== "Python"
```python
def write_file(self, path, msg=""):
    now = datetime.now()
    formatted_string = now.strftime("%B %d, %Y %H:%M:%S")
    if not path:
        return
    with open(path, "a", encoding="utf-8") as f:
        f.write(f"{formatted_string}\t{self.callSign}\t{msg}\n")
    sys.stdout.flush()
```

**Purpose:** Logs chat messages with timestamps.

**File format:**
```
TIMESTAMP\tCALLSIGN\tMESSAGE
```

This makes logs easy to:
- read
- parse
- replay later

### Make Chat Group

=== "Python"
```python
def make_chat_groups(self, line:str, type:str) -> str:
    return HelperFunctions().make_text_file(line, type)
```

**Purpose:** Creates or opens a chat topic.

**Note:** This function delegates work to `HelperFunctions` to avoid duplication.

### List Chat Groups

=== "Python"
```python
def list_chat_groups(self):
    """Print available group chat files."""
    
    base_dir = os.path.join(os.path.dirname(__file__), CHAT_DIR_NAME)
    os.makedirs(base_dir, exist_ok=True)
    
    files = [f for f in os.listdir(base_dir) if f.endswith(".txt")]
    if not files:
        sys.stdout.write("No chat groups yet\r\n")
        sys.stdout.flush()
        return
    
    sys.stdout.write("Chat groups:\r\n")
    for f in sorted(files):
        sys.stdout.write(f"- {os.path.splitext(f)[0]}\r\n")
    sys.stdout.flush()
```

**Purpose:** Lists all available chat topics.

### Show Connected Users

=== "Python"
```python
def show_users(self):
    users_file = os.path.join(os.path.dirname(__file__), CONNECTED_USER_TEXT_LOG)
    if not os.path.exists(users_file):
        sys.stdout.write("No connected users\r\n")
        sys.stdout.flush()
    else:
        with open(users_file, "r", encoding="utf-8") as f:
            users = []
            for row in f:
                parts = row.rstrip("\n").split("\t")
                if len(parts) >= 2 and parts[1]:
                    users.append(parts[1])
        sys.stdout.write("Connected users:\r\n" + "\r\n".join(users) + "\r\n")
        sys.stdout.flush()
```

**Purpose:** Displays currently connected users.

**How it works:**

1. Reads `connected users.txt`
2. Extracts callsigns

### Show Messages

=== "Python"
```python
def show_messages(self, path, topic):
    msg = HelperFunctions().get_file_lines(path)
    sys.stdout.write(f"Chat history for {topic}:\r\n" + "".join(msg))
    sys.stdout.flush()
```

**Purpose:** Displays the full chat history for a topic.

This mimics joining a conference and seeing previous messages.

### Delete User from the Session

=== "Python"
```python
def delete_user(self):
    """Remove this call sign from `connected users.txt` (best-effort)."""
    
    users_file = os.path.join(os.path.dirname(__file__), CONNECTED_USER_TEXT_LOG)
    if not self.callSign or not os.path.exists(users_file):
        return
    
    with open(users_file, "r", encoding="utf-8") as f:
        lines = f.readlines()
    
    kept = []
    for row in lines:
        parts = row.rstrip("\n").split("\t")
        if len(parts) >= 2 and parts[1] == self.callSign:
            continue
        kept.append(row)
    
    with open(users_file, "w", encoding="utf-8") as f:
        f.writelines(kept)
```

**Purpose:** Removes a user cleanly when they disconnect.

**Why this is critical:** Without cleanup, the system would show ghost users.

### Main Chat

=== "Python"
```python
def chat_main(self, topic="General"):
    """Run the chat REPL loop."""
    
    path = self.make_chat_groups(topic, CHAT_DIR_NAME)
    
    while True:
        line = HelperFunctions().ask_for_input(self.callSign).strip()
        
        if line in ("/H", "/h"):
            HelperFunctions().choose_menu(CHAT, "Help menu")
        elif line in ("/U", "/u"):
            self.show_users()
        elif line == "/t" or line == "/T":
            self.list_chat_groups()
        elif line.startswith("/t ") or line.startswith("/T "):
            topic = line[3:].strip()
            path = self.make_chat_groups(topic, CHAT_DIR_NAME)
            sys.stdout.write("Now you are in " + topic + " chat!\r\n")
            sys.stdout.flush()
        elif line in ("/b", "/B"):
            return
        elif line in ("/m", "/M"):
            self.show_messages(path, topic)
        elif line.startswith("/"):
            print("Invalid command")
        else:
            self.write_file(path, line)
```

**Purpose:** This is the core chat REPL loop.

**Command handling flow:**

- `/H` → show help
- `/U` → show users
- `/T` → list topics
- `/T name` → join/create topic
- `/M` → show messages
- `/B` → exit chat
- `text` → broadcast message

**LinBPQ Equivalent:** This function is effectively a node port handler.

## Program Entry Point

=== "Python"
```python
Chat(HelperFunctions().record_call_sign()).chat_main()
```

**What happens here:**

1. User is prompted for callsign
2. Callsign is recorded
3. Chat session begins

This mirrors how a packet node:
- accepts a connection
- registers the station
- hands control to an application port

## Program Code Summary

=== "Python"
```python
import sys
import os
from datetime import datetime
from typing import List

# ----------------------------
# Constants / Display strings
# ----------------------------
CHAT = [
    "/U - Show Users",
    "/T - Show Topics",
    "/T Name - Join/Create Topic",
    "/B - Return to Main",
    "/M - Show Current Topic Messages",
    "/bye",
]
CALLSIGN_LENGTH = 6
CHAT_DIR_NAME = "Group chats"
CONNECTED_USER_TEXT_LOG = "connected users.txt"
CONNECTED_USER_TEXT_LOG_RECORD = "connected user records.txt"

class HelperFunctions():

    def _chat_exit(self, callSign: str, stripped: str, stringToExit: str) -> None:
        if stripped.lower() == stringToExit and (callSign != None):
            sys.stdout.write("Goodbye \r\n")
            sys.stdout.flush()
            Chat(callSign).delete_user()
            raise SystemExit(0)

    def ask_for_input(self, callSign=None, stringToExit= '/bye') -> str:
        """Read a single line from stdin.
        If `callSign` is not None and the user types `bye`, this function:
        - prints a goodbye message
        - removes the call sign from `connected users.txt` via `Chat(callSign).delete_user()`
        - exits the program
        """
        line = sys.stdin.readline()
        if not line:
            return ''

        stripped = line.strip()
        self._chat_exit(callSign, stripped, stringToExit)

        return stripped
    
    def record_call_sign(self, logName="connected users.txt", logRecordName="connected user records.txt")->str:
        """Prompt for a call sign and record it into the connected-user files."""

        while True:
            sys.stdout.write("Enter your call sign " + "\r\n")
            sys.stdout.flush()
            callSign = self.ask_for_input().strip()
            if len(callSign) < CALLSIGN_LENGTH:
                sys.stdout.write("The call sign that you've entered is incorrect " + "\r\n")
                sys.stdout.flush()
            else:
                chat = Chat(callSign)
                users_file = os.path.join(os.path.dirname(__file__), logName)
                chat.write_file(users_file)
                users_record_file = os.path.join(os.path.dirname(__file__), logRecordName)
                chat.write_file(users_record_file)
                return callSign
            
    def choose_menu(self, names: List[str], menuType: str) -> None:
        """Print any type of menu."""

        lines = [f"{name}" for _, name in enumerate(names)]
        sys.stdout.write(f"{menuType}:\r\n" + "\r\n".join(lines) + "\r\n")
        sys.stdout.flush()
        return
    
    def make_dir(self, line, type: str) -> str:
        """Build a safe `*.txt` file path under a directory named `type`."""
        name = (line or "").strip()
        if not name:
            sys.stdout.write("Invalid name\r\n")
            sys.stdout.flush()
            return ""

        safe_name = name.replace("/", "_").replace("\\", "_")
        base_dir = os.path.join(os.path.dirname(__file__), type)
        os.makedirs(base_dir, exist_ok=True)
        return os.path.join(base_dir, f"{safe_name}.txt")
    
    def make_text_file(self, line:str, type:str) -> str:
        """Ensure a group chat file exists and return its path."""

        chat_file = self.make_dir(line, type)
        if not chat_file:
            return ""

        if not os.path.exists(chat_file):
            open(chat_file, "a", encoding="utf-8").close()

        return chat_file
    
    def get_file_lines(self, path) -> List[str]:
        """All chat messages"""
        with open(path, "r", encoding="utf-8") as f:
            msg = []
            for row in f:
                msg.append(row)
        return msg


class Chat():

    def __init__(self, callSign):
        self.callSign = callSign
    
    def write_file(self, path, msg=""):
        """Append a timestamped line to `path` in TSV format."""

        now = datetime.now()
        formatted_string = now.strftime("%B %d, %Y %H:%M:%S")
        if not path:
            return
        with open(path, "a", encoding="utf-8") as f:
            f.write(f"{formatted_string}\t{self.callSign}\t{msg}\n")
        sys.stdout.flush()
    
    def make_chat_groups(self, line:str, type:str) -> str:
        """Ensure a group chat file exists and return its path."""
        return HelperFunctions().make_text_file(line, type)

    def list_chat_groups(self):
        """Print available group chat files."""

        base_dir = os.path.join(os.path.dirname(__file__), CHAT_DIR_NAME)
        os.makedirs(base_dir, exist_ok=True)

        files = [f for f in os.listdir(base_dir) if f.endswith(".txt")]
        if not files:
            sys.stdout.write("No chat groups yet\r\n")
            sys.stdout.flush()
            return

        sys.stdout.write("Chat groups:\r\n")
        for f in sorted(files):
            sys.stdout.write(f"- {os.path.splitext(f)[0]}\r\n")
        sys.stdout.flush()

    def show_users(self):
        """Print call signs listed in `connected users.txt`."""

        users_file = os.path.join(os.path.dirname(__file__), CONNECTED_USER_TEXT_LOG)
        if not os.path.exists(users_file):
            sys.stdout.write("No connected users\r\n")
            sys.stdout.flush()
        else:
            with open(users_file, "r", encoding="utf-8") as f:
                users = []
                for row in f:
                    parts = row.rstrip("\n").split("\t")
                    if len(parts) >= 2 and parts[1]:
                        users.append(parts[1])
            sys.stdout.write("Connected users:\r\n" + "\r\n".join(users) + "\r\n")
            sys.stdout.flush()
    
    def show_messages(self, path, topic):
        """Prints all chat messages"""
        msg = HelperFunctions().get_file_lines(path)
        sys.stdout.write(f"Chat history for {topic}:\r\n" + "".join(msg))
        sys.stdout.flush()

    def delete_user(self):
        """Remove this call sign from `connected users.txt` (best-effort)."""

        users_file = os.path.join(os.path.dirname(__file__), CONNECTED_USER_TEXT_LOG)
        if not self.callSign or not os.path.exists(users_file):
            return

        with open(users_file, "r", encoding="utf-8") as f:
            lines = f.readlines()

        kept = []
        for row in lines:
            parts = row.rstrip("\n").split("\t")
            if len(parts) >= 2 and parts[1] == self.callSign:
                continue
            kept.append(row)

        with open(users_file, "w", encoding="utf-8") as f:
            f.writelines(kept)

    def chat_main(self, topic="General"):
        """Run the chat REPL loop."""

        path = self.make_chat_groups(topic, CHAT_DIR_NAME)

        while True:
            line = HelperFunctions().ask_for_input(self.callSign).strip()

            if line in ("/H", "/h"):
                HelperFunctions().choose_menu(CHAT, "Help menu")
            elif line in ("/U", "/u"):
                self.show_users()
            elif line == "/t" or line == "/T":
                self.list_chat_groups()
            elif line.startswith("/t ") or line.startswith("/T "):
                topic = line[3:].strip()
                path = self.make_chat_groups(topic, CHAT_DIR_NAME)
                sys.stdout.write("Now you are in " + topic + " chat!\r\n")
                sys.stdout.flush()
            elif line in ("/b", "/B"):
                return
            elif line in ("/m", "/M"):
                self.show_messages(path, topic)
            elif line.startswith("/"):
                print("Invalid command")
            else:
                self.write_file(path, line)

Chat(HelperFunctions().record_call_sign()).chat_main()
```