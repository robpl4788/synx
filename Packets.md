# Packet Format
Packets are sent as most significant byte first (big endian).

# Packet Structure

| Command | Request | Data | CRC |
|---------|---------|------|-----|
| Nibble 0:7-4 | Nibble 0:3-0 | 13 Bytes 1 -> 13 | 2 Bytes 14 -> 15 |

When referring to a specific bit in this page we say byte:bit. 15:7 is the most significant bit of the most significant byte (for our 16 byte packets). We can also refer to an entire byte at once with just its index. 0 is the least significant byte for example. Ranges of bytes are denoted with a - (eg 1:3-4), and across bytes are depicted with -> (eg 1:3 -> 2:5) and are inclusive on both ends.

Any bits or bytes not used should be packed with zeros.

# Data Types

String is UTF8 zero filled string.

Float is :
 - 1 Byte = Open Compute Project OFP8 E4M3
 - 2 Byte = IEEE 754 binary16
 - 4 Byte = IEEE 754 binary32

Floats should be valid numbers, using NaN or the like is not recommended and may result in undesirable behaviour.

When given size options, floats, ints and uints are only valid for 1, 2, or 4 byte options. 

Only strings are valid for max length options. 

Enums are only valid for 0 -> 60 options.

# Commands

| Command | Meaning |
|---------|---------|
| 0 | Send Control Signals |
| 1 | Send Logged Values |
| 2 | Request/Send Logged Value Names |
| 3 | Bind Request/Accept |
| 4 | Setting Name |
| 5 | Apply Setting |
| 6 | Get Setting Value |
| 7 to 15 | Reserved |

Reserved values should not be used and may be defined later, perhaps for ranging or other purposes.

# Command and Request Byte

Command is what this packet being sent is.

Request is the desired response.

Request is not enforced, other responses are allowed.

The receiver should almost always do as requested. There are no known reasons for responding differently to that requested.

The transmitter can do things other than requested, primarily that would be requesting or applying settings when the receiver requests a control signal.

| Position | Name | Description |
| ---- | ---- | ---- |
| 0:7-4 | Command | |
| 0:3-0 | Request | |



# CRC

To calculate the CRC, prepend the UID to the packet (excluding the CRC), then use CRC16 IBM 3740 (Also known as CRC16 CCITT FALSE) as described in https://reveng.sourceforge.io/crc-catalogue/all.htm

# Send Control Signals (0)

6x 12 bit analog channels (9 Bytes)
1x Button byte

| Position | Name | Description |
| ---- | ---- | ---- |
| 1:7 -> 2:4 | Left X | |
| 2:3 -> 3:0 | Left Y | |
| 4:7 -> 5:4 | Left Trigger | |
| 5:3 -> 6:0 | Right X | |
| 7:7 -> 8:4 | Right Y | |
| 8:3 -> 9:0 | Right Trigger | |
| 10:1 | Left Bumper | |
| 10:0 | Right Bumper | |

# Send Logged Values (1)

| Position | Name | Description |
| ---- | ---- | ---- |
| 1:7-6 | Logged Value Number of Bytes  | n = 00:1, 01:2, 10:4, 11:12 |
| 1:5-0 | Logged Value Index (0-60) |  |
| 2 -> (2 + n - 1) | Logged Value | |

Number of Bytes, Index and Value sets can be repeated to fill 13 bytes total.

Fill with 0xFF (which is invalid as a Logged Value Index).

# Request/Send Logged Value Names (2)

| Position | Name | Description |
| ---- | ---- | ---- |
| 1:7-6 | Logged Value Type  | 00:uint, 01:int, 10:float, 11:string |
| 1:5-0 | Logged Value Index | 0 -> 60 |
| 2 -> 13 | Logged Value Name | Request for name if all 0's |

When the transmitter requests the name of a logged value, use type as 00, fill name with 0's.

Logged Value Names are strings and cannot be empty (so cannot be all 0's).

# Bind Request/Accept (3)
If not in bind mode, do not respond to a bind request.

If no telemetry is requested, a bind accept is still required, but no normal responses should be given (so the only packet such a receiver would ever send is a Bind Response).

A bind request packet uses a UID of 0 for CRC purposes. Bind accepts should use the correct UID for CRC.

| Position | Name | Description |
| ---- | ---- | ---- |
| 1 | Protocol Version | As uint. Version described in this document is 1 |
| 2 -> 5 | Unique Id | As 4 byte unsigned int |
| 6:1 | Telemetry yes/no | Yes = 1, No = 0 |
| 6:0 | Accept/Request | Accept = 1, Request = 0 |

The Transmitter should choose a fully random non-zero UID.



# Setting Name (4)
Request/provide the name of a specific setting or enum value.

| Position | Name | Description |
| ---- | ---- | ---- |
| 1:7 | Request? | 1 = Requesting, 0 = Providing |
| 1:6 | Enum? | 1 = Enum, 0 = Setting |
| 1:5-0 | Setting Index | 0 -> 60 |
| 2:7-6 | Setting Value Type | 00:enum, 01:int, 10:float, 11:string |
| 2:5-0 | Enum Index | 0 -> 60 |
| 3 -> 13 | Name |  |

When requesting the name of a setting, use Setting Value Type = 00, Enum Index = 0.

When providing the name of a setting that is represented as an enum, Enum Index is the number of enum values.


# Apply Setting (5)

| Position | Name | Description |
| ---- | ---- | ---- |
| 1:7-6 | Setting Number of Bytes  | n = 00:1, 01:2, 10:4, 11:11 |
| 1:5-0 | Setting Index | 0 -> 60 |
| 2 -> (2 + n - 1) | Setting Value | |

Number of Bytes, Index and Value sets can be repeated to fill 13 bytes total.

Fill with 0xFF (which is invalid as a Logged Value Index).

Setting Index 0 is the total number of available settings. This is read only, and writes should not be attempted.

Typically a transmitter will request Apply Setting when commanding Apply Setting. This should echo the packet, allowing the transmitter to validate the settings were communicated succesfully.

# Get Setting Value (6)
Request/provide the current value of a specific setting.

| Position | Name | Description |
| ---- | ---- | ---- |
| 1:7-6 | Setting Number of Bytes  | n = 00:1, 01:2, 10:4, 11:11 |
| 1:5-0 | Setting Index | 0 -> 60 |
| 2:0 | Request? | 1 = Requesting, 0 = Providing |
| 3 -> (3 + n - 1) | Setting Value | |