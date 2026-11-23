# General Design rules
Every packet should be interpretable in a vacuum with no information about the state of the system. This does not mean no understanding of the system is needed to understand the packet, just that the information can be decoded and stored correctly. For example, you should be able to recieve a logged value and store it, even though you do not know the name of the logged value.

Every packet should be loseable without critical failure. This means packets should not communicate a state change that affects future packets.

# SX1280 Config

```
SF7
812 kHz
FEC 4/6
Preamble 12
High Sensitivity Mode
Tx Power = 12.5dBm
No Hardware CRC
16 Byte Packet
```

# General Theory of Operation
Transmitter sends out packets at 40-50Hz. Each time the reciever gets a packet it responds with a packet.

In normal operation, the transmitter will cycle through an 80 band loop, changing band before transmitting each new packet. The band loop is defined by the UID and is constant for a given bind.

If a reciever does not recieve packets for a given timeout period (not defined here, different recievers can use different timeouts) the reciever should wait on a given band until it sees a new valid packet from the transmitter, at which point it can resume operation. The transmitter always continues on the band loop.

## Timing
The transmitter sends out packets at a constant rate as defined by the Target Transmission Period.

The reciever will need to transition to the next band before the transmitter begins sending it's packet on that band. If the reciever gets a packet it should transition immediately after sending it's response. If the reciever hasn't heard a packet, it should typically transition anyway based on the target transmission period under the assumption that it has dropped the packet (the time needed to transition bands is hardware dependent so left to specific implementations). 

The reciever should synchronise it's internal clock with the transmitters based on when the reciever gets the packet, to ensure that it doesn't loose sync with the transmitter.

## Packets
There are 4 main groups of packets:

| Type | Description |
|----|-----------|
| Control Signals | The transmitter tells the receiver what to do. |
| Logged Values | The receiver tells the transmitter what it's doing. |
| Settings | The transmitter configures how the receiver should behave. |
| Bind | The transmitter and receiver work out how to talk to each other. |

Further details on how packets are implemented are available in Packets.md

## Binding
During binding the reciever can be configured with or without telemetry. This allows additional recievers which can listen in to transmitter and telemetry communication without interfering with it. Typical applications include redundant/seperate recievers or allowing a computer to directly log the communication.

### Binding Procedure
1. Transmitter and Reciever are put into bind mode. 
2. Reciever listens on the bind band. 
3. Transmitter sends Bind Requests on bind band at 50 Hz. 
4. If a reciever hears a Bind Request it responds with a Bind Accept, and is locked to the Bind Requests UID.
5. If the transmitter hears a Bind Accept it is bound, and transitions into normal operating mode, skipping forward in it's band loop to the bind band, such that the first normal packet it sends is on the bind band. 
6. The reciever waits on the bind channel. If it hears a new bind request with a differnt UID it will be ignored. If it hears a new bind request with the same UID it will resend the acceptance. If it hears a control signal on the bind channel with a matching UID the reciever is bound.

# Expected Values

To enable transmitters to display in a standard way we expect some standard values.

## Expected Logged Values

| Logged Value Index | Name | Type | Size | Notes |
|--------------------|------|------|------|-------|
| 0 | Vbat | Float | 2 Bytes | Battery Voltage |

## Expected Settings

| Setting Index | Name | Type | Size | Notes |
|---------------|------|------|------|-------|
| 0 | Setting Cnt | Int | 1 Byte | Read Only |
| 1 | Rx Name | String | 11 Bytes |

# Random Number Generator Usage

See  Random Number Generator.md for description of the RNG we use. Since the RNG is based on state, it is critical that it is used in the exact order specified or the transmitter and reciever won't match.

The RNG is seeded with the UID from the bind.

The RNG is used only for the initial setup of the random band table and transmission period. This should happen in the Transmitter when starting communication with a specific receiver, including immediately after the bind is completed, and in the Receiver when starting and immediately after the bind is completed.

We shuffle the 80 bands with the Fisher-Yates shuffle as follows:

```
function shuffle_1_to_80():
    list = [1, 2, 3, ..., 80]

    for i from 79 down to 1:
        j = next_u32() mod (i + 1)
        swap(list[i], list[j])

    return list
```

We then produce a target transmission period between 20 and 25ms as:

```
period = 20.0 + 5.0 * (float(next_u32()) / float(2^32 - 1))
```
