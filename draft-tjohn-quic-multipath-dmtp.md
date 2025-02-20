---
title: "Deadline Aware Streams in MP-QUIC"
category: info # TODO: Figure out correct category

docname: draft-tjohn-quic-multipath-dmtp-latest
submissiontype: IETF # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "QUIC"
keyword:
 - deadline-aware
 - multipath
 - scheduling
 - path aware networks
venue:
  group: "QUIC"
  type: ""
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "netsys-lab/mpquic-dmtp-draft"
  latest: "https://netsys-lab.github.io/mpquic-dmtp-draft/draft-tjohn-quic-multipath-dmtp.html"

author:
 -
    fullname: "Tony John"
    organization: "Otto-von-Guericke University Magdeburg"
    email: "tony.john@ovgu.de"
 -
    fullname: "Till-Frederik Riechard"
    organization: "Otto-von-Guericke University Magdeburg"
    email: "riechard@ovgu.de"

normative:
   QUIC: rfc9000
   QUIC-TLS: rfc9001
   MP-QUIC: I-D.draft-ietf-quic-multipath
   QUIC-AFEC: I-D.draft-dmoskvitin-quic-adaptive-fec
   RFC3339:

informative:
   DMTP: DOI.10.23919/IFIPNetworking57963.2023.10186417
   SCION: DOI.10.1007/978-3-031-05288-0
   QUIC-DTP: I-D.draft-shi-quic-dtp


--- abstract

This document proposes the Deadline-aware Multipath Transport Protocol (DMTP) concept as an extension to the Multipath extension of QUIC (MP-QUIC). This extension aims to support data streams with strict latency requirements by enabling the signaling of per-stream deadlines and by combining multipath scheduling, congestion control adaptations, and optional forward error correction (FEC). Furthermore, by abstracting the different end-to-end paths available in a Path Aware Network (PAN) such as SCION into MP-QUIC's path identifiers, we allow an application to select its preferred paths while maintaining interoperability with standard MP-QUIC endpoints. This new mechanism provides a standardized way for endpoints to exchange and schedule deadline-aware streams across multiple network paths.

--- middle

# Introduction

The Multipath Extension of QUIC {{MP-QUIC}} enhances performance by simultaneously utilizing multiple paths between endpoints. However, it currently lacks direct mechanisms to meet per-stream deadline requirements, which are increasingly important for real-time applications such as teleoperation, live video streaming, and interactive gaming. These applications demand low and bounded latency, and can often tolerate partial or no retransmission of late data.

Previous work on deadline-aware protocols for QUIC includes a single-path approach, namely Deadline-aware Transport Protocol {{QUIC-DTP}}, that introduced per-stream deadlines but did not exploit multipath capabilities. Meanwhile, our conference paper proposing DMTP {{DMTP}} highlighted how taking advantage of multiple paths, combined with forward error correction (FEC) and intelligent retransmissions, can significantly increase the fraction of packets meeting their deadlines, especially over lossy or high-latency links.

By integrating deadline-aware concepts into MP-QUIC, we seek to enable:

1. Multipath streams with Deadlines: Scheduling and transmitting data across multiple paths, with per-stream deadlines that inform scheduling decisions.
2. Option for Path-Aware Networking: Abstracting path selection from Path-Aware Networks (e.g., {{SCION}}) by mapping each potential path to an MP-QUIC path identifier.
3. Deadline-Based Retransmission / FEC: Combining optional adaptive FEC (such as {{QUIC-AFEC}}) and “smart” retransmissions only when there is time left to meet the deadline.

This draft specifies a minimal set of protocol extensions for MP-QUIC to exchange deadline information at the transport layer, so that endpoints can coordinate scheduling for multipath transmissions with strict time constraints. It allows different implementations to interoperate and exchange deadline-aware streams over MP-QUIC.

## Motivation and Applications

Real-time applications often produce data blocks (e.g., video frames or control messages) that are only valuable if delivered before their deadline. Example use cases include:

- Teleoperation and Remote Control: Robotic control or telepresence systems require deterministic and low latency feedback. Missing control signals,  sensor data or video frame deadlines can lead to system instability or degraded user experience.
- Live Streaming and Interactive Media: Latency-sensitive video or audio streams (e.g., for live concerts, online VR gaming, cloud rendering) benefit from leveraging multiple paths to sustain low-latency delivery even under varying network conditions.
- Online Gaming: Multiplayer networked games exchange frequent, time-critical state updates. Late updates are effectively wasted, so a mechanism to drop or deprioritize old data can save bandwidth and improve real-time responsiveness.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Within this document:
- "Deadline-aware streams" refers to streams in which an application indicates a time by which data must be delivered, beyond which data is no longer useful.
- "Path" aligns with the MP-QUIC concept: each path is identified by a unique Path ID, referencing a specific combination of source and destination IP:port tuples or multiple distinct end-to-end routes in a path-aware network architecture.

# Design Overview

## Integrating Deadline-Aware Streams into MP-QUIC

The design goal is to extend MP-QUIC with minimal changes. The extensions enable endpoints to signal per-stream deadlines. Implementations that support deadline-aware streams MUST implement:

1. Path Selection: Select paths for transmitting frames, retransmissions and acknowledgements based on metrics relevant to meeting deadlines, including:
   - Path latency measurements
   - Available bandwidth estimation
   - Observed packet loss rates

2. Packet Scheduling: Implement scheduling algorithms that:
   - Prioritize packets from streams with earlier deadlines
   - Account for path characteristics when making scheduling decisions
   - Consider current congestion state of available paths

3. Retransmission Control: Implement retransmission policies that:
   - Evaluate whether retransmitted packets can meet remaining deadlines
   - Skip retransmissions when deadlines cannot be met
   - Consider path conditions when selecting retransmission paths

4. Optional Forward Error Correction: MAY implement FEC mechanisms that:
   - Reduce retransmission overhead for deadline-sensitive streams
   - Adapt FEC overhead based on path conditions
   - Apply FEC selectively based on stream requirements (e.g., stream priority)

5. Deadline Monitoring: Track deadline status and:
   - Detect when deadlines cannot be met
   - Signal deadline misses to the application layer
   - Allow applications to specify handling of missed deadlines

### Minimal Changes to MP-QUIC

Our extensions reuse MP-QUIC’s multipath concepts (paths, path IDs, path validation, etc.) with only the following additions:

- Deadline Transport Parameter: `enable_deadline_aware_streams`.
- Custom Deadline Frame: DEADLINE_CONTROL frame to specify deadlines for streams (see {{deadline-control-frame}}).
- Optional DMTP Acknowledgement: DMTP_ACK frames enable more precise per-path delay estimation and real-time path
performance feedback (see {{dmtp-ack-frame}}).
- Optional AFEC Transport Parameter & Frames: If implementations wish to use Adaptive FEC as in {{QUIC-AFEC}}, it introduces an additional transport parameter for FEC support and two new frames for carrying source and repair symbol metadata across multiple paths.

By doing so, we preserve MP-QUIC’s wire format except for newly introduced frames and transport parameters, ensuring interoperability with multipath-capable endpoints that do not recognize these extensions (they will treat unknown frames as errors if negotiated incorrectly, or ignore them if permitted).

### Path Selection in Path-Aware Networks (PAN)

When running over a Path-Aware Network such as {{SCION}}, endpoints may discover multiple disjoint or partially disjoint paths. In SCION, a host can receive multiple end-to-end path options from the network. We leverage MP-QUIC’s concept of multiple path IDs to represent each distinct path from the SCION set. This means that SCION’s path selection is performed outside of MP-QUIC, but MP-QUIC uses the resulting IP:port tuples (plus path ID) to treat each route as a separate path. The advantage is that no additional MP-QUIC modifications are needed to “support SCION” - we simply map each SCION path to a separate MP-QUIC path ID and rely on our new deadline-aware logic to decide scheduling and retransmissions.

## Deadlines

### Signalling Deadlines

To signal deadlines, endpoints use the DEADLINE_CONTROL frame (see {{deadline-control-frame}}). This frame associates a specific deadline with a stream, indicating the relative time by which the data should be delivered.

### Deadline Semantics

- Deadline Representation: Deadlines are represented as a relative time in milliseconds from the time the frame is sent.
- Stream Association: A deadline applies to a specific stream identified by its Stream ID
- Transport Behavior: Upon receiving a DEADLINE_CONTROL frame, the transport layer SHOULD attempt to schedule and retransmit packets carrying data for the specified stream to meet the indicated deadline.
- Retransmissions and Scheduling: Endpoints MAY implement custom schedulers and congestion controllers optimized for deadline-aware traffic, such as those based on DMTP concepts.

### Handling Missed Deadline

If the transport layer determines that the deadline cannot be met, it MAY choose to:

- Discard the data associated with the deadline-aware stream.
- Inform the application of the missed deadline.
- Continue delivering the data if it is still deemed useful.

The specific behavior is implementation-specific and MAY be configurable by the application.

## Adaptive Forward Error Correction (FEC)

When deadlines are tight and packet losses frequent, relying solely on retransmissions may cause data to miss its deadline. To mitigate this risk, this extension optionally uses Adaptive FEC (AFEC) as proposed in {{QUIC-AFEC}}. AFEC can reduce the need for retransmissions, particularly in networks with random or bursty loss characteristics.

When using AFEC with {{MP-QUIC}}, the Tag Type of the FEC_Tag MUST be set to 1 to indicate “Long Flow Usage.” In turn, both source symbol packets and repair symbol packets MUST carry the FEC_Tag frame so that repair packets can be correctly matched to their corresponding source packets across different paths.

If multiple paths are available, FEC repair packets SHOULD be sent over a path different from the one carrying the source data. This de-correlates losses and increases the likelihood that repair symbols arrive even if other paths experience congestion or packet loss. The coding rate (i.e., the ratio of repair symbols to source symbols) MAY be configured on a per-stream basis, depending on the stream’s tolerance for overhead versus its deadline sensitivity.

## Smart Retransmissions

Smart retransmissions in a deadline-aware context mean that lost frames are only retransmitted if there is still enough time left to meet the deadline via one or more paths. The sender computes whether the frames can arrive on time, factoring in the path’s estimated one-way delay or RTT. If not, the sender discards the frames rather than wasting congestion window or scheduling capacity.

## Path Metrics

To schedule traffic effectively, the sender should gather:

- One-Way Delays or RTT: For selecting the path(s) that can deliver data before the deadline.
- Loss Rate: For deciding whether to apply adaptive FEC or more aggressive retransmissions.
- Available Bandwidth: So that sending on path(s) with insufficient capacity does not cause additional delay.

### Per Path Delay

A crucial metric for DMTP is the one-way or round-trip delay of each path. This is used to decide whether a new or retransmitted packet can arrive before its deadline. In a path-aware network like {{SCION}}, the one-way delay might be advertised or inferred from routing information. Otherwise, endpoints measure RTT or one-way delay themselves.

For accurate one-way delay measurements, endpoints MAY use synchronized clocks; if full clock sync is not feasible, a fallback to round-trip time measurements is still acceptable. The DMTP_ACK frame (see {{dmtp-ack-frame}}) is introduced primarily for improved delay tracking.

### Gathering Path Metrics

1. Path-Aware Networks (e.g. SCION) might provide direct metrics, such as path latency or bandwidth as part of path metadata.
2. Active Probing: If the underlying network does not provide metrics, the endpoint MAY send periodic PING frames or small test packets along each active path.
3. Path Measurement Frames: This draft introduces an optional DMTP_ACK frame ({{dmtp-ack-frame}}) for deeper path measurements, including timestamps of packet receipts to estimate per path one-way delay.
4. Congestion windows, RTT estimates, and packet loss detection from MP-QUIC’s standard loss recovery can inform scheduling.

# Extension of Multipath QUIC

This extension builds upon {{MP-QUIC}}. Below we list the protocol additions and modifications. Unless otherwise noted, all rules of MP-QUIC remain.

## Handshake Negotiation and Transport Parameter {#transport-parameter}

This extension defines a new transport parameter, used to negotiate the use of deadline-aware streams during the connection handshake, as specified in {{QUIC}}. The new transport parameter is defined as follows:

- enable_deadline_aware_streams (value TBD): A zero-length value that, if present, indicates that the endpoint supports deadline-aware streams.

Endpoints negotiate the use of deadline-aware streams by including the enable_deadline_aware_streams transport parameter in the handshake. Both endpoints MUST include this transport parameter to enable the use of deadline-aware streams. If an endpoint receives a DEADLINE_CONTROL frame without having negotiated support, it MUST treat this as a connection error of type PROTOCOL_VIOLATION

## DEADLINE_CONTROL Frame {#deadline-control-frame}

The DEADLINE_CONTROL frame (type=TBD) is used to signal deadline-awareness for specific streams and to indicate their associated deadlines.

~~~
  DEADLINE_CONTROL Frame {
    Type (i) = TBD,
    Stream ID (i),
    Deadline (i),
  }
~~~

The DEADLINE_CONTROL frame contains the following fields:

Stream ID:
: A variable-length integer indicating the Stream ID to which the deadline applies.

Deadline:
: A variable-length integer representing the relative deadline in milliseconds from the time the frame is sent.
An endpoint sends a DEADLINE_CONTROL frame to indicate that data on the specified stream should be delivered by the given deadline. Upon receiving this frame, the peer MUST attempt to schedule and deliver the data on the specified stream within the indicated deadline.

Usage Constraints:

- Endpoints MUST NOT send the DEADLINE_CONTROL frame unless both endpoints have negotiated support via the enable_deadline_aware_streams transport parameter.
- If an endpoint receives a DEADLINE_CONTROL frame without having negotiated support, it MUST treat it as a connection error of type PROTOCOL_VIOLATION.
- The DEADLINE_CONTROL frame MUST only be sent in 1-RTT packets.

## DMTP_ACK Frame {#dmtp-ack-frame}

The DMTP_ACK frame (type=TBD) is used to acknowledge the reception of a packet and feedback the reception time to the sender. If the received frame contains a PING (type=0x01) frame, the DMTP_ACK frame MUST be sent back on the same path that it was received at. The DMTP_ACK Frame contains the same information as a {{QUIC}} ACK frame, but adds a timestamp to it, in order to communicate if a packet has met its deadline or not.

When using deadline-awareness the receiver SHOULD acknowledge each packet separately.

~~~
  DMTP_ACK Frame {
   Type (i) = TBD,
   Largest Acknowledged (i),
   ACK Delay (i),
   ACK Range Count (i),
   First ACK Range (i),
   ACK Range (..) ...,
   [ECN Counts (..)],
   Timestamp (i),
  }
~~~

The DMTP_ACK frame adds the Timestamp field to the {{QUIC}} ACK frame. It MUST be formatted according to {{RFC3339}} with a resolution down to the nanosecond, i.e. with 9 digits after the decimal point. If an endpoint uses a clock with a lower resolution, the remaining digits SHOULD be padded with zeros.

# API

Though this draft primarily focuses on wire-level protocol changes, an implementation that exposes a user-level API might provide:

- SetStreamDeadline(stream_id, deadline_ms):
  Informs the transport that data on `stream_id` must arrive before `deadline_ms`.
- SetStreamPriority(stream_id, priority):
  Assigns or updates the priority for a stream. Lower numerical priority can indicate higher reliability requirement.
- OnMissedDeadline(stream_id):
  (Optional) callback that the transport can invoke if data is considered impossible to deliver on time. The application can choose to send new data, discard, or do nothing.

These calls let an application specify deadlines and priorities dynamically.

# Security Considerations

This extension retains all the security features and considerations of {{QUIC}}, {{QUIC-TLS}} and {{MP-QUIC}}.
Nevertheless, it introduces additional considerations:

- Deadline Signaling: Knowledge of deadlines or priorities may be sensitive if it reveals application timing patterns or critical data intervals. Implementations SHOULD carefully handle metadata (e.g., by encrypting frames in 1-RTT packets).
- Resource Exhaustion and Flooding: The ability to manage multiple concurrent paths and to schedule or drop data based on deadlines must not weaken QUIC’s anti-amplification measures. Endpoints MUST still follow QUIC path validation procedures, ensuring that an attacker cannot exploit deadline-aware frames to amplify traffic.
- When employing DMTP_ACK frames for one-way delay measurement with clock synchronization, the clock synchronization must also be secured. Otherwise, an attacker injecting false timestamps could mislead scheduling. Endpoints that rely heavily on these measurements should be aware of that risk and possibly cross-check with measured RTT or other heuristics.

# IANA Considerations

This document defines a new transport parameter for the negotiation of enable multiple paths for QUIC, and two new frame types. The draft defines provisional values for experiments, but we expect IANA to allocate short values if the draft is approved.

The following entry in {{transport-parameters}} should be added to the "QUIC Transport Parameters" registry under the "QUIC Protocol" heading.

Value                                         | Parameter Name.   | Specification
----------------------------------------------|-------------------|-----------------
TBD                                           | enable_deadline_aware_streams  | {{transport-parameter}}
{: #transport-parameters title="Addition to QUIC Transport Parameters Entries"}

The following frame type defined in {{frame-types}} should be added to
the "QUIC Frame Types" registry under the "QUIC Protocol" heading.

Value                                              | Frame Name          | Specification
---------------------------------------------------|---------------------|-----------------
TBD (1)                                            | DEADLINE_CONTROL    | {{deadline-control-frame}}
TBD (2)                                            | DMTP_ACK            | {{dmtp-ack-frame}}
{: #frame-types title="Addition to QUIC Frame Types Entries"}

--- back

# Acknowledgments
{:numbered="false"}

The authors thank the QUIC working group and the designers of both {{MP-QUIC}} and {{QUIC-DTP}} for paving the way for deadline-aware features in QUIC. The concept of scheduling data with deadlines over multiple paths builds on numerous discussions around partial reliability, adaptive FEC, and optimal path selection.
