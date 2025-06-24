# QUIC: A UDP-Based Multiplexed and Secure Transport

## Streams

Streams in QUIC provide a lightweight, ordered byte-stream abstraction to an application. Streams can be unidirectional or bidirectional. Streams can be created by either endpoints, QUIC does not provide any means of ensuring ordering between bytes on different streams. QUIC allows for an arbitrary amount of data to be sent on any stream, subject to flow control constraints and stream limit.

### Stream Types and Identifiers

Streams are identified within a connection by a numeric value, referred to as stream ID. A stream ID is a 62-bit integer that is unique for all streams on a connection. The LSB identifies the initiator 0 if initiated by the client, 1 if initiated by the server. The second LSB defines if the stream is bidirectional, bit set to 0, or unidirectional, bit set to 1.

### Sending and Receiving Data

STREAM frames encapsulate data sent by an application. An endpoint uses the 
stream ID and Offset fields in STREAM frames to place data in order. Delivering an ordered byte stream requires that an endpoint buffer any data that is received out of order, up to the advertised flow control limit. The data at a given offset must not change if it is sent multiple times. Streams are an ordered byte-steam abstraction with no other structure visible to QUIC. An endpoint must not send data on any stream without ensuring that it is within the flow control limits set by its peer.

### Steam Prioritization

QUIC does not provide a mechanism for exchanging prioritization information, Instead, it relies on receiving priority information from the application.

### Operations on Streams

On the sending part of a stream, an application protocol can:
- write data, understanding when stream flow credit has successfully been reserved to send the written data;
- end the stream, resulting in a STREAM frame with the FIN bit sent; and
- reset the stream, resulting in a RESET_STREAM frame if the stream was not already in a terminal state.

On the sending part of a stream, an application protocol can:
- read data; and
- abort reading of the stream and request closure, possibly resulting in a STOP_SENDING frame.

An application protocol can also request to be informed of state changes on steams, including when the peer has opened or reset a stream, when new data is available, and when data can or cannot be written to the stream due to flow control.

## Stream States
