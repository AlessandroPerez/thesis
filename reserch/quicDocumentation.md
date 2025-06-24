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

Unidirectional streams used ether the sending or rreceiving state machine, depending on the stream type and endpoint role. Bidirectional streams use both state machines at both endpoints.

### Sending Stream States

This state machine shows the states for the part of a stream that sends data to a peer.

       o
       | Create Stream (Sending)
       | Peer Creates Bidirectional Stream
       v
   +-------+
   | Ready | Send RESET_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Send STREAM /             |
       |      STREAM_DATA_BLOCKED  |
       v                           |
   +-------+                       |
   | Send  | Send RESET_STREAM     |
   |       |---------------------->|
   +-------+                       |
       |                           |
       | Send STREAM + FIN         |
       v                           v
   +-------+                   +-------+
   | Data  | Send RESET_STREAM | Reset |
   | Sent  |------------------>| Sent  |
   +-------+                   +-------+
       |                           |
       | Recv All ACKs             | Recv ACK
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Recvd |                   | Recvd |
   +-------+                   +-------+

The sending part of a strem that the endpoint initiates is opened by the application. The "Ready" state represents a newly created state taht is able to accept data from the application. Stream daataaa might be buffered in this sate in prepartioning for sending.

Sending the first STREAM or STREAM_DATA_BLOCKED frame causes a sending part of a stream to enter the "Send" state.The sending part of a bidirectional stream initiated by a peer starts in the "Ready" state when the receiving part is created.

In the "Send" state, an endpoint transmits -- and retransmits as necessary -- stream data in STREAM frames. The endpoint respects the flow control limits set by its peer and continues to accept and process MAX_STREAM_DATA frames. An endpoint in the "Send" state generates STREAM_DATA_BLOCKED frames if it is blocked from sending by stream flow control limits. 

After the application indicates that all stream data has been sent and a STREAM frame containing the FIN bit is sent, the sending part of the stream enters the "Data Sent" state. From this state, the endpoint only retransmits stream data as necessary. The endpoint does not need to check flow control limits or send STREAM_DATA_BLOCKED frames for a stream in this state. MAX_STREAM_DATA frames might be received until the peer receives the final stream offset. The endpoint can safely ignore any MAX_STREAM_DATA frames it receives from its peer for a stream in this state.

Once all stream data has been successfully acknowledged, the sending part of the stream enters the "Data Recvd" state, which is a terminal state.From any state that is one of "Ready", "Send", or "Data Sent", an application can signal that it wishes to abandon transmission of stream data. Alternatively, an endpoint might receive a STOP_SENDING frame from its peer. In either case, the endpoint sends a RESET_STREAM frame, which causes the stream to enter the "Reset Sent" state.

An endpoint MAY send a RESET_STREAM as the first frame that mentions a stream; this causes the sending part of that stream to open and then immediately transition to the "Reset Sent" state. Once a packet containing a RESET_STREAM has been acknowledged, the sending part of the stream enters the "Reset Recvd" state, which is a terminal state.

### Receiving Steam States

The states for the part of a steam that receives data form a peer are:

       o
       | Recv STREAM / STREAM_DATA_BLOCKED / RESET_STREAM
       | Create Bidirectional Stream (Sending)
       | Recv MAX_STREAM_DATA / STOP_SENDING (Bidirectional)
       | Create Higher-Numbered Stream
       v
   +-------+
   | Recv  | Recv RESET_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Recv STREAM + FIN         |
       v                           |
   +-------+                       |
   | Size  | Recv RESET_STREAM     |
   | Known |---------------------->|
   +-------+                       |
       |                           |
       | Recv All Data             |
       v                           v
   +-------+ Recv RESET_STREAM +-------+
   | Data  |--- (optional) --->| Reset |
   | Recvd |  Recv All Data    | Recvd |
   +-------+<-- (optional) ----+-------+
       |                           |
       | App Read All Data         | App Read Reset
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Read  |                   | Read  |
   +-------+                   +-------+

The receiving part of a stream initiated by a peer is created when the first STREAM, STREAM_DATA_BLOCKED, or RESET_STREAM frame is received for that stream. For bidirectional streams initiated by a peer, receipt of a MAX_STREAM_DATA or STOP_SENDING frame for the sending part of the stream also creates the receiving part. The initial state for the receiving part of a stream is "Recv".

For a bidirectional stream, the receiving part enters the "Recv" state when the sending part initiated by the endpoint enters the "Ready" state.

An endpoint opens a bidirectional stream when a MAX_STREAM_DATA or STOP_SENDING frame is received from the peer for that stream. Receiving a MAX_STREAM_DATA frame for an unopened stream indicates that the remote peer has opened the stream and is providing flow control credit. Receiving a STOP_SENDING frame for an unopened stream indicates that the remote peer no longer wishes to receive data on this stream. Either frame might arrive before a STREAM or STREAM_DATA_BLOCKED frame if packets are lost or reordered.

Before a stream is created, all streams of the same type with lower-numbered stream IDs MUST be created. This ensures that the creation order for streams is consistent on both endpoints.

In the "Recv" state, the endpoint receives STREAM and STREAM_DATA_BLOCKED frames. Incoming data is buffered and can be reassembled into the correct order for delivery to the application. As data is consumed by the application and buffer space becomes available, the endpoint sends MAX_STREAM_DATA frames to allow the peer to send more data.

When a STREAM frame with a FIN bit is received, the final size of the stream is known. The receiving part of the stream then enters the "Size Known" state. In this state, the endpoint no longer needs to send MAX_STREAM_DATA frames; it only receives any retransmissions of stream data.

Once all data for the stream has been received, the receiving part enters the "Data Recvd" state. This might happen as a result of receiving the same STREAM frame that causes the transition to "Size Known". After all data has been received, any STREAM or STREAM_DATA_BLOCKED frames for the stream can be discarded.

The "Data Recvd" state persists until stream data has been delivered to the application. Once stream data has been delivered, the stream enters the "Data Read" state, which is a terminal state.

Receiving a RESET_STREAM frame in the "Recv" or "Size Known" state causes the stream to enter the "Reset Recvd" state. This might cause the delivery of stream data to the application to be interrupted.

Sending a RESET_STREAM means that an endpoint cannot guarantee delivery of stream data; however, there is no requirement that stream data not be delivered if a RESET_STREAM is received. An implementation MAY interrupt delivery of stream data, discard any data that was not consumed, and signal the receipt of the RESET_STREAM. A RESET_STREAM signal might be suppressed or withheld if stream data is completely received and is buffered to be read by the application. If the RESET_STREAM is suppressed, the receiving part of the stream remains in "Data Recvd".

Once the application receives the signal indicating that the stream was reset, the receiving part of the stream transitions to the "Reset Read" state, which is a terminal state.

### Permitted Frame Types
