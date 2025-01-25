---
title: "Synchronized Social Video-on-Demand (VoD) Viewing [System]? [with/for]? Media over QUIC" #?
abbrev: shareplay-moq #?
docname: draft-{names}-shareplay-moq-latest #?
date: {DATE}
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
  #? Name ordering?
  -
    ins: A. Begen
    name: Ali Begen
    organization: Networked Media
    email: ali.begen@networked.media

  -
    ins: Z. Gurel
    name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr

  -
    ins: K. Bekmez
    name: Kerem Bekmez
    organization: Ozyegin University
    email: kerem.bekmez@ozu.edu.tr

  -
    ins: B. Yumakogullari
    name: Başar Yumakogulları
    organization: Ozyegin University
    email: basar.yumakogullari@ozu.edu.tr

  -
    ins: A. Pehlivanoglu
    name: Ahmet Pehlivanoglu
    organization: Ozyegin University
    email: ahmet.pehlivanoglu@ozu.edu.tr
  
--- abstract

This document presents an Synchronized Social Video-on-Demand (VoD) Viewing [System]? [with/for]? Media over QUIC (MoQ) that extends Media Over QUIC Transport (MOQT) protocol to enable synchronized Video-on-Demand (VoD) functionality. The [system]? adapts MoQ’s push-content architecture to include interactive features such as pause, resume, seek, and replay, addressing limitations in current implementations.

--- middle

# Introduction

Media Over QUIC (MoQ) is an emerging protocol designed for efficient media streaming over the QUIC transport protocol. While MoQ has shown promise for live-edge streaming, its current design lacks full support for interactive VoD functionalities, such as pause, resume, seek, and replay. This document addresses these gaps by adapting MoQ for on-demand, synchronized media consumption.

This document outlines the architectural designs, control mechanisms, and synchronization logic implemented to achieve these objectives. This document's innovations include new MOQT tracks for synchronisation control, a media publisher model for serving on-demand video, and a master-slave client infrastructure to demonstrate the potential of MoQ for enhancing synchronized VoD consumption.

# Playback Control via Sync-track
Until of the latest Media over QUIC Transport draft (draft-ietf-moq-transport-07), MOQT primarily supports a push-based content delivery architecture. The protocol does not explicitly define dedicated control messages for play, pause, or seek operations. Instead, these functionalities are achieved through the following mechanisms:


Playback is initiated by sending a SUBSCRIBE control message for the desired media track. The SUBSCRIBE message specifies the track namespace, track name, and a filter type (e.g., Latest Group or Absolute Start) to determine the starting point of the media delivery. Pausing playback is accomplished by sending an UNSUBSCRIBE control message for the active subscription. This stops the publisher from sending further objects for the specified track. To resume playback, the client must issue a new SUBSCRIBE message.
Seeking to a specific position within a track is supported using the Absolute Start filter type in the SUBSCRIBE message. This allows the client to request media starting from a specific group and object ID. However, this functionality requires that the publisher supports the ability to serve media from arbitrary positions within the track, which may not be universally available in all implementations.

This document introduces new Sync-track messages, which can only be sent by the master client. When the master client sends these messages, a compatible publisher will respond by playing, pausing, or seeking as instructed. Upon receiving a "pause" command, the publisher halts broadcasting at the specified group number, freezing playback until a "play" command resumes it from the paused position. A "seek" command directs the publisher to calculate the frame corresponding to the provided Group Number and restart broadcasting from that frame. This mechanism ensures all subscribers (or clients) remain synchronized, allowing them to watch the stream simultaneously from the adjusted playback point.

## Playback Control Messages:

Play:

: PLAY Message { }

Pause:

: PAUSE Message {}

Seek:

: SEEK Message {
  GroupNumber } 

### Message Semantics
These messages MUST be sent exclusively by the Master Client (Client ID 0).
Relay servers MUST discard such messages if received from non-Master clients.


# Master-Slave Client Model

The Master-Slave Client Model is a subscriber coordination mechanism designed to enforce synchronized media playback across multiple clients in a Media over QUIC (MoQ) session. This model ensures deterministic control over playback operations (e.g., play, pause, seek) to prevent conflicting actions and maintain synchronization.

## Role Assignment:

: The first subscriber to join a session is designated as the Master Client (Client ID 0). Subsequent subscribers are assigned incremental Slave Client IDs (e.g., 1, 2, ...) and inherit playback state from the Master.

## Playback Control:

Exclusive Control:

: Only the Master Client may issue playback control messages (PLAY, PAUSE, SEEK).

State Propagation:

: Control messages from the Master Client are distributed to all Slave Clients via the relay server, ensuring synchronized execution.

Master Failure Handling:

: If the Master Client disconnects or becomes unresponsive, the lowest-ID active Slave Client assumes the Master role. The new Master Client inherits Client ID 0, but subsequent Slave IDs do not change.


