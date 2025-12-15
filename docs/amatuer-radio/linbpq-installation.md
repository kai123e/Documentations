# LinBPQ Installation on Ubuntu

## Overview
LinBPQ is a flexible packet node/BBS for amateur radio operators who want to bridge TCP/IP-based services with RF links. This guide focuses on Ubuntu systems and walks through:

- Preparing a dedicated system user with the right serial/audio permissions.
- Installing the Direwolf-based software TNC stack and radio interface dependencies.
- Downloading the LinBPQ binary plus the optional web interface so you can administer the node remotely.

You can follow the sections sequentially on a fresh host, or cherry-pick the steps you still need on an existing setup.

## Install LinBPQ
### Create User
Creating a dedicated service account keeps LinBPQ isolated from other workloads and ensures the process always owns the serial/audio devices it needs. If you already have a non-root operator account in the `sudo` and `dialout` groups, you can reuse it and skip to the next section.

Steps below assume you are logged in as `root` (use `su -l` first on minimal installs). Install `sudo` if it is missing, add a `linbpq` user, and grant the minimal groups required for packet work.


=== "Bash"
    ``` bash
    su -l
    apt install sudo 
    adduser linbpq
    usermod -aG sudo linbpq
    usermod -a -G dialout linbpq
    su linbpq
    ```
!!! Warning "Execution"

    For the remaining sections, run commands as the `linbpq` user. Use `sudo` only when the step explicitly requires elevated privileges.

## LinBPQ Installation
With the user in place, install the tooling LinBPQ depends on: archive utilities, capability helpers, audio libraries, and Direwolf support packages. The commands below refresh the package index, install the prerequisites, and create a working directory for LinBPQ builds.

=== "Bash"
    ``` bash
    sudo apt update 
    sudo apt-get install wget unzip libcap2-bin screen 
    sudo apt install libpcap0.8-dev libasound2-dev libz3-4 zlib1g libminiupnpc17 
    mkdir ~/linbpq 
    cd ~/linbpq
    ```

### Download the LinBPQ Binary
Fetch the current `linbpq64`. Keeping this download step separate makes it easy to update later without reinstalling every dependency.

=== "Bash"
    ``` bash
    sudo apt-get update
    wget http://www.cantab.net/users/john.wiseman/Downloads/Beta/linbpq64
    ```

### Set Executable Bit
Mark the freshly downloaded binary as executable so it can be launched without invoking an interpreter explicitly.

=== "Bash"
    ``` bash
    sudo chmod +x linbpq64
    ```

### Grant Low-port and Raw-socket Access
LinBPQ needs privileged networking capabilities to bind to packet ports, interact with raw sockets, and manage PTT control without running as `root`. The `setcap` command below grants only the required kernel capabilities.

=== "Bash"
    ``` bash
    sudo setcap "CAP_NET_ADMIN=ep CAP_NET_RAW=ep CAP_NET_BIND_SERVICE=ep" linbpq64
    ```

## Downloading the Web Interface
The optional HTML interface is handy for monitoring ports and BBS sessions from a browser. Create a subfolder, download the archive, and unpack it in place.

=== "Bash"
    ``` bash
    mkdir HTML
    cd HTML
    wget http://www.cantab.net/users/john.wiseman/Downloads/Beta/HTMLPages.zip
    unzip HTMLPages.zip
    ```

## LinBPQ Configuration File
LinBPQ reads its runtime settings from `bpq32.cfg`, which must live in the same directory as the `linbpq64` binary. The sample below defines a simple two-port node (RF + Telnet) that hands off audio to Direwolf. Feel free to adjust call signs, aliases, coordinates, and frequencies to match your station before saving.

Start Direwolf in a second terminal so audio is already flowing when LinBPQ tries to connect. Serial TNC users should confirm their device path (for example `/dev/ttyUSB0`) before continuing.

Back in the LinBPQ directory, create or edit the configuration file with your preferred editor:

=== "Bash"
    ``` bash
    cd ~/linbpq
    nano bpq32.cfg
    ```

Paste the baseline configuration and edit the highlighted lines so they reflect your station. Double-check every `PORT` block to ensure the paths, speeds, and calls match your hardware and network plan.

=== "bpq32.cfg"
    ``` linenums="1" hl_lines="2 3 39 47 74 77 78 7 13 17 21"
    SIMPLE
    LOCATOR=-37.64318,145.06853
    NODECALL=VK3ZUW-7
    NODEALIAS=WHI

    CTEXT:
    Welcome to the VK3ZUW-7 LinBPQ Node.
    HMNDE> BBS CHAT CONNECT BYE INFO NODES ROUTES PORTS USERS MHEARD
    ***
    FULL_CTEXT=0

    IDMSG:
    LinBPQ Node CITY OF WHITTLESEA AU VK3ZUW-11 BBS VK3ZUW-12 CHAT
    ***

    BTEXT:
    LinBPQ BBS/CHAT @ 439.100
    ***

    INFOMSG:
    Sysop VK3ZUW.
    BBS to connect to the BBS.
    CHAT to connect to CHAT server.
    ***

    EnableM0LTEMap=0                                                                                                                                                            
                                                                                                                                                                                
    IDINTERVAL=10                                                                                                                                                               
    BTINTERVAL=15
    NODESINTERVAL=25

    PACLEN=236
    BBS=1
    NODE=1
    HIDENODES=1

    PORT
        PORTNUM=1
        ID=439.100 MHz 1200 bps
        TYPE=ASYNC
        SPEED=19200
        PROTOCOL=KISS
        IPADDR=127.0.0.1
        TCPPORT=8001
        RESPTIME=1500
        MHEARD=Y
        BCALL=VK3ZUW-7
        MINQUAL=100
        KISSOPTIONS=NOPARAMS
        CHANNEL=A
        FULLDUP=0
        NOKEEPALIVES=1
    ENDPORT

    PORT
        PORTNUM=2
        ID=Telnet
        DRIVER=Telnet
        QUALITY=0
        CONFIG
        LOGGING=1
        DisconnectOnClose=0
        SECURETELNET=1
        LOGGING=1
        TCPPORT=8100
        FBBPORT=8101
        IPV4=1
        HTTPPORT=8080
        LOGINPROMPT=Username:
        PASSWORDPROMPT=Password:
        MAXSESSIONS=15
        CloseOnDisconnect=1
        CTEXT=Welcome to LinBPQ Telnet Server\nEnter ? for list of commands\n\n
        USER=TECHSCHOOL,TECHSCHOOL,VK3ZUW,"",sysop
    ENDPORT

    APPLICATION 1,BBS,,VK3ZUW-11,MHBBS,255
    APPLICATION 2,CHAT,,VK3ZUW-12,MHCHT,255
    LINMAIL
    LINCHAT
    ```

!!! Warning "Customise Before Running"

    Update every highlighted field in `bpq32.cfg` before going live:
    - Identity: replace `NODECALL`, `NODEALIAS`, `BCALL`, and the `APPLICATION` call signs with your licensed calls.
    - Location and RF info: set `LOCATOR`, `ID`, `BTEXT`, and beacon text (`CTEXT`, `IDMSG`, `INFOMSG`) so they describe your site and frequency accurately.
    - Port specifics: ensure each `PORT` block uses the correct `IPADDR`, `TCPPORT`, `SPEED`, `USER=` credentials, and `KISSOPTIONS` for your Direwolf/Telnet setup.
    - Security: choose unique Telnet usernames/passwords in the `USER=` line and keep them private.
    Save the file (`Ctrl+O` in `nano`) and exit (`Ctrl+X`) once everything matches your station plan.

### Validate and Test the Configuration
1. In Terminal A, launch Direwolf with your KISS/TCP configuration:
    ```bash
    direwolf
    ```
2. In Terminal B, start LinBPQ (ensure in linbpq user):
    ```bash
    cd ~/linbpq
    sudo ./linbpq64
    ```
3. From another machine or local machine, verify connectivity:
    - `telnet <host> 8100` for the application port
    - `telnet <host> 8080` to load the built-in web interface

!!! note "Connection Tips"

    Use `127.0.0.1` when testing locally; otherwise point to the LAN/WAN IP of the machine running LinBPQ and Direwolf. If you changed `TCPPORT`, `FBBPORT`, or `HTTPPORT`, update the test commands accordingly.


## Terminology Reference

??? info "Identity & Messaging"
    - `SIMPLE`: Uses the condensed node menu and hides advanced prompts for small BBS installs.
    - `LOCATOR`: Latitude/longitude (or Maidenhead pair) so mapping tools know where the node sits.
    - `NODECALL` / `NODEALIAS`: RF call sign and short alias announced to other nodes.
    - `CTEXT`: Greeting displayed when a user connects over RF or Telnet; terminated by `***`.
    - `FULL_CTEXT`: When set to `1`, the entire `CTEXT` repeats at each prompt; `0` shows it once per session.
    - `IDMSG`: Text transmitted during ID frames so neighbouring nodes recognise you.
    - `BTEXT`: Beacon payload typically containing frequency and service summary.
    - `INFOMSG`: Message returned when someone issues the `INFO` command.
    - `EnableM0LTEMap`: Controls whether the node uploads heard-station data to the public map maintained by M0LTE (off in this sample).

??? info "Timing & Broadcasts"
    - `IDINTERVAL`: Minutes between automatic ID frames.
    - `BTINTERVAL`: Minutes between beacon transmissions that use `BTEXT`.
    - `NODESINTERVAL`: Minutes between node list broadcasts over RF.

??? info "Core Behaviour"
    - `PACLEN`: Maximum AX.25 frame length in bytes (236 is safe for 1200-baud links).
    - `BBS`: Enables (`1`) or disables (`0`) the mailbox application.
    - `NODE`: Enables the node-switching service so users can reach other stations through you.
    - `HIDENODES`: Hides downstream nodes from the public list when set to `1`.

??? info "RF Port (PORTNUM=1)"
    - `PORT` / `ENDPORT`: Wraps the definition of a single transport.
    - `PORTNUM`: Unique number LinBPQ uses when routing to this port.
    - `ID`: Friendly description shown in logs and status screens.
    - `TYPE`: `ASYNC` means the port talks to a serial/TCP TNC as opposed to an HDLC card.
    - `SPEED`: Serial bitrate for the KISS connection or pseudo-serial socket.
    - `PROTOCOL`: `KISS` instructs LinBPQ to speak the KISS framing protocol to Direwolf.
    - `IPADDR` / `TCPPORT`: Host and port of the Direwolf KISS listener (often 127.0.0.1:8001).
    - `RESPTIME`: Milliseconds LinBPQ waits for acknowledgements before retrying.
    - `MHEARD`: When `Y`, maintains the "stations heard" list for this port.
    - `BCALL`: Call sign used in beacons originating on this port.
    - `MINQUAL`: Minimum neighbour quality score required before accepting their routes.
    - `KISSOPTIONS`: Extra directives for Direwolf/KISS; `NOPARAMS` keeps the link simple.
    - `CHANNEL`: Distinguishes multiple radios driven from one TNC (A/B/C...).
    - `FULLDUP`: Set to `0` for half-duplex simplex work, `1` for full-duplex links.
    - `NOKEEPALIVES`: Disables periodic keepalive frames to chatty TNCs.

??? info "Telnet/Service Port (PORTNUM=2)"
    - `DRIVER`: `Telnet` enables the built-in Telnet/HTTP server stack.
    - `QUALITY`: Advertised path quality for this logical port (0 keeps it off RF routing tables).
    - `CONFIG` ... `ENDPORT`: Nested block containing driver-specific settings.
    - `LOGGING`: When `1`, records logins and session info to the console/log file.
    - `DisconnectOnClose` / `CloseOnDisconnect`: Control how Telnet sockets tear down when the client exits.
    - `SECURETELNET`: Forces login prompts before giving node access.
    - `TCPPORT`: Telnet port for interactive BBS/node work (8100 in this sample).
    - `FBBPORT`: Optional forwarding port compatible with FBB-style mail exchanges.
    - `IPV4`: Enables IPv4 listening (set to `0` to disable).
    - `HTTPPORT`: Port that serves the HTML status pages (8080 here).
    - `LOGINPROMPT` / `PASSWORDPROMPT`: Custom strings shown during authentication.
    - `MAXSESSIONS`: Maximum concurrent Telnet users.
    - `CTEXT` (inside CONFIG): Telnet-specific greeting; `\n` sequences add new lines.
    - `USER=USERNAME,PASSWORD,CALL,"",LEVEL`: Defines login credentials and privilege level (`sysop` grants full control).

??? info "Applications & Mail"
    - `APPLICATION n,NAME,,CALL,ALIAS,QUAL`: Maps menu numbers (1=BBS, 2=CHAT) to specific services or call signs.
    - `LINMAIL`: Enables the built-in LinBPQ mailer required for BBS message storage and forwarding.
    - `LINCHAT`: Enables the  built-in LINBPQ chat required

## References
* [TheModernHam LinBPQ BBS Install on Ubuntu](https://themodernham.com/install-linbpq-bbs-packet-node-on-debian-ubuntu-and-raspbian/)