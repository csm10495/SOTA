# Serial Open Transport Apparatus
The *Serial Open Transport Apparatus* also known as *SOTA* is a transport-level specification designed to allow the transmission of specification-agnostic commands over a serial bus. This is an early version of this specification and is therefore subject to change. Commands not outlined explicitly in this specification are outside of this document's scope, though are still required to use the [Standard Payload Header](#standard-payload-header) and normal-sized buffers.

## Definitions
This section describes various terminology used throughout the document.

#### Ack
A acknowledgement byte sent by the [Slave](#slave) to the [Master](#master) to notify that the sending of the command thus far has not resulted in an error. This code is designated by the [Command Acknowledgements Table](#command-acknowledgements-table). Some acknowledgement bytes may directly correspond with a required action taken by the [Master](#master).

#### Buffer
Often used to designate the hard-coded transfer size of 128 bytes until a [Command Acknowledgement](#command-acknowledgement) shall be sent by the [Slave](#slave) to the [Master](#master). The final two bytes of the Buffer is always a checksum of the entirety of preceeding 126 bytes. The only transfers that do not need to consist of 128 byte buffers are that of the [Command Acknowledgement](#command-acknowledgement).

#### Command
Set of bytes sent to a [Slave](#slave) with the expectation of a particular response taken by that [Slave](#slave).

#### Command Acknowledgement
Either an [Ack](#ack) or [Nack](#nack) sent from the [Slave](#slave) to the [Master](#master) upon the completion of the transmission of a [Buffer](#buffer) from the [Master](#master) to the [Slave](#master). This initial transfer is 2 bytes in size. The first byte is the [Ack](#ack) or [Nack](#nack), the second byte is the last byte received in the 128 byte buffer. Note that a Command Acknowledgement may specify another read to perform a specific action.

#### Data Direction
A command is considered to either be primarily a transfer of data from either the [Master](#master) to the [Slave](#slave) (sometimes called a read) or the [Slave](#slave) to the [Master](#master). In both cases, the command must specify the amount payload data to transfer in the DTL field of the [Standard Payload Header](#standard-payload-header).

#### Endianness
Unless otherwise noted, all numeric byte-diagrams and fields are considered to be little-endian. All [String](#string)s are big-endian.

#### Filler Bytes
Used to fill a given buffer to 126 bytes, so that adding on a 2 byte checksum, completes the 128 byte [Buffer](#buffer)

#### Master
The device that is sending commands to the [Slave](#slave).

#### Nack
A byte sent by the [Slave](#slave) to the [Master](#master) to notify that the sending of the command has run into an issue. These codes are designated by the [Command Acknowledgements Table](#command-acknowledgements-table).

#### Payload
A set of [Buffers](#buffer) used to send a command or reap a command response.

#### Sequence
A set of multiple [Command](#command)s sent for a particular flow. For example: sending a command, processing the [Command Acknowledgement](#command-acknowledgement) and then reading back the resulting data.

#### Slave
The device that is receiving commands from the [Master](#master).

#### String
Sequence of bytes denoting ASCII text. All strings in all commands are in a big-endian format. Strings shall not be byte-swapped.

#### Sync Packet
4 byte value set to 0xBB77AAFE in little-endian. This must be the first 4 bytes of all [Standard Payload Headers](#standard-payload-header).

## Commands
All commands designated in this section shall be supported by any device supporting this SOTA Specification. The minimum payload for a command is the entirety of the 128 byte buffer. The last two bytes of any buffer must be a checksum of the preceeding 126 bytes. A command may span multiple buffers, though the total size after the [Standard Payload Header](#standard-payload-header), including checksums must be specified in the Data Transfer Length field.

### Command Table
| Operation Code | Command Name | Data Direction  |
|----------------|--------------|-----------------|
| 0x0001         | [Identify](#identify) | Read |
| 0x2000         | [Abort](#abort)       | Non-Data |

* All SOTA read (slave -> master) commands must have an Operation Code less than 0x1000
* All SOTA write (master -> slave) commands must have an Operation Code less than 0x2000 and greater than or equal to 0x1000
* All SOTA non-data commands must have an Operation Code less than 0x3000 and greater than or equal to 0x2000
* All Vendor-Unique read (slave -> master) commands must have an Operation Code less than 0x4000 and greater than or equal to 0x3000
* All Vendor-Unique write (master -> slave) commands must have an Operation Code less than 0x5000 and greater than or equal to 0x4000
* All Vendor-Unique non-data commands must have an Operation Code less than 0x6000 and greater than or equal to 0x5000
* All other Operation Code regions are reserved.

#### Standard Payload Header
All payloads must initially begin with a [Standard Payload Header](#standard-payload-header), also referred to as an SPH. DTL must only be set by the operand sending data. For example: If a master is reading data from the slave, it shall set DTL to 0. Though when the slave is sending the data, it's SPH must have DTL set to the data length it is sending (minus the SPH size itself). There is only an SPH in both directions on a read command as a write does not have the slave return data past the acknowledgement packet.

For commands with a data transfer from master to slave, DTL must be set by the master if there is a data transfer needed. For commands with a data transfer from slave to master, the master shall set DTL to 0 and based of the transfer direction of the command, the master should know to read the SPH after the acknowledgement packet.

| Bytes | Abbreviation | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|-------|--------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 00:03 | SP           | Sync Packet. This value must be 0xBB77AAFE. The intention of this packet is to ensure that the master and slave are synchronized.  If a command does not start with the designated Sync Packet, the slave shall reject the command via a Nack with Invalid Sync Packet.                                                                                                                                                                                                                                                      |
| 04:07 | OPC          | Operation Code. This value shall designate the specific command that is being invoked. Every command shall have a unique Operation Code.                                                                                                                                                                                                                                                                                                                                                                                                             |
| 08:15 | DTL          | Data Transfer Length. The number of bytes to be written from sender to receiver after the completion of the standard payload header. This number does not need to be divisible by the buffer size (128 bytes). If not divisible, the sender shall fill the remaining data in the current 128 byte buffer with 0xCC and the 2 byte checksum. This value shall not include bytes in the Standard Payload Header. On Non-data commands this must be set to 0, though the buffer must be completely filled out.|

#### Command Sequence
Often sending a single command will be useful since the master would not be able to qualify success or failure of the given command. This section gives an for how commands can be sequenced together.

##### Identify Sequence (Example)
For the following sequence assume that there were no transmission errors.
1. Master writes the Identify SPH, with DTL set to 0, with filler bytes and checksum to the slave.
3. Slave verifies the given [Standard Payload Header](#standard-payload-header) and the ending checksum.
4. Slave sends back an acknowledgement and follows it with all data required by the [Standard Payload Header](#standard-payload-header) followed by the [Identify Return Payload Definition](#identify-return-payload-definition). DTL is set to the size of the Identify Return Payload.
5. Master reads the acknowledgement byte sent by the slave. Since the acknowledgement byte was "Good", reads 16 bytes to get the [Standard Payload Header](#standard-payload-header), then read DTL worth of data to complete the command transfer. The master reads until the final-needed buffer is completed in order to validate the return checksum.

### Identify
The Identify command is used to query the device for various properties including it's serial number and general status.
#### Identify Return Payload Definition
| Bytes | Abbreviation | Description                                                                                                                                         |
|-------|--------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| 00:03 | VER          | Version for the SOTA specification supported. To denote support for this specification, this field shall be set to 1.                               |
| 04:07 | VUVER        | Vendor Unique Version supported. This field is vendor unique and is outside the scope of this specification. It is recommended that this field directly correspond to vendor unique commands supported by the device.                                       |
| 08:27 | MOD          | Model String for the given device.                                                                                                                  |
| 28:47 | PR           | Product Revision for the given device. This could be a firmware identification string.                                                              |
| 48:67 | SS           | Serial String for the given slave. This value is often intended to be unique for a given Product Revision.                                          |
| 68:87 | ST           | Status String for the given slave. The slave shall place "Healthy" in this location if the device is considered to be in a fully operational state. |

### Abort
The Abort command is used to request that the slave abort the current command. The current command must be in progress and the slave must be currently in its busy-loop. After sending this command, it is possible that the device may begin the abort process but still return a Busy status. This command is a best-effort and it should not be considered a guarantee that the currently executing command will be definitively aborted. The success of the eventual abort can be determined via receipt of the Failure Due To Abort acknowledgement. The master may send this command multiple times to request the abort. If the slave receives it multiple times and aborts the current command, it shall continue to return Failure Due To Abort until a command other than Abort is sent.

## Command Acknowledgements
During the transmission process, it is possible that an error could be detected by the slave that should percolate back to the master. Therefore, with every individual buffer transfer of 128 bytes, sent by the master, the slave must reply with one of the below command acknowledgement bytes followed by the last non-checksum byte of data from the buffer.

## Command Acknowledgement Payload
| Byte | Abbreviation | Description                                                                  |
|------|--------------|------------------------------------------------------------------------------|
| 00   | ACK          | Ack or Nack byte for the previous command or buffer. See [Command Acknowledgements Table](#command-acknowledgements-table).                          |
| 01   | LBB          | Last byte of the buffer data that has led to this ack. This byte is intended to be used for [Communication Synchronization](#communication-synchronization). |

### Command Acknowledgements Table
| Ack Byte    | Meaning             | Extended Meaning                                                      |
|------------:|--------------------:|----------------------------------------------------------------------:|
|        0x01 | Good                | The command has completed successfully, or the buffer was received and validated successfully. |
|        0x02 | Busy                | Processing the command and can still reply. Will send a Busy Ack Byte every second, while the current command is processed, followed by a single Good or Failure byte after completion. The master shall read until something besides Busy is returned, or a general timeout occurs. An [Abort](#abort) may be issued during the described busy loop. |
|        0x03 |      Pause Sequence | After this action, the device may not be able to respond. Master shall read 2 bytes right after getting Pause Sequence. This 2 byte value corresponds with the amount of time (in seconds) to wait to continue the current sequence. Master shall pause and expect a Good or Failure acknowledgement byte from the slave waiting after sleeping for the given wait time. The last byte after the wait time, is the next acknowledgement byte. Pause Sequences may not be chained together.
|        0xF9 | Failure Due To Abort | The command was aborted during it's busy-loop and has therefore failed. |
|        0xFA | Buffer Error        | The last buffer of the sequence was not processed due to a small-scale suspected transmission error. Consider retrying the buffer. |
|        0xFB | Sequence Error      | The last command of the sequence was not processed due to a large-scale suspected transmission error. Consider retrying the sequence. |
|        0xFC | Invalid Operation Code | The sent operation code is not supported on this device. Do not retry. |
|        0xFD | Invalid Sync Packet | The sync packet did not match 0xBB77AAFE. The command was not processed. Retry the buffer. |
|        0xFE | Invalid Checksum    | The buffer checksum did not match the data transferred. The command was not processed. Retry the buffer. |
|        0xFF | Failure             | The command was processed and has failed. Do not retry. |

Any ack byte yielded by the slave to the master that is not defined in the above chart can lead to undefined behavior.

#### Pause Sequence
Pause Sequence is a special acknowledgement byte in that it requires that the master read an additional 2 bytes of non-checksum-validated data from the slave. This is a special consideration to allow the slave to specify that it will be unable to pass back additional ack bytes for a specific amount of time (at most) though will be busy and unable to pass more return bytes. A Pause Sequence may not specify another Pause Sequence as the following acknowledgement, however it pass a Busy and then Pause Sequence again, though this is not normally recommended. Both Pause Sequence and Busy are intended to denote situations where the slave is processing an outstanding command.

## Communication Synchronization
When the master begins to communicate with the slave, it is possible that for various reasons, the two may not be in sync in terms of number of bytes to send to get a Command Acknowledgement. The expected method to resolve this is described here.
1. Master flushes its receive buffer.
2. Master sends a 128 byte buffer with data that shall start at 0 and increment upward. This buffer is unique in that in contains no checksum or SPH. And is expected to therefore be rejected via an Invalid Sync Packet ack byte.
```
// In hex:
00 01 02 03 04 05 .... 7D 7E 7F 80
                                ^^
// In decimal:                  128
```
3. If the LBB field of the resulting acknowledgement corresponds with 128, the master and host are synchronized. If not, try again from step 1, but instead of sending 128 bytes in step 2, send LBB bytes. Now the master and slave should be synchronized as the new LBB should be 128.