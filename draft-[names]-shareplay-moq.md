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
