Description of data exchange protocol for GREE air conditioners via UART based on my reverse engineering.

I made some reverse engineering of GREE UART protocol in my free time.
I already finished the research to find a way to control and get data from my own GREE ACs and still in work for additional (maybe useful) information.
I also made external component for ESPHOME to control my ACs and they are working for a long time with my Home Assistant server.

Any corrections and improvements are welcome.

There are several types of original GREE WiFi modules such as CS532AE, CS532AH, CS532AX and other. As I understand one of the difference between them is the length of cord.
I used CS532AX for researching from AliExpress.

<span id="uart_parameters"/>
## UART PARAMETERS
4800 8E1 5V

Original module according to my router log connects to (using GREE+ or EWPE Smart apps):
- ru.dis.gree.com (162.62.5.211:5000)
- ru02.as.gree.com (162.62.16.210:16384)
- 162.62.16.210:16385
- 100.116.20.150:1027
More likely it depentds on country select in the app.

<span id="packet_structure"/>
## PACKET STRUCTURE

Every message that sent by UART includes:
1. **header**: 2 bytes (always 0x7E 0x7E);
2. **packet length (length of data packet + check digit**): 1 byte;
3. **data packet**: X bytes;
4. **check digit** ((packet length + data packet) % 256): 1 byte.

## TYPES OF PACKETS
Usually packets have different length byte so it's convenient to use this byte as a packet identifier.
Further, "outgoing packet" is the packet originally from Wi-Fi module, "incoming packet" is the packet (answer) from AC.

Summary the usual behavior after the power has sent to module:

| OUTGOING | INCOMING | REPEAT |
| 0x10 | 0x03 | x1 |
| 0x05 | 0x1a | x2 |
| 0x0e | 0x2f | x4 |
| 0x2c | 0x2f | every 300ms |

Probably, the exchange of 0x0e -> 0x2f belongs to module wifi connection and/or wifi signal strength. It needs to test and research.

Anyway, there is no need to get any answer from AC to initiate the power ON or mode change. Just send 0x2C packet with all necessary bytes set.
Also I found out that there is no need to send/receive 0x2c/0x2f packets every 300ms. My DIY ESP8266 module works well with 10 seconds interval. However the interval should be optimized to catch changes that was made from IR remote (or indoor unit power button) concurrently.

<span id="packets_description"/>
## OUTGOING PACKETS

### 0x10 (outgoing)
The packet is sending by module after it gets power. The content is the same every time but still researching. It gets corresponding incoming packet 0x03 as the answer.
Content status: unknown
Example: 10 02 00 00 00 00 00 00 01 00 28 1e 19 23 23 00 b8

### [0х2C](packet_type_0x2c) (outgoing)
Main packet. It's sent by module to AC every 300ms.
Also used as a primary packet to control AC functions.
Probably, the contents corrected after module power on and corresponding answer of AC.
Frequency: 300ms
Contents status: researching but it's enough to control AC.

### 0x0E (outgoing)
Ping packet?
Frequency: 30s
Content status: unknown
Example: 0E 03 00 00 00 00 00 00 00 00 00 00 80 7e 0f (usual, fully connected to router; sometimes 2c packet changed to it completely as a request of AC 2f response, needs more investigation)
Example: 0E 03 00 00 00 00 00 00 00 00 00 00 ==00== 7e 8f (if I blocked DHCP lease on router)
After the module is powered on within 1s (no difference from time to time):
0E 03 01 00 00 00 00 00 00 00 00 00 00 00 12
0E 03 01 00 00 00 00 00 00 00 00 00 00 7e 90

### 0x05 (outgoing)
Ping? packet. Contents the same.
It's the second sent packet after module has power on (as the answer to 0x03?).
Frequency: unknown, rare
Content status: unknown
Example: 05 04 07 00 00 10

## INCOMING PACKETS

### 0x03 (incoming)
Frequency: only as the answer to 0x10 after module power on
Content status: unknown
Example: 03 32 00 35

### 0х2F (incoming)
Description: Main incoming packet. Contains current AC information.
Frequency: only as the answer to 0x2c which has been sent every by module.
Comments: get it ONLY after module request. For example, AC doesn't send it after commands that have been sent by IR.

### 0x1A (incoming)
1A 44 01 00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 61
Frequency: unknown, rare
Content status: unknown



The following is a detailed description of the packets 0x2C & 0x2F.


<span id="packet_type_0x2c"/>
### 0x2C PACKET
---
Packets of this type are sent to AC by Wi-Fi module.

##### Byte 0, 1 (HEADER)
VEALUE: 0x7E 0x7E

##### Byte 2 (PACKET LENGTH)
VALUE: 0x2C

##### Byte 3 (?)
always 0x01

##### Byte 4 (?)
always 0x00
used as power state in incoming 2f?

##### Byte 5 (?)
always 0x00

##### Byte 6 (?)
always 0x00

##### Byte 7 (CMD)
0x00 - get update from AC w/o making changes
0xAF - as a command to AC to change current setting including ON/OFF state

##### Byte 8 (MODE + FAN)
0x10 - set AC to turn OFF
0x80 - AUTO MODE (0x80 - auto FAN SPEED, 0x81 - low, 0x82 - medium, 0x83 - high)
0x90 - COOL MODE (0x90 - auto FAN SPEED, 0x91 - low, 0x92 - medium, 0x93 - high)
0x98 - ?
0xA1 - DRY MODE (only low FAN SPEED, no other settings are available in my AC)
  // fanonly 0-1-2-3
0xB0 - FAN ONLY MODE (0xB0 - auto FAN SPEED, 0xB1 - low, 0xB2 - medium, 0xB3 - high)
0xC0 - HEAT MODE (0xC0 - auto FAN SPEED, 0xC1 - low, 0xC2 - medium, 0xC3 - high)
0x01 - ?
0x21 - ?


##### Byte 9 (TEMP)
0x00 - 16c
0x10 - 17c
0x20 - 18c
0x30 - 19c
0x40 - 20c
0x50 - 21c
0x60 - 22c
0x70 - 23c
0x80 - 24c
0x90 - 25c
0xA0 - 26c
0xB0 - 27c
0xC0 - 28c
0xD0 - 29c
0xE0 - 30c
Comment: I use temperature in Celsius. Maybe different when Fahrenheit used.



##### Byte 10 (DISPLAY+TURBO)


##### Byte 11 (?)
always 0x02

##### Byte 12 (SWING)
00 - don't make any changes to current set?
14 - vertical
41 - horizontal (if supported)
11 - both
44 - off

##### Byte 13 (DISPLAY)
0x00 - off ?

##### Byte 14 (?)
always 0x00

##### Byte 15 (ECO?)
0x00
0x40 

##### Bytes 16-21 (?)
always 0x00

##### Byte 22 (?)
0x00
0x01
0x02
0x03
0x04
0x05
(where i got these values?...)

##### Bytes 20-27 (?)
always 0x00

##### Byte 28 (?)
0x00 0x1a 0x1c
the same as in response?

##### Byte 29-34 (?)
always 0x00


##### Byte 35 (?)
0x00
0x03 for one time after strange byte3 incoming 2f packet

##### Byte 36 (?)
0x00
0x7f for one time after strange byte3 incoming 2f packet

##### Bytes 37-40 (?)
always 0x00

##### Byte 41 (WIFI?)
0x04 for one time after strange byte3 incoming 2f packet
0x04 - only when off
0x08 - off/on
0x0c - off/on

##### Byte 42 (?)
always 0x00

##### Byte 43 (?)
usual 0x02
but sometimes 0x00

##### Byte 44-45 (?)
always 0x00

##### Byte 46 (CRC)
control digit (sum of all packet bytes w/o two header bytes % 256)

---
--------------------------------------------------------------------------------------------------------------------------------------------
### 0x2F PACKET
Response packet from AC to 0x2F packet from module.

##### Byte 0-1 (HEADER)
VALUES: 0x7E 0x7E

##### Byte 2 (LENGTH)
VALUES: 0x2F

##### Byte 3 (?)
VALUES: 0x31 (usual packet), 0x33 (fully unknown packet )
but sometimes 0x33. examples:
2F 33 00 00 40 00 08 20 19 0A 00 10 00 14 14 5B 04 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 94
2F 33 04 00 40 00 08 20 19 0A 00 10 00 14 14 5B 04 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 98

##### Byte 4 (POWER STATE)
VALUES: 0x00 - off, 0x04 - on

##### Byte 5 (?)
VALUES: 0x00

##### Byte 6 (?)
VALUES: 0x40
on/off status?

##### Byte 7 (?)
VALUES: 0x00

##### Byte 8 (MODE + FAN)
current state of AC fan? analog to 0x2c packet? see description there
also unknown values:
0x01 - ?
0x11 - ? 
0x12 - ?
0x98 - ?


##### Byte 9 (TEMPERATURE)
current state of set temperature, analog to the same byte in 0x2c packet.
from 0x00 (16c) to 0xE0 (30c)
(minimal temp 16 + 16 for every next degree)
there were also some unknown exceptions:
0x88 - ?
0x89 - ?
0x98 - ?
0x99 - ?
0xA8 - ?
0xA9 - ?
0xAA - ?
0xAC - ?
0xAD - ?

##### Byte 10 (DISPLAY + TURBO?)
current state of AC temperature? analog to 0x2c packet?
0x00 - AC display off
0x02 - AC off, red power sign is on
?0x03 - off after TURBOO COOL?
0x04 - AC on, display OFF
0x06 - AC on, display of some temperature (AUTO,COOL, DRY, VENT), usual mode, NOT TURBO
0x07 - COOL mode and TURBO ON
0x0E - HEAT mode, usual mode, NOT TURBO
0x0F - HEAT mode ON + TURBO MODE
(according to manual TURBO works only with COOL or HEAT modes as well as SE mode; SE (save energy) mode not discovered yet)

##### Byte 11 (?)
always 0x02

##### Byte 12 (SWING)
swing status of AC? analog to 0x2c packet? (without 0x80, 0xA0)
0x00 - no swing on screen
0x10 - 12345 - all range
(when?) 0x11 -?
0x20 - 1 - top
0x30 - 2 - upper-middle
0x40 - 3 - middle
0x50 - 4 - below-middle
0x60 - 5 - bottom
0x70 - 345 - middle to bottom
0x90 - 234 - about the middle
0xb0 - 123 - middle to top

##### Byte 13 (DISPLAY OF TEMP)
0x00 - (no house icon on remote)
0x10 - current temp set (empty house icon on remote)
0x20 - current temp in room (thermometer sign in the house icon)
0x20 - also when in TURBO COOL mode (mode = 90)
0x30 - current outdoor temp (thermometer sign outside the house icon)
0x40 - TURBO?  (COOL & HEAT)
???0x60 - I FEEL ON
0x40 - I FEEL ON

##### Byte 14 (?)
always 0x00

##### Byte 15 (?)
0x00 usual
0x40 - very rare, something with ECO mode?

##### Byte 17-19 (TIMER SET?)
00 00 00 - off
0e 01 10 - timer to OFF after 0.5h
0c 03 10 - timer to OFF after 1h
0a 05 10 - timer to OFF after 1.5h
08 07 10 - ?
06 09 10 - ?
04 0b 10 - ?
02 0d 10 - ?
00 0f 10 - ?
0e 10 10 - ?
0c 12 10 - ?
0a 14 10 - ?
08 16 10 - ?
06 18 10 - ?
04 1a 10 - ?
02 1c 10 - ?
00 1e 10 - ?
0e 1f 10 - ?
0a 23 10 - ?
08 25 10 - ?
00 5a 10 - ?
02 58 10 - ?
04 56 10 - ?
06 54 10 - ?
08 52 10 - ?

##### Bytes 20-27 (?)
0x00 always

##### Byte 28 (???)
value after I FEEL mode is on. temp from the remote?
still here when IFEEL off by IR
0x00
0x1a - 26
0x1c - 28
0x1d - 29
looks like temperature in Celsius

##### Bytes 29-40 (?)
0x00 always

##### Byte 41 (?)
0x00
0x80 (changing when AC ON from 0x00 to 0x80 and back several times) may be when something received from IR remote?

##### Bytes 42-45 (?)
0x00 always

##### Byte 46 (INDOOR TEMPERATURE)
0x3E - 16 =  indoor temp 22c OR 0x3E -> 62 - 40 = 22c
possible values: 0x3e, 0x41, 0x42, 0x43, 0x44, 0x45
changed to quite different temp when in I FEEL mode, looks like it changes to IR temperature sensor

##### Bytes 47-48 (?)
always 0x00

##### Byte 49 (CRC)
control digit (sum of all packet bytes w/o two header bytes % 256)


Anyway, you must test anything that I researched before using. Your AC may be different and behave different as well.

If you have any suggestion and updates you are welcome.