# Serial Port Communication Research

This document covers the fundamentals of serial port communication relevant to building a CLI application that replaces the physical console of a LifeSpan walking treadmill.

---

## 1. Serial Port Basics

### What is a Serial Port?

A serial port transmits data one bit at a time over a single communication line. The treadmill uses a **9-pin RS-232 serial connector** (DE-9), which was historically the standard for connecting peripherals like modems, printers, and industrial equipment.

### Key Pins on a 9-Pin (DE-9) Connector

| Pin | Name | Abbreviation | Direction | Purpose |
|-----|------|--------------|-----------|---------|
| 1 | Data Carrier Detect | DCD | In | Indicates remote device is connected |
| 2 | Receive Data | RxD | In | Data received from remote device |
| 3 | Transmit Data | TxD | Out | Data sent to remote device |
| 4 | Data Terminal Ready | DTR | Out | Signals this device is ready |
| 5 | Signal Ground | GND | — | Common ground reference |
| 6 | Data Set Ready | DSR | In | Remote device is ready |
| 7 | Request to Send | RTS | Out | Requests permission to send |
| 8 | Clear to Send | CTS | In | Permission granted to send |
| 9 | Ring Indicator | RI | In | Indicates incoming call (rarely used) |

For a treadmill, the most important pins are **TxD (3)**, **RxD (2)**, and **GND (5)**. Many simple devices only use these three and ignore hardware flow control entirely.

### Communication Parameters

Before two devices can talk over serial, they must agree on these settings:

- **Baud rate** — The speed of communication in bits per second. Common values: 9600, 19200, 38400, 57600, 115200. Consumer fitness equipment very commonly uses **9600 baud**.
- **Data bits** — Number of bits per character. Almost always **8**.
- **Parity** — Error-checking bit. Common values: None, Even, Odd. Most devices use **None**.
- **Stop bits** — Marks the end of a character. Usually **1**.
- **Flow control** — Mechanism to prevent data overflow. Options: None, Hardware (RTS/CTS), Software (XON/XOFF). Simple devices often use **None**.

This set of parameters is often written in shorthand like **9600 8N1** (9600 baud, 8 data bits, No parity, 1 stop bit), which is the most common configuration and a good starting guess for the treadmill.

### How Data Travels

Serial data is sent as a stream of bytes. Each byte is framed with a start bit, the data bits, optional parity, and stop bit(s):

```
Idle ──┐  ┌─Start─┬─D0─┬─D1─┬─D2─┬─D3─┬─D4─┬─D5─┬─D6─┬─D7─┬─Stop─┐  ┌── Idle
       └──┘       │    │    │    │    │    │    │    │    │    │      └──┘
```

The line sits high (idle/mark state) when nothing is being sent. A start bit pulls the line low to signal that a byte is coming.

---

## 2. Uni-Directional vs. Bi-Directional Communication

### Uni-Directional (Simplex)

One device sends, the other only listens. Example: a sensor that continuously reports readings.

```
Console ──────TxD────────> Treadmill
              (commands)
```

If the treadmill were simplex, the console would send speed commands and the treadmill would blindly obey, with no feedback.

### Bi-Directional Half-Duplex

Both devices can send and receive, but only one at a time. They take turns. This is common in request/response protocols.

```
Console ──────TxD────────> Treadmill     (console sends command)
Console <─────RxD───────── Treadmill     (treadmill responds)
```

### Bi-Directional Full-Duplex

Both devices can send and receive simultaneously over separate wires (TxD and RxD). RS-232 supports this natively.

```
Console ──────TxD────────> Treadmill     (simultaneous)
Console <─────RxD───────── Treadmill     (simultaneous)
```

### What to Expect from the Treadmill

Fitness equipment typically uses a **request/response** or **polling** pattern:

- The console periodically sends a status request or heartbeat.
- The treadmill motor controller responds with current state (speed, distance, steps, etc.).
- The console sends commands (start, stop, change speed) and the controller acknowledges.

Some devices also send **unsolicited data** — for example, the treadmill might continuously stream status at a fixed interval whether the console asks or not. Capturing traffic will reveal which pattern is used.

---

## 3. Capturing Traffic with a Serial Splitter

### Hardware Setup

You have two key pieces of equipment:

1. **Serial port splitter (Y-cable / tap)** — Splits the serial lines so a third device can passively listen.
2. **Serial-to-USB adapter** — Connects the tap to your computer.

The wiring concept:

```
                    ┌──────────────────┐
                    │  Original Cable  │
  Physical    TxD   │                  │   TxD    Treadmill
  Console ──────────┤                  ├──────── Motor
             RxD    │                  │   RxD   Controller
          <─────────┤                  ├────────
                    │                  │
                    └───────┬──────────┘
                            │
                     Tap / Splitter
                            │
                    ┌───────┴──────────┐
                    │  Serial-to-USB   │
                    │    Adapter        │
                    └───────┬──────────┘
                            │
                        Your Computer
                     (passive listener)
```

**Important considerations:**

- The splitter must be wired as a **passive tap**, meaning your computer is only connected to the RxD (receive) lines. If the splitter connects your computer's TxD to the bus, you risk injecting garbage data and confusing the treadmill.
- You may need **two USB adapters** to capture both directions independently — one tapping the console's TxD (what the console sends) and one tapping the treadmill's TxD (what the treadmill sends).
- Verify that your serial-to-USB adapter shows up as a device. On Linux, it will appear as `/dev/ttyUSB0` or `/dev/ttyACM0`.

### Verifying the USB Adapter is Detected

```bash
# Check for the device appearing when you plug it in
dmesg | tail -20

# List serial devices
ls -la /dev/ttyUSB* /dev/ttyACM* 2>/dev/null

# Check device permissions (you may need to add yourself to the dialout group)
groups $USER
sudo usermod -aG dialout $USER
# Log out and back in for group change to take effect
```

### Capturing Traffic from the CLI

Several command-line tools can read raw serial data:

#### Using `screen`

```bash
# Connect to serial port at 9600 baud
screen /dev/ttyUSB0 9600

# To exit screen: Ctrl-A then K, then confirm with Y
```

`screen` is interactive and shows data as ASCII text. Useful for a quick look, but not ideal for binary protocols.

#### Using `minicom`

```bash
# Install if needed
sudo dnf install minicom

# Run with device and baud rate
minicom --device /dev/ttyUSB0 --baudrate 9600

# Enable hex display: Ctrl-A then N
# Enable capture to file: Ctrl-A then L, then enter filename
# Exit: Ctrl-A then X
```

`minicom` supports logging to a file and hex display, making it better for protocol analysis.

#### Using `picocom`

```bash
# A simpler alternative to minicom
sudo dnf install picocom

picocom --baud 9600 /dev/ttyUSB0

# Exit: Ctrl-A then Ctrl-X
```

#### Using `cat` and `stty` (Raw Capture)

```bash
# Configure the serial port parameters
stty -F /dev/ttyUSB0 9600 cs8 -cstopb -parenb raw

# Dump raw bytes to a file
cat /dev/ttyUSB0 > capture.bin &

# View hex dump of captured data
xxd capture.bin | head -50

# Or view in real-time with hexdump
cat /dev/ttyUSB0 | xxd
```

This approach is the most flexible for binary protocols. You configure the port with `stty`, then read the raw byte stream.

#### Using `socat` (Advanced)

```bash
# Install if needed
sudo dnf install socat

# Log all traffic to a file while displaying hex
socat -x /dev/ttyUSB0,b9600,raw,echo=0 STDOUT

# Create a virtual serial port pair (useful for testing without hardware)
socat -d -d pty,raw,echo=0 pty,raw,echo=0
# This creates two linked virtual ports like /dev/pts/3 and /dev/pts/4
```

`socat` is especially useful for creating virtual serial port pairs for testing your Go application without the physical treadmill connected.

### Capture Strategy

1. **Start simple** — Use `cat /dev/ttyUSB0 | xxd` to see if any data appears when the treadmill is powered on and the console is connected.
2. **Try different baud rates** — If you see garbled data, the baud rate is probably wrong. Try 9600 first, then 19200, 4800, 2400.
3. **Press buttons and observe** — With the capture running, press each console button one at a time with pauses between. Look for patterns in the hex output that correlate with button presses.
4. **Look for packet structure** — Most protocols have a recognizable structure:
   - A start/header byte (often `0x02` STX, `0xAA`, `0xFF`, or similar)
   - A command or message type byte
   - Data payload
   - A checksum or CRC byte
   - An end/footer byte (often `0x03` ETX)
5. **Record everything** — Save raw captures with timestamps. Note what physical action you performed for each capture session.

### What Captured Data Might Look Like

Binary protocol (common for fitness equipment):

```
Raw hex:    02 06 01 00 14 00 00 1D 03
            │  │  │  │  │        │  │
            │  │  │  │  │        │  └─ End byte (ETX)
            │  │  │  │  │        └─── Checksum
            │  │  │  │  └──────────── Speed: 0x14 = 20 = 2.0 mph
            │  │  │  └─────────────── Parameter
            │  │  └────────────────── Command: 0x01 = Set Speed
            │  └───────────────────── Length: 6 bytes
            └──────────────────────── Start byte (STX)
```

ASCII protocol (less common but possible):

```
":SPD020\r\n"   →  Set speed to 2.0
":STA\r\n"      →  Start belt
":STP\r\n"      →  Stop belt
```

---

## 4. Sending Debug Commands

Once you understand the protocol from captured traffic, you can send commands to the treadmill.

### Using `echo` and Redirection

```bash
# Configure the port
stty -F /dev/ttyUSB0 9600 cs8 -cstopb -parenb raw

# Send a raw hex command
echo -ne '\x02\x06\x01\x00\x14\x00\x00\x1D\x03' > /dev/ttyUSB0

# Send an ASCII command
echo -ne ':SPD020\r\n' > /dev/ttyUSB0
```

### Using `minicom` or `picocom`

These tools let you type interactively. For hex bytes, `minicom` supports a hex send mode.

### Using `socat` for Two-Way Communication

```bash
# Interactive bidirectional connection with hex logging
socat -x /dev/ttyUSB0,b9600,raw,echo=0 READLINE
```

### Safety Warning

**Be cautious when sending commands to the treadmill.** An incorrect command could:

- Start the belt unexpectedly (trip hazard)
- Set an unsafe speed
- Put the controller into an unknown state

Always ensure the treadmill is in a safe physical state before sending test commands. Consider disconnecting the belt motor during initial protocol testing if possible, or at minimum, ensure nobody is standing on the treadmill.

---

## 5. Go Application Concerns

### Serial Port Libraries for Go

The primary library for serial communication in Go:

- **[go.bug.st/serial](https://pkg.go.dev/go.bug.st/serial)** — Well-maintained, cross-platform serial port library. Supports setting baud rate, data bits, parity, stop bits, and flow control. This is the recommended choice.

Basic usage:

```go
import "go.bug.st/serial"

mode := &serial.Mode{
    BaudRate: 9600,
    DataBits: 8,
    Parity:   serial.NoParity,
    StopBits: serial.OneStopBit,
}

port, err := serial.Open("/dev/ttyUSB0", mode)
if err != nil {
    log.Fatal(err)
}
defer port.Close()

// Write a command
_, err = port.Write([]byte{0x02, 0x06, 0x01, 0x00, 0x14, 0x00, 0x00, 0x1D, 0x03})

// Read a response
buf := make([]byte, 128)
n, err := port.Read(buf)
response := buf[:n]
```

### Important Coding Concerns

#### 1. Read Timeouts and Partial Reads

Serial `Read()` calls do not guarantee a complete message. You might get 3 bytes from one call and 6 bytes from the next, even if the device sent 9 bytes as one message. You must buffer incoming data and parse complete packets from the buffer.

```go
// Set a read timeout so Read() doesn't block forever
port.SetReadTimeout(100 * time.Millisecond)

// Buffer and accumulate reads
var buffer []byte
for {
    tmp := make([]byte, 128)
    n, err := port.Read(tmp)
    if err != nil && err != io.EOF {
        log.Fatal(err)
    }
    if n > 0 {
        buffer = append(buffer, tmp[:n]...)
        // Try to parse complete packets from buffer
        buffer = processPackets(buffer)
    }
}
```

#### 2. Device Permissions

On Linux, serial ports require the user to be in the `dialout` group (or `uucp` on some distributions). The application should produce a clear error message if it cannot open the port due to permissions.

#### 3. Graceful Shutdown

When the application exits, it must close the serial port properly. If the port is left in an odd state, the treadmill might not respond to the physical console until power-cycled. Use `defer port.Close()` and handle OS signals:

```go
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

go func() {
    <-sigChan
    // Send stop command to treadmill before exiting
    port.Write(stopCommand)
    port.Close()
    os.Exit(0)
}()
```

#### 4. Concurrency

A CLI application that reads user input and serial data simultaneously needs concurrent goroutines:

- One goroutine reading from the serial port (incoming treadmill data)
- One goroutine reading user input (CLI commands)
- A main loop coordinating between them

Go's goroutines and channels are well-suited for this pattern.

#### 5. Byte Order (Endianness)

Multi-byte values in serial protocols can be big-endian or little-endian. When parsing values like speed, distance, or time, you need to determine which byte order the treadmill uses. The `encoding/binary` package handles this:

```go
import "encoding/binary"

// If big-endian
speed := binary.BigEndian.Uint16(data[3:5])

// If little-endian
speed := binary.LittleEndian.Uint16(data[3:5])
```

#### 6. Checksum Validation

Most serial protocols include a checksum to detect transmission errors. Common schemes:

- **XOR checksum** — XOR all bytes in the packet (excluding start/end markers)
- **Sum checksum** — Sum all bytes, take the low byte
- **CRC-8 or CRC-16** — More robust but more complex

Always validate checksums on received data and compute them correctly on sent data.

#### 7. Port Discovery

The serial port device path (`/dev/ttyUSB0`) can change depending on which USB port you use or what order devices are plugged in. The `go.bug.st/serial` library provides port enumeration:

```go
ports, err := serial.GetPortsList()
for _, port := range ports {
    fmt.Println("Found port:", port)
}
```

Consider allowing the user to specify the port via a CLI flag, with auto-detection as a fallback.

#### 8. Testing Without Hardware

Use `socat` to create virtual serial port pairs for development and testing:

```bash
socat -d -d pty,raw,echo=0,link=/tmp/treadmill pty,raw,echo=0,link=/tmp/console
```

Then point your Go application at `/tmp/console` and write a simple simulator that connects to `/tmp/treadmill`. This lets you develop and test the protocol handling without the physical treadmill present.

---

## 6. Recommended Investigation Steps

1. **Identify the serial port parameters** — Connect the tap, power on the treadmill with the console attached, and try capturing at 9600 8N1 first.
2. **Determine the protocol type** — Is it binary or ASCII? Look at the raw hex dump for printable characters vs. raw bytes.
3. **Map the packet structure** — Identify start/end markers, length fields, command bytes, and checksums.
4. **Correlate commands to actions** — Press one button at a time and note what bytes appear. Build a command table.
5. **Determine if there is a heartbeat/keep-alive** — Some controllers expect periodic communication and will shut down if it stops.
6. **Test command injection** — Carefully send a known command from your computer (while the console is disconnected) and verify the treadmill responds.
7. **Document everything** — Record baud rate, packet format, command table, and timing requirements.

---

## 7. Useful References

- RS-232 standard overview: search for "RS-232" on Wikipedia for pin diagrams and signal levels
- `go.bug.st/serial` package documentation: `go doc go.bug.st/serial`
- `stty` manual: `man stty`
- `socat` manual: `man socat`
- Reverse engineering serial protocols: search for "reverse engineering serial protocol" for community guides on the general approach
