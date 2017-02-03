# Simple Serial Slave-Master Protocol (S3MP)

`S3MP` is a simple protocol for defining communication between two devices. It is a [Master/Slave](https://en.wikipedia.org/wiki/Master/slave_(technology)) protocol.

## Motivation

This protocol was created for being used on personal projects where two devices need to exchange data through a serial channel. Think of a Raspberry Pi sending commands to an Arduino that has a lot of sensors plugged to it.

## Transport

Messages are encoded using [Consistent Overhead Byte Stuffing](https://en.wikipedia.org/wiki/Consistent_Overhead_Byte_Stuffing) (COBS), as detailed on [this paper](http://www.stuartcheshire.org/papers/cobsforton.pdf).

Encoded messages are terminated by the marker `0x00`. In sum, the transmitted data is the following:

```
COBS(message) + CHECKSUM(COBS(message)) + MARKER
```

The following is a small variation of the original C code, that returns the number of bytes read/written:

```c
#define FinishBlock(X) (*code_ptr = (X), code_ptr = dst++, code = 0x01)

size_t cobsEncode(const byte *ptr, size_t length, byte *dst) {
  const uint8_t *original_dst = dst;
  const uint8_t *end = ptr + length;
  uint8_t *code_ptr = dst++;
  uint8_t code = 0x01;

  while (ptr < end) {
    if (*ptr == 0) FinishBlock(code);
    else {
      *dst++ = *ptr;
      if (++code == 0xFF) FinishBlock(code);
    }
    ptr++;
  }

  FinishBlock(code);

  return dst - original_dst - 1;
}

size_t cobsDecode(const byte *ptr, size_t length, byte *dst) {
  const uint8_t *original_dst = dst;
  const uint8_t *end = ptr + length;
  while (ptr < end) {
    int code = *ptr++;
    for (int i = 1; i < code; i++) *dst++ = *ptr++;
    if (code < 0xFF) *dst++ = 0;
  }
  return dst - original_dst - 1;
}
```


## Checksum

A checksum field is used as part of the exchanged messages in order to validate that a message was fully and correctly received.

The algorithm used is documented [here](https://en.wikipedia.org/wiki/Longitudinal_redundancy_check), being the following its pseudocode:

```
Set LRC = 0
For each byte b in the buffer
do
    Set LRC = (LRC + b) AND 0xFF
end do
Set LRC = (((LRC XOR 0xFF) + 1) AND 0xFF)
```

It is applied to the string of `Code (Command or Response) + Address + Data`.

An example of implementation is the following:

```c
byte crc8(byte *data, int len) {
  byte lrc = 0;
  const byte *end = data + len;
  while (data < end) {
    lrc = (lrc + *data++) & 0xFF;
  }
  return ((lrc ^ 0xFF) + 1) & 0xFF;
}
```


## Addresses

Sensors, actuators, and the device itself are represented by different addresses:

- `0x00` addresses the slave host;
- address `0xFF` means all sensors/actuators on the slave;
- Any other address (`0x00` - `0xFE`) represents sensors/actuators.


## Counters

Counters should have values between `0x01` and `0xFF`. Counters with `0x00` should be used on `PUSH` notifications or when the Slave replies with errors (i.e. `BAD_REQUEST` or `INVALID`). When a command is understood by the Slave, it should reply using the same counter value. If a reply with a counter value different from the expected is received, the command can be marked as failed.


## Commands

Commands are messages sent from master to slave. They can be sent to the slave host, or any of the sensors/actuators available. Every command issued by the master must be followed by a response from the slave. New commands cannot be issue while a response wasn't received. Every command can be responded with an `ERROR` message, detailing the issue that happened.

The message format is the following:

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  | Command Code  |    Address    |    Counter    |    Data ...
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     ... (variable size)                                          |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The fields are:

- `Checksum`: checksum of message, as described on the previous sections;
- `Command Code`: the byte code of the issued command;
- `Address`: the address on the slave that this message is intended to;
- `Counter`: a number identifying the message;
- `Data`: an optional field with data to be sent to the address, specific to the addressed resource.

### `STATUS` [0x00]

- Description: Check the status of the slave
- Responses:
    - `ACK`, if slave is OK, optinally with the timestamp of when the device was started

### `DESCRIBE` [0x02]

- Description: Setups the connection between the devices
- Details:
    - Expected response content is defined only for the slave host
- Responses:
    - When sent to the slave host, must return an `ACK` listing the sensors/actuators, where names are separated by semicolon (`;`), and their position indicates the respective address. For instance:

```
   DATA: temp;humidity;time;
ADDRESS: -1-- -2------ -3--
```

### `GET` [0x10]

- Description: Get the current value of a sensor/actuator
- Responses:
    - `ACK` with the sensor/actuator current value

### `SET` [0x11]

- Description: Set the current value of a sensor/actuator
- Responses:
    - `ACK` with empty data, if successful

### `INVERT` [0x12]

- Description: If the sensor/actuator is a boolean value (`true`/`false`), invert its value from `false` to `true`, and vice-versa.
- Responses:
  - `ACK` with the new value

### `SUBSCRIBE` [0xA0]

- Description: subscribe to a sensor, receiving `PUSH` notifications later
- Details:
    - Must reply with an `ACK` with initial data
    - A `PUSH` is sent only on data change
- Responses:
    - `ACK` with initial data

### `UNSUBSCRIBE` [0xA1]

- Description: Cancels an existing subscription to a given address
- Responses:
    - `ACK`, if unsubscribed 

### `RESET` [0xFF]
  
- Description: Reset the configurations for the address
- Details:
    - When issued to the slave host, can cause a full device reset, needing to issue a new `DESCRIBE`
- Responses:
    - No response should be expected


## Responses

Responses are messages from the slave to master, that answer commands issued by the master. The base format is the following:

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  | Response Code |    Address    |    Counter    |    Data ...
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     ... (variable size)                                          |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Its fields are:

- `Checksum`: checksum of message, as described on the previous sections;
- `Response Code`: the byte code of this response;
- `Address`: the address on the slave that this message is intended to;
- `Counter`: the number used to identify the original command;
- `Data`: an optional field with data to be sent to the address, specific to the addressed resource.

### `ACK` [0x00]
  
- Description: notifies the master that a message was received and successfully handled

### `BAD_REQUEST` [0x40]

- Description: answer that the handler could not understand the given command data

### `INVALID` [0x41]

- Description: warns the master that the last command received didn't match checksums

### `NOT_FOUND` [0x44]

- Description: warns the master that the given address doesn't exists

### `NOT_IMPLEMENTED` [0x45]

- Description: let the master know that the given address doesn't know how to handle the given command code

### `ERROR` [0x50]

- Description: sends back a generic error, where the data contains a message detailing it

### `PUSH` [0xA0]

- Description: sends updates from a subscribed address
- Details
    - `PUSH` is fire-and-forget and can be sent at any time
    - `Counter` is set to 0


## Patterns

- Establish the connection by sending `STATUS` commands
    - Send a `RESET` when master starts
    - Poll the slave with `STATUS` messages, till an `ACK` is received

- Monitor the Slave with `STATUS` commands
    - Send `STATUS` every defined interval
    - If the slave didn't reply a number of times, consider it to be down

- Use `SUBSCRIBE` together with regular `GET`, to prevent lost messages from not updating the data
    - `SUBSCRIBE` to an address
    - Use timeouts and send a `GET` if no `PUSH` was received in a while
    - Use `PUSH` messages as soon as received

