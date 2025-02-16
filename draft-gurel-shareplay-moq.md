---
title: "Synchronized Social Video-on-Demand (VoD) Viewing with Media over QUIC"
abbrev: shareplay-moq 
docname: draft-gurel-shareplay-moq-latest
date:
  year: 2025
  month: February
  day: 16
category: exp #or std, info, bcp?

ipr: trust200902
area: Applications and Real-Time
submissionType: IETF
workgroup: MOQ
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: Z. Gurel
    name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr

  -
    ins: A. Begen
    name: Ali Begen
    organization: Networked Media
    email: ali.begen@networked.media

  -
    ins: A. Pehlivanoglu
    name: Ahmet Pehlivanoglu
    organization: Ozyegin University
    email: ahmet.pehlivanoglu@ozu.edu.tr

  -
    ins: K. Bekmez
    name: Kerem Bekmez
    organization: Ozyegin University
    email: kerem.bekmez@ozu.edu.tr
  
  -
    ins: B. Yumakogullari
    name: Basar Yumakogullari
    organization: Ozyegin University
    email: basar.yumakogullari@ozu.edu.tr

informative:
  MoQTransport: I-D.ietf-moq-transport

  
--- abstract

This draft presents an approach to Synchronized Social Video-on-Demand (VoD) Viewing with Media over QUIC. Extending current Media Over QUIC Transport (MOQT) protocol to enable synchronized Video-on-Demand (VoD) functionality. This approach adapts MoQ’s push-content architecture to include interactive features such as pause, resume, and seek. Addressing limitations in current implementations.

--- middle

# Introduction

Media Over QUIC (MoQ) is a novel protocol designed for efficient media streaming over the QUIC transport protocol ({{?RFC9000}}). While MoQ has shown promise for live-edge streaming, its current design lacks support for essential VoD functionalities, such as pause, resume, and seek. Leaving gaps in its applicability for interactive media consumption. This document extends MoQ to enable synchronized VoD playback, introducing mechanisms that allow users to have more control over the media playback.

This document outlines the architectural designs, control mechanisms, and synchronization logic implemented to achieve these objectives. This document's innovations include new MOQT tracks for synchronisation control, an example media publisher model for serving on-demand video, and a Leader-Follower client infrastructure to demonstrate the potential of MoQ for enhancing synchronized VoD consumption.

## Terms and Definitions

{::boilerplate bcp14-tagged}

Client:
: An entity that initiates a MoQT session and requests media content.

Server:
: An entity that accepts incoming MoQT sessions and provides media content.

Endpoint:
: A Client or Server participating in a MoQ session.

Producer:
: An endpoint that publishes media content to a MoQ network.

Consumer:
: An endpoint that subscribes to and receives media content from a MoQ network.

Relay:
: An intermediary node in a MoQ network that forwards media content between Producers and Consumers.

Track:
: A logical media stream (e.g., video or audio). Tracks contain a sequential series of Groups and serve as the primary subscribable entity in MoQT ({{?MoQTransport, Section 2.4}}).

Group:
: A temporally sequential set of Objects within a Track. A Group provides a synchronization point within a stream ({{?MoQTransport, Section 2.3}}).

Object:
: A discrete, addressable unit of media data within a Group. Objects contain media payloads (e.g., video frames or audio samples) ({{?MoQTransport, Section 2.1}}).

Sync-Track:
: A dedicated track used for synchronizing playback across multiple clients by transmitting PLAY, PAUSE, and SEEK commands.

Leader Client:
: The designated control authority within a synchronized playback session. The Leader Client issues playback control messages ({{?MoQTransport, Section 6}}) that all Consumers must follow.

Follower Client:
: A Consumer that follows playback instructions from the Leader Client to maintain synchronization.

Video-on-Demand (VoD) Session:
: A MoQ-based session where pre-recorded media is streamed with user-controlled playback features.

## Notational Conventions

This document uses the conventions detailed in ({{?RFC9000, Section 1.3}})
when describing the binary encoding.

As a quick reference, the following list provides a non-normative summary
of the parts of RFC9000 field syntax that are used in this specification.

x (..):

: Indicates that x can be any length including zero bits long.  Values
 in this format always end on a byte boundary.

x (i):

: Indicates that x holds an integer value using the variable-length
  encoding as described in ({{?RFC9000, Section 16}}).

# Synchronized Playback Control

Until the latest Media over QUIC Transport draft (draft-ietf-moq-transport-07), MOQT primarily supported a push-based content delivery architecture. The protocol does not explicitly define dedicated control messages for play, pause, or seek operations. Instead, these functionalities are achieved through subscription-based messaging.

## Playback Control Mechanisms

Playback is initiated by sending a SUBSCRIBE ({{?MoQTransport, Section 6.4}}) control message for the desired media track. The SUBSCRIBE message specifies the track namespace, track name, and a filter type (e.g., Latest Group or Absolute Start) to determine the starting point of the media delivery. Pausing playback is accomplished by sending a specific PAUSE message with its corresponding GroupId via Sync-Track. This stops the publisher from sending further objects for the specified track. To resume playback, the client must issue a PLAY message again via Sync-Track. In the case of staying inactive for a long-period of time, the client will be unsubscribed automatically via MOQT's UNSUBSCRIBE ({{?MoQTransport, Section 6.6}}). In order to start playback again, the client must send a SUBSCRIBE control message.

Seeking to a specific position within a track is supported using the SEEK message via Sync-Track, with the corresponding GroupId.  This allows the client to request media starting from a specific group,  continuing in a push-content fashion. However, this functionality requires that the publisher supports the ability to serve media from arbitrary positions within the track, which may not be universally available in all implementations. This is discussed in section 4 in detail.

This document introduces new Sync-Track messages, which can only be sent by the Leader client. When the Leader client sends these messages, a compatible publisher will respond by playing, pausing, or seeking as instructed. Upon receiving a PAUSE command, the publisher halts broadcasting at the specified group id, freezing playback until a PLAY command resumes it from the paused position. A SEEK  command directs the publisher to calculate the frame corresponding to the provided group id and restart broadcasting from that frame. This mechanism ensures all subscribers remain synchronized, allowing them to watch the stream simultaneously from the adjusted playback point.

### Decoupling Seek from Play/Pause

The SEEK operation functions independently of PLAY and PAUSE commands. A client can issue a SEEK command regardless of the current playback state. This means a client may seek to a different position while paused and later resume playback from the new position. Likewise, if a client is currently playing, a SEEK command will override the current playback position without requiring a PAUSE beforehand.

## Sync-Track Messages

Every single message on the Sync-Track is a MoQT object, which is transmitted using the MoQT streaming architecture. These messages encapsulate playback control commands (PLAY, PAUSE, SEEK) within MoQT objects, ensuring they are processed and relayed consistently across the session. The Sync-Track operates as a track within the MoQT session, where control messages are formatted as follows:

~~~
Sync-Track Message {
  Message Type (i),
  Message Length (i),
  Message Payload (..),
}
~~~
{: #shareplay-moq-message-format title="Overall Sync-Track Message Format"}

Each message type (PLAY, PAUSE, SEEK) is identified by its unique Message Type ID and follows a structured format to ensure integration with MoQT transport mechanisms. These messages are reliably delivered and processed in sequence, maintaining synchronization among all session participants. Since Sync-Track messages are standard MoQT objects, they leverage existing MoQT stream handling mechanisms.


|-------|-----------------------------------------------------|
| ID    | Messages                                            |
|------:|:----------------------------------------------------|
| 0x1   | PLAY ({{message-play}})                             |
|-------|-----------------------------------------------------|
| 0x2   | PAUSE ({{message-pause}})                           |
|-------|-----------------------------------------------------|
| 0x3   | SEEK ({{message-seek}})                             |
|-------|-----------------------------------------------------|
| 0x4   | STATUS ({{message-status}})                         |
|-------|-----------------------------------------------------|


### PLAY {#message-play}
~~~
PLAY Message {
  Type (i) = 0x1,
  Length (i),
  Timestamp (i),
}
~~~
{: #shareplay-moq-play-format title="Sync-Track PLAY Message"}

### PAUSE {#message-pause}
~~~
PAUSE Message {
  Type (i) = 0x2,
  Length (i),
  Timestamp (i),
  GroupId (i),
}
~~~
{: #shareplay-moq-pause-format title="Sync-Track PAUSE Message"}

### SEEK {#message-seek}

~~~
SEEK Message {
  Type (i) = 0x3,
  Length (i),
  Timestamp (i),
  GroupId (i),
}
~~~
{: #shareplay-moq-seek-format title="Sync-Track SEEK Message"}


### STATUS {#message-status}

Playing state is represented by 1, paused state is represented by 0 in PlaybackState parameter.

~~~
STATUS Message {
  Type (i) = 0x4,
  Length (i),
  Timestamp (i),
  GroupId (i),
  PlaybackState (i),
}
~~~
{: #shareplay-moq-status-format title="Sync-Track STATUS Message"}


## Timestamp for Playback Synchronization
Each playback control message (PLAY, PAUSE, SEEK) contains a timestamp field, which represents the intended media position when the action is triggered. This timestamp ensures that playback actions (especially SEEK) are synchronized across clients, preventing head-of-line blocking due to out-of-order commands. The publisher uses this timestamp to adjust the playback position accordingly.

## Grouping of Playback Control Messages
Play and Pause commands must be treated as dependent operations and thus belong to the same control subgroup. However, SEEK operates independently and belongs to a separate subgroup. This separation ensures that PLAY/PAUSE messages are processed sequentially within their stream, while SEEK commands are handled in a distinct stream to avoid blocking playback state transitions.

## Playback Status
To achieve better synchronization, a new status reporting message is introduced. This stream carries periodic updates from the publisher, indicating the current playback position, active state (playing/paused), and last acknowledged seek position. Clients can subscribe to this stream to stay updated on playback state changes.

## Playback Message Flow

The following message flow illustrates how playback operations (PLAY, SEEK, and PAUSE) are used through the relay servers, thus enabling clients to stay in a synchronized playback.

~~~ 
Leader Client             Relay Server(s)               Publisher   Follower Clients
     |                          |                          |               |
     |---------- PLAY --------->|                          |               |
     |                          |---------- PLAY --------->|               |
     |                          |<----- Media Stream ------|               |
     |<----- Media Stream ------|-------------- Media Stream ------------->|
     |<----- Media Stream ------|-------------- Media Stream ------------->|
     |<----- Media Stream ------|-------------- Media Stream ------------->|
     |---- SEEK {Group 10} ---->|                          |               |
     |                          |---- SEEK {Group 10} ---->|               |
     |<-- Media Obj from G10 ---|-----------  Media Obj from G10 --------->|
     |<----- Media Stream ------|-------------- Media Stream ------------->|
     |<----- Media Stream ------|-------------- Media Stream ------------->|
     |---- PAUSE {Group 18} --->|                          |               |
     |                          |---- PAUSE {Group 18} --->|               |
     |                          |<--- Stop Media At G18 ---|               |
     |                          |                          |               |
~~~
{: #shareplay-moq-message-flow-example title="Sync-Track Message Flow and Playback Example Scenario"}

### Message Semantics
These messages MUST be sent exclusively by the Leader Client (Client ID 0).
Relay servers MUST discard such messages if received from non-Leader clients.

# Leader-Follower Client Model

The Leader-Follower Client Model is a subscriber coordination mechanism designed to enforce synchronized media playback across multiple clients in a Media over QUIC (MoQ) session. This model ensures deterministic control over playback operations (e.g., play, pause, seek) to prevent conflicting actions and maintain synchronization.

## Role Assignment

For simplicity, no leader election algorithm is implemented in this model. The first subscriber to join a session is automatically designated as the Leader Client (Client ID 0). Subsequent subscribers are assigned incremental Follower Client IDs (e.g., 1, 2, ...) and inherit playback state from the Leader. This ensures straightforward session initiation without requiring complex coordination mechanisms.

## Playback Control

Exclusive Control:

: Only the Leader Client may issue playback Sync-Track messages (PLAY, PAUSE, SEEK). 
: The Sync-Track message requests that are coming from the Follower clients will be forwarded to the Leader Client for review. 

State Propagation:

: Control messages from the Leader Client are distributed to all Follower Clients via the relay servers, ensuring synchronized execution.

Leader Failure Handling:

: If the Leader Client disconnects or becomes unresponsive, the lowest-ID active Follower Client assumes the Leader role. The new Leader Client inherits Client ID 0, but subsequent Follower IDs do not change.

# Heartbeat-Based Session Management and Advanced Synchronization



While the Leader-Follower paradigm handles playback control, additional functionality is needed for:

1. Monitoring client synchronization and connectivity.

2. Handling clients that wish to get out of synchronization or rejoin later.

3. Making group-level decisions in the face of network issues.



This section defines a “heartbeat” mechanism and outlines how clients can join, leave, and optionally “fall behind” or “skip ahead”.



## Heartbeat Messages



Every client (Leader or Follower) periodically sends a **Heartbeat** message to the Relay (or publisher, depending on implementation). The Relay aggregates these messages to maintain an up-to-date view of each client’s playback status and connectivity. A Heartbeat message contains at least:

1. **Local Playback Time**: The client’s current playback position (e.g., Group ID or time offset (PTS) in the video).

2. **Current Global Time**: The client’s global timestamp, allowing the Relay (and/or Leader) to account for the delay in message transmission. 



### New Client Announcement



When a new client joins the session, the **first** Heartbeat message includes an additional “New-Client-Join” indicator. This allows the Relay (and indirectly the Leader) to:

- Recognize a new participant in the session.

- Potentially notify other participants (if needed).



### Heartbeat Timeouts and Disconnections



Each client is expected to send Heartbeat messages within a pre-decided interval. If a Heartbeat is not received for a duration exceeding that interval (plus some tolerance), the Relay assumes the client has become inactive or disconnected. This triggers either:

- **Temporary Removal**: The Relay marks the client as “inactive” but does not remove them from the session state immediately.

- **Full Removal**: If no further Heartbeat arrives within a longer grace period, the client is marked as “disconnected” and removed from the synchronized group.



## Out of Sync and Re-Syncing



A client may decide to pause or seek independently of the group or otherwise desynchronize its playback. For instance, a user might rewind to re-watch a scene without causing the entire group to follow. Such actions are handled as follows:



1. **Local-Only Rewind/Seek**:  

   The client can apply a local override of its playback buffer. It continues to send Heartbeats but indicates via a flag (e.g., “Out-of-Sync”) that it is not following the group.

2. **Resuming Sync**:  

   Once the client is ready to rejoin synchronized playback, it sends a Heartbeat with a special “Re-Sync Request” flag or re-subscribes to the Sync-Track. The Leader (or Relay) then supplies the necessary offset or group position so the client can jump to the current playback point and synchronize with others.



## Handling Network Issues and Group Policy



Clients experiencing connection issues (e.g., low bandwidth or high latency) can degrade the overall group experience if every other client must wait. To mitigate this:

1. **Adaptive Bitrate (ABR) at the Relay**:  

   The Relay can detect from Heartbeat messages that a client is consistently behind or losing packets. It may choose to deliver lower-resolution encodings to that specific client (if available) to help it catch up.

2. **Lobby Policy: Wait vs. Continue**:  

   An initial session-level policy can determine if the group is willing to wait for slow clients or not. If the policy is set to “continue,” the lagging client might partially skip forward (or remain unsynchronized) until network conditions improve. If the policy is set to “wait,” the group collectively applies additional buffering or pause states to allow the slow client to keep up.



## Fetch-Based Architecture



The current design builds on MoQ’s push-based content delivery. An upcoming revision (based on [draft-ietf-moq-transport-07] “fetch” functionality) will introduce a more scalable, fetch-oriented mechanism for client synchronization. This approach will further refine:

- The structure of Heartbeat/Status messages.

- How group membership is signaled and updated.

- Interaction between “push” and “fetch” paradigms for adaptive VoD synchronization.

- More effective use of relays for more scalable systems.


# Security Considerations {#security}

TODO: Expand this section.

# IANA Considerations {#iana}

TODO: Expand this section.

--- back

# Example Publisher Model

This section defines an example architecture for adapting Media over QUIC (MoQ) to Video-on-Demand (VoD) use cases. The model enables efficient retrieval and streaming of archived fragmented MP4 (fMP4) content through a specialized publisher (`moq-pub`), while maintaining compatibility with live-edge workflows.

## Media Preparation  
Encoding:
: Media is encoded in H.264 (AVC) format.

Containerization:
: Media is packaged in fragmented MP4 (fMP4) format, where content is divided into discrete "moof" (Movie Fragment) and "mdat" (Media Data) atom pairs.

Initialization Segments:
: The `ftyp` (File Type) and `moov` (Movie Metadata) atoms are prepended to the media file. These provide global metadata (e.g., codec information, track layouts) and are transmitted once per session.

## Publisher (`moq-pub`) Responsibilities  
Storage Integration:
: Stream archived fMP4 content directly from publishers local storage, eliminating the need for real-time transcoding.

Efficient Retrieval of Media:
: Support arbitrary access to media fragments (moof+mdat pairs) for low-latency seek operations.

Media Codec: `moq-pub` supports H.264 encapsulated in fMP4.

## Streaming Mechanism  

Atomic Units:
: Media is streamed as sequential "moof+mdat" atom pairs, each representing a MoQ *object*. The `moof` atoms precede its corresponding `mdat` atom to ensure decodability.
 
Group Mapping:
: Each moof+mdat pair corresponds to a frame in the H.264 stream. A group of frames form a Group of Picture (GOP). A GOP begins with an Intra-coded frame (I-frame), followed by Predictive frames (P/B-frames).
The `GroupId` parameter in MoQ aligns with the GOP index in H.264. This ensures that seeking to a specific `GroupId` starts playback from the I-frame at the beginning of the requested GOP, enabling correct decoder initialization.
Publisher maintains an internal index mapping `GroupId` to the byte offset of the corresponding GOP’s moof+mdat pair.

### Session Initialization  
At the start of a streaming session:
: 1. The publisher transmits the `ftyp` and `moov` atoms as an initialization segment.
2. Subsequent objects (moof+mdat pairs) are transmitted incrementally, adhering to the subscriber’s playback state. 

## Dynamic Playback Control  
The publisher asynchronously monitors a dedicated `Sync-Track` for control messages (e.g., `PLAY`, `PAUSE`, `SEEK`) issued by the *Leader Client*.

### Command Processing
Seek Operations: On receiving a `SEEK` command:
: 1. The publisher identifies the start of the nearest GOP boundary for the requested `GroupId`, and resumes publishing from the corresponding frame. 
  2. Streaming resumes from the start of the GOP containing the target I-frame, ensuring all dependent P/B-frames are included for seamless decoding.
  3. Subscribers receive the full GOP sequence, eliminating decoder errors caused by missing reference frames.
Pause/Resume:
: A `PAUSE` command halts the transmission according to the GroupId specified in the PAUSE message, where GroupId corresponds to current group of the playback. A subsequent `PLAY` command resumes streaming from the next sequential object. 
