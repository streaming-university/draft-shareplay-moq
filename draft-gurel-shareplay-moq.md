---
title: "Synchronized Social Video-on-Demand (VoD) Viewing with Media over QUIC"
abbrev: shareplay-moq 
docname: draft-gurel-shareplay-moq-latest
date:
  year: 2025
  month: January
  day: 31
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

This document outlines the architectural designs, control mechanisms, and synchronization logic implemented to achieve these objectives. This document's innovations include new MOQT tracks for synchronisation control, a media publisher model for serving on-demand video, and a master-slave client infrastructure to demonstrate the potential of MoQ for enhancing synchronized VoD consumption.

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

Master Client:  
: The designated control authority within a synchronized playback session. The Master Client issues playback control messages ({{?MoQTransport, Section 6}}) that all Consumers must follow.

Slave Client:  
: A Consumer that follows playback instructions from the Master Client to maintain synchronization.

Video-on-Demand (VoD) Session:  
: A MoQ-based session where pre-recorded media is streamed with user-controlled playback features.


# Synchronized Playback Control

Until of the latest Media over QUIC Transport draft (draft-ietf-moq-transport-07), MOQT primarily supported a push-based content delivery architecture. The protocol does not explicitly define dedicated control messages for play, pause, or seek operations. Instead, these functionalities are achieved through subscription-based messaging.


## Playback Control Mechanisms

Playback is initiated by sending a SUBSCRIBE ({{?MoQTransport, Section 6.4}}) control message for the desired media track. The SUBSCRIBE message specifies the track namespace, track name, and a filter type (e.g., Latest Group or Absolute Start) to determine the starting point of the media delivery. Pausing playback is accomplished by sending a specific pause message with its corresponding group_id via Sync-Track. This stops the publisher from sending further objects for the specified track. To resume playback, the client must issue a Play message again via Sync-Track. In the case of staying inactive for a long-period of time, the client will be unsubscribed automatically via MOQT's UNSUBSCRIBE({{?MoQTransport, Section 6.6}}) via control track. In order to start playback again, the client must send a SUBSCRIBE control message again.

Seeking to a specific position within a track is supported using the Seek message via Sync-Track, with the corresponding group_id.  This allows the client to request media starting from a specific group, and continue receiving in a push-content fashion. However, this functionality requires that the publisher supports the ability to serve media from arbitrary positions within the track, which may not be universally available in all implementations. This is discussed in section 4 in detail.

This document introduces new Sync-track messages, which can only be sent by the master client. When the master client sends these messages, a compatible publisher will respond by playing, pausing, or seeking as instructed. Upon receiving a "pause" command, the publisher halts broadcasting at the specified group number, freezing playback until a "play" command resumes it from the paused position. A "seek" command directs the publisher to calculate the frame corresponding to the provided Group Number and restart broadcasting from that frame. This mechanism ensures all subscribers (or clients) remain synchronized, allowing them to watch the stream simultaneously from the adjusted playback point.

## Sync-Track Messages
Play:

: PLAY Message {}

Pause:

: PAUSE Message {
  GroupNumber }

Seek:

: SEEK Message {
  GroupNumber } 

### Message Semantics
These messages MUST be sent exclusively by the Master Client (Client ID 0).
Relay servers MUST discard such messages if received from non-Master clients.


# Master-Slave Client Model

The Master-Slave Client Model is a subscriber coordination mechanism designed to enforce synchronized media playback across multiple clients in a Media over QUIC (MoQ) session. This model ensures deterministic control over playback operations (e.g., play, pause, seek) to prevent conflicting actions and maintain synchronization.

## Role Assignment

: The first subscriber to join a session is designated as the Master Client (Client ID 0). Subsequent subscribers are assigned incremental Slave Client IDs (e.g., 1, 2, ...) and inherit playback state from the Master.

## Playback Control

Exclusive Control:

: Only the Master Client may issue playback sync-track messages (PLAY, PAUSE, SEEK). 
: The sync-track message requests that are coming from the slave clients will be forwarded to the Master Client for review. 

State Propagation:

: Control messages from the Master Client are distributed to all Slave Clients via the relay server, ensuring synchronized execution.

Master Failure Handling:

: If the Master Client disconnects or becomes unresponsive, the lowest-ID active Slave Client assumes the Master role. The new Master Client inherits Client ID 0, but subsequent Slave IDs do not change.


# Video-on-Demand Architecture over MoQ

This section defines an architecture for adapting Media over QUIC (MoQ) to Video-on-Demand (VoD) use cases. The model enables efficient retrieval and streaming of archived fragmented MP4 (fMP4) content through a specialized publisher (`moq-pub`), while maintaining compatibility with live-edge workflows.  

## Media Preparation Requirements  
Encoding:
: Media MUST be encoded in H.264 (AVC) format.  
Containerization:
: Media MUST be packaged in fragmented MP4 (fMP4) format, where content is divided into discrete "moof" (Movie Fragment) and "mdat" (Media Data) atom pairs.  
Initialization Segments:
: The `ftyp` (File Type) and `moov` (Movie Metadata) atoms MUST be prepended to the media file. These provide global metadata (e.g., codec information, track layouts) and are transmitted once per session.  

## Publisher (`moq-pub`) Responsibilities  
Storage Integration:
: Stream archived fMP4 content directly from publishers local storage, eliminating the need for real-time transcoding.  

Efficient Retrieval of Media:
: Support arbitrary access to media fragments (moof+mdat pairs) for low-latency seek operations.  

Media Codec: `moq-pub` MUST support H.264 encapsulated in fMP4.  

## Streaming Mechanism  

Atomic Units:
: Media is streamed as sequential "moof+mdat" atom pairs, each representing a MoQ *object*. The `moof` atom MUST precede its corresponding `mdat` atom to ensure decodability.  
 
Group Mapping:  
: Each moof+mdat pair corresponds to a frame in the H.264 stream. A group of frames form a Group of Picture (GOP). A GOP begins with an Intra-coded frame (I-frame), followed by Predictive frames (P/B-frames).  
The `GroupNumber` parameter in MoQ aligns with the GOP index in H.264. This ensures that seeking to a specific `GroupNumber` starts playback from the I-frame at the beginning of the requested GOP, enabling correct decoder initialization.  
Publishers MUST maintain an internal index mapping `GroupNumber` to the byte offset of the corresponding GOP’s moof+mdat pair.  

### Session Initialization  
At the start of a streaming session:  
: 1. The publisher transmits the `ftyp` and `moov` atoms as an initialization segment.  
2. Subsequent objects (moof+mdat pairs) are transmitted incrementally, adhering to the subscriber’s playback state. 

## Dynamic Playback Control  
The publisher asynchronously monitors a dedicated `sync-track` for control messages (e.g., `PLAY`, `PAUSE`, `SEEK`) issued by the *Master Client*.  

### Command Processing  
Seek Operations: On receiving a `SEEK { GroupNumber }` command:  
:   1. The publisher identifies the start of the nearest GOP boundary for the requested `GroupNumber`, and resumes publishing from the corresponding frame. 
  2. Streaming resumes from the start of the GOP containing the target I-frame, ensuring all dependent P/B-frames are included for seamless decoding.  
  3. Subscribers receive the full GOP sequence, eliminating decoder errors caused by missing reference frames.  
Pause/Resume:
: A `PAUSE` command halts the transmission according to the group_id specified in the pause message, where group_id corresponds to current group of the playback. A subsequent `PLAY` command resumes streaming from the next sequential object. 


# Security Considerations {#security}

TODO: Expand this section.

# IANA Considerations {#iana}

TODO: Expand this section.
