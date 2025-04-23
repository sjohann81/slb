# Single Line Bus

The Single Line Bus is a simple bus and communications protocol for
microcontrollers, based on a single line for bidirectional half-duplex
data transfers. Only the sync/data line (SDL) and a ground reference
are needed to implement a network of microcontrollers, acting as
controllers and peripherals.


## Bus electrical specification

The sync/data line is pulled up to Vcc with a 4k7 to 47k ohm resistor.
Smaller or greater pullup values can be used (1k ~ 100k ohms). On longer
cable runs (20m to 100m) smaller resistors should be used (470 ~ 1k
ohm), along with twisted pair cable. Only one pullup resistor is
needed in the bus, and for short networks (inside a board for example)
an internal (weak) pullup from a GPIO pin can be used without an
external resistor.

The protocol can be implemented using open collector outputs or push-
pull outputs with a change in the pin direction (input/output). In both
cases, the pullup resistor is responsible to pull the line high, so no
short circuits occour. An output pin in push-pull mode should always
pull the line to ground (output set to '0').

The following signals form the power supply to the pullup resistor,
the data line (SDL) and a reference ground for all devices connected
in the network:

- Vcc - 3.3v ~ 5v typical (pullup)
- Gnd - ground
- SDL - sync/data line

```
                                                             Vcc ---
                                                                  |
                                                              4k7 O
                                                                  |
SDL ---------------------------------------------------------------
          |              |
        device1        device2       ...
          |              |
Gnd ---------------------------------------------------------------
```

It is possible to mix 5v and 3.3v devices in the same bus, as long as
the pullup resistor is tied to 3.3v. This is valid as long as the 5v
device is able to read a 3.3v level as logic high, and the bus is kept
short.


## Data encoding and timing

Data in the bus is encoded using a series of line level changes (SDL in
low or high states). Devices encode data by pulling the SDL line low or
leaving it a high impedance state, being pulled high by an external
pullup resistor. A data transfer is composed by a series of symbols,
including a device select condition, start of transmission, bit
synchronization, zeroes and ones, a stop condition and an optional
acknowledge symbol by the receiving device. For a 100us max bit period
or symbol rate (around 10kbit/s data rate), the following timing is
used:

| Symbol		| Timing (typical 10kbit/s data rate)			|
| :-------------------- | :---------------------------------------------------- |
| select		| >= 800us low (>= 8x bit period)			|
| start			| 100us high						|
| sync bit		| 33us low						|
| zero			| 33us high (+ optionally 33us low)			|
| one			| 66us high						|
| stop			| 100us high						|
| ack (optional)	| 100us low or 33us low, 66us high (piggybacked data)	|

The bitstream can be easily decoded as follows: after a select
condition, wait for a start (100us high). Then, wait while the line
is low (sync bit). When the signal rises, measure how much time it stays
in this state. If the line stays high for less than 40us, it is a zero.
If it stays high between 40us to 80us, it is a one. Otherwise, its a
stop. The process continues from the sync bit condition for each bit
and stops if a stop condition occours (100us high).

For 100kbit/s, the timing is tighter as each symbol takes roughly 10us.
After a select condition (>= 80us low), wait for a start (10us high).
Wait while the line is low (sync) and measure how much time the line
stays high. A zero is encoded if the line stays high for less than 4us
and a one if it stays high from 4us to 8us.

No specific data rate is defined, although the upper bound is limited
to around 100kbit/s and the lower bound should not be < 1kbit/s. Higher
rates are possible, but not recommended.


## Collision avoidance

No special technique is mandatory for collision avoidance but a
transmitter should listen to the bus before sending data. If two symbols
worth of silence are detected (line high for 200us at 10kbit/s data
rate, a no carrier condition) the bus is assumed to be free, and the
transmission can be started. In the event of a collision, data will be
corrupted resulting in a failed start or stop sequence or an invalid
checksum. Waiting time is limited to 512 (bit periods). Up to 3 retries
can be used before a master gives up.

If collision avoidance needs to be implemented, the additional algorithm
besides carrier sense is specified:

- When no silence is detected (bus is busy), the master device waits for
a random amount of time (multiple of the bit period) before initiating
a transmission. The random delay helps to prevent multiple masters
from retransmitting their frames simultaneously causing additional
collisions.

- The random delay time is determined using an exponential back-off
algorithm. The delay time starts at a minimum value and doubles after
each collision, up to a maximum value. The maximum delay time is 
set to 1024 (bit periods). If the collision cannot be avoided after this
stage, the transmission is considered to have failed.


## Frame format

Data is transferred in 8 bit bytes. There is no need (nor is efficient)
to transfer only one byte at a time. It is up to the application to
transfer the right amount of bytes between a start and a stop condition.
A typical application of the protocol, used to transfer quantities of 8
bit bytes can be represented by (MSB transferred first):

```
(controller)
       1st byte                                2nd byte
START | b7 | b6 | b5 | b4 | b3 | b2 | b1 | b0 | ... ... | STOP |
(peripheral)
ACK
```

A typical frame consists of:

- A start condition
- 1 address / control byte
- 1 to 32 data bytes (no more than 128 bytes)
- 1 checksum byte (sum of all bytes modulo 256)
- A stop condition
- An optional acknowledge from the receiver or a pyggybacked ack

The address byte contains 7 bits used for device addressing (up to 127
devices in the same line) and 1 bit for device operation (0 - read,
1 - write). Data bytes are followed by a simple checksum, composed by
the sum of all previous bytes modulo 256.

A frame can also carry data in a piggybacked mode. In such case, data
transfers to and from a peripheral device occours in a single bus
transaction:

```
(controller)
       1st byte                                2nd byte
START | b7 | b6 | b5 | b4 | b3 | b2 | b1 | b0 | ... ... | STOP |
(peripheral)
       1st byte                                2nd byte
PBACK | b7 | b6 | b5 | b4 | b3 | b2 | b1 | b0 | ... ... | STOP |
```


## Controller and peripheral operation

The system can have multiple transmitters and receivers, which act
as masters or slaves according to the operation on the bus. Data
transfers are always initiated by a master (bus controller) and are
consisted of frames, composed by the slave (peripheral) address
information, transfer type (read or write), data payload and a checksum.

- Write - a write operation consists in a frame transmitted by a bus
master (controller). ADDRESS is the 7 bit slave (peripheral) address and
W is a '1' (write operation). A master does not wait for a transfer from
the slave, unless an acknowledge condition is specified before the
transmission.

```
(controller)
ADDRESS,W | DATA | ... | CHECKSUM
```

- Read - a read operation consists in a frame transmitted by a bus
master (controller), followed by a frame transmitted by a slave
(peripheral). ADDRESS is the 7 bit slave address and R is a '0' (read
operation). A master waits for a response from the slave containing a
matching address from the request. A slave response can be transmitted
in a separate data transfer or in piggybacked mode.

```
(controller)                         (peripheral)
ADDRESS,R | DATA | ... | CHECKSUM .. ADDRESS,R | DATA | ... | CHECKSUM
```

In a typical application, data is composed by internal addresses
and values. An example write operation contains the following data (in
this example, 16 bit addresses are used, divided in two 8 bit parts).

```
(controller)
ADDRESS,W | ADDRESS_H | ADDRESS_L | DATA0 | DATA1 | ... | CHECKSUM
```

An example read operation contains two transfers (or one piggybacked).
The first transfer (or the first part of a transfer in piggybacked
mode) is sent by the master and another (the second part of a transfer)
is sent by the slave (in this example, 16 bit addresses are used).

```
(controller)
ADDRESS,R | ADDRESS_H | ADDRESS_L | CHECKSUM
(peripheral)
ADDRESS,R | DATA | ... | CHECKSUM
```
