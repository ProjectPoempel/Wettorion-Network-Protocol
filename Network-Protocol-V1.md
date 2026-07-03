# Wettorion Network Protocol V1

**Current Version-Value:** ```1```

The protocol is TCP based.  
  
**Offset** and **Size** are in ***bytes***.  
Endianess: **Big-Endian**.  
  
The protocol works with a one-time registration (until the next connection). The client registers itself with the **Hello** packet and it's **Station ID**. The server internally associated the connection with the **Station ID**, so that the client doesn't need to send it's **Station ID** with every packet.

## Possible Sensor Values

Sensor values are either specified via a bitmask or a integer value.

| Bitmask Placement | ID / Integer Value | Name
|---|---|--|
| 0 | 0x01 | Temperature
| 1 | 0x02 | Humidity
| 2 | 0x03 | Air Pressure (hPa)
| 3 | 0x04 | PM10
| 4 | 0x05 | PM2.5
| 5 | 0x06 | CO2

## Base Packet Structure

> Min. packet size is 4 bytes.

| Offset | Size | Value | Description |
|---|---|---|---|
| 0 | 1 | 1..255 (?), u8 | Protocol-Version
| 1 | 1 | 0..255 (?), u8 | Packet ID
| 2 | 2 | ?, u16 | Packet Payload Length
| ... | ... | ... | ...

## ID Ranges

0x**0** - Initialization / Registration packets  
0x**1** - Ping / Connection status packets  
0x**2** - Status packets  
0x**3** - Data packets  
0x**4** - Server action packets (server does something and tells the client / tells the client to do something)  
0x**5** - Client action packets (client does something and tells the server)  

---

## Packets

### Overview

**Server to Client**

- [Ping](#ping) ```0x10```
- [Data ACK](#data-ack) ```0x31```
- [Data FAIL](#data-fail) ```0x32```
- [Hello](#hello) ```0x01```
- [Disconnect](#disconnect) ```0x40```
- [Config Push](#config-push) ```0x41```

**Client to Server**

- [Hello](#hello) ```0x01```
- [Capabilities](#capabilities) ```0x02```
- [Pong](#pong) ```0x11```
- [Data](#data) ```0x30```
- [Live Data](#live-data) ```0x33```
- [Status](#status) ```0x20```
- [Batch Data](#batch-data) ```0x34```
- [Client Disconnect](#client-disconnect) ```0x50```

### Server to Client

#### Ping

> Packet ID: 0x10

Server sends a ping request to a client. The client should respond within 30 seconds. If it fails to do so, the server will terminate the connection. A [Ping](#ping) packet received while in maintenance mode can be ignored.

#### Data ACK

> Packet ID: 0x31

The server sends this packet back after the received data from a [Data](#data) or [Batch Data](#batch-data) packet was successfully stored in the database.

#### Data FAIL

> Packet ID: 0x32

The server sends this packet back after the received data from a [Data](#data) or [Batch Data](#batch-data) packet was rejected or an error occurred while storing it.

#### Disconnect

> Packet ID: 0x40

The server sends this packet if the client should disconnect gracefully by itself. This also may indicate a server shutdown. The specific reason is given via a bitmask and an additional and optional message.

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 2 | ?, u16 | Bitmask
| 6 | 1 | ?, u8 (max. 255 chars) | Message Length (0 = no message attached)
| 7 | ? | ? | The attached message in UTF-8. (not 0 terminated)

**Possible Reasons:** (ordered)

| Bitmask Placement | Description
|---|---|
| 0 | Server shutdown
| 1 | Server rebooting
| 2 | Server Maintenance
| 3 | Client with **Station ID** is already connected
| 4 | Banned
| 5 | Unknown / Custom

#### Config Push

> Packet ID: 0x41

This packet contains a new config the receiving client should immediately use.  
The payload consists of BSON (binary json).  
Configs may vary and are *(currenty)* not explicitly defined.

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 2 | ?, u16 | Size of BSON
| 6 | ? | ? | The binary data

---

### Client to Server

#### Hello

> Packet ID: 0x01

A client sends a hello packet right after the TCP connection is established. If it doesn't arrive within 1 minute, the connection is terminated by the server. Any packet besides this will be rejected until the server sends a hello response packet. If the connection succeeds the server will echo the same [Hello](#hello) packet back. Unknown / invalid **Station IDs** will be ignored. After a certain amount of times the IP-Address will be blocked for some time.

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 2 | ?, u16 | Station ID

#### Capabilities

> Packet ID: 0x02

This packet contains information on the sensors and features of the client station.  
This packet can only be sent after the ([Hello](#hello) packet) registration.

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 2 | ?, u16 | Entry Count
| 6 | ? | ... | Entries

**Entry Layout:**

| Offset | Size | Value | Description |
|---|---|---|---|
| 0 | 1 | ?, u8 | Value ID (SEE: [Possible Sensor Values](#possible-sensor-values))
| 1 | 1 | ?, u8 (max. 255 chars) | Sensor Name Length (required)
| 2 | ? | ? | The attached name in UTF-8. (not 0 terminated)

#### Pong

> Packet ID: 0x11

A client sends a ping response to the server. This packet should only be sent after the client recieves a ping request.

#### Data

> Packet ID: 0x30

A client sends a data packet to the server. This packet contains the collected station data. It does not need to have every value. Present values are specified via the bitmask. Values should appear in the order they are defined in the bitmask.  
Data packets do not need to be send immediately after the values were captured. They can be stored and sent later.

The server may reject values if it determines that they are invalid or impossible.  
The server will respond with a [Data ACK](#data-ack) or [Data FAIL](#data-fail) packet.  
ACK = all stored, FAIL = some or all rejected

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 4 | ?, u64 | UNIX Timestamp
| 8 | 8 | ?, u32 | Bitmask
| 16 | ? | ? | Values

**Possible Values:** (ordered)

| Bitmask Placement | Name | Size | Datatype | Precision | Packing |
|---|---|---|---|---|---|
| 0 | Temperature | 2 | s16 | 2 decimal places | float32 as s16 (value * 100)
| 1 | Humidity | 2 | u16 | 2 decimal places | float32 as u16 (value * 100)
| 2 | Air Pressure (hPa) | 4 | float32 | 2 decimal places | float32
| 3 | PM10 | 2 | u16 | 2 decimal places | float32 as u16 (value * 100)
| 4 | PM2.5 | 2 | u16 | 2 decimal places | float32 as u16 (value * 100)
| 5 | CO2 | 2 | u16 | no decimal places | raw u16

#### Batch Data

> Packet ID: 0x34

A client should send this packet when providing a large amount of datasets to the server insead of sending many [Data](#data) packets.  
  
The datasets are normal [Data](#data) packets without the packet header packed tightly inside this packet.  
The server will respond with one [Data ACK](#data-ack) or [Data FAIL](#data-fail) packet for all datasets.

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 2 | ?, u16 | Dataset count
| 6 | ... | ... | The datasets

#### Live Data

> Packet ID: 0x33

A client may send live data to the server. The containing values may not (but can) be older than 5 minutes.  
The structure of this packet is the same as the [Data](#data) packet.

#### Status

> Packet ID: 0x20

A client may send a status packet to the server to report it's current state. Statuses are specified via a bitmask. A status message may be attached to a packet to provide additional information.

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 8 | ?, u64 | Bitmask
| 12 | 1 | ?, u8 (max. 255 chars) | Message Length (0 = no message attached)
| 13 | ? | ? | The attached message in UTF-8. (not 0 terminated)

**Possible Statuses:** (ordered)

| Bitmask Placement | Description
|---|---|
| 0 | Fault 
| 1 | Sensor error
| 2 | Low battery
| 3 | Low storage
| 4 | Rebooting
| 5 | Maintenance (all following packets apart from [Hello](#hello) and [Status](#status) (only with maintenance status) packets will be ignored, no ping requests will be sent)
| 6 | Unknown / Custom (a message should be provided)

#### Client Disconnect

> Packet ID: 0x50

A client should send this packet before terminating the connection to let the server gracefully close it's internal connection, to avoid waiting for a ping timeout. A reason and an additional, optional message can be given.

| Offset | Size | Value | Description |
|---|---|---|---|
| 4 | 2 | ?, u16 | Bitmask
| 6 | 1 | ?, u8 (max. 255 chars) | Message Length (0 = no message attached)
| 7 | ? | ? | The attached message in UTF-8. (not 0 terminated)

**Possible Reasons:** (ordered)

| Bitmask Placement | Description
|---|---|
| 0 | Client shutdown
| 1 | Client rebooting
| 2 | Sleep
| 3 | Maintenance
| 4 | Unknown / Custom