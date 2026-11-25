---
title: "QLOG extensions for Multiple Paths in QUIC"
abbrev: "qlog-multipath"
category: info

docname: draft-huitema-quic-qlog-multipath-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "QUIC"
keyword:
 - QUIC
 - multipath
 - qlog
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  github: "huitema/qlog-multipath-extension"

author:
 -
    fullname: Christian Huitema
    organization: Private Octopus Inc.
    email: huitema@huitema.net
 -
    fullname: Yanmei Liu
    organization: Alibaba Inc.
    email: miaoji.lym@alibaba-inc.com
 -
    fullname: Altanai Bisht
    organization: Cisco Systems
    email: albisht@cisco.com
 -
    fullname: Noriyuki Kurosu
    organization: Individual
    email: 3p0rta26@gmail.com


normative:

   QUIC-TRANSPORT:
       RFC9000
   QUIC-MULTIPATH:
       RFC10069
   QLOG-MAIN:
       I-D.ietf-quic-qlog-main-schema
   QLOG-QUIC:
       I-D.ietf-quic-qlog-quic-events


informative:
  ANRW-2020:
    title: "Debugging QUIC and HTTP/3 with qlog and qvis"
    date: "2020"
    seriesinfo: "ANRW '20: Proceedings of the 2020 Applied Networking Research Workshop"
    author:
    -
      ins: R. Marx
    -
      ins: M. Piraux
    -
      ins: P. Quax
    -
      ins: W. Lamotte

...

--- abstract

The qlog specifications define structured logs for QUIC connections.
This draft extends the qlog specification for QUIC events. It adds
an optional Path ID element to the description of events, defines new events to log the
creation and management of multiple paths, and specifies the logging format
of new frames introduced in RFC 10069.

--- middle

# Introduction {#intro}

The QLOG drafts define the syntax of QLOG {{QLOG-MAIN}}
and the events and frames for logging QUIC events {{QLOG-QUIC}}
found in QUIC connections per {{QLOG-QUIC}}. The extensions to {{QUIC-TRANSPORT}} for
managing multiple paths for a QUIC connections are defined in {{QUIC-MULTIPATH}},
which introduces new concepts such as path, new events such as creating or
abandoning paths, and new frames for managing traffic when using multiple
paths.

This draft defines an extension to the QUIC events specification in
{{QLOG-QUIC}} to enable logging of QUIC connections that manage
multiple paths according to {{QUIC-MULTIPATH}}.

Debugging tools like qvis have proven very useful when debugging QUIC implementations
(see {{ANRW-2020}}). This document provides a standard way to add specific information
in QLOG traces so updated tools can properly represent QUIC connections that use
multiple paths according to {{QUIC-MULTIPATH}}. For example, a key function of these
tools is to correlate packet departures and the arrival of the corresponding
acknowledgement. For plain QUIC connections, this can be done by examining
packet numbers. With multipath, there is a packet number space per path, and
correlation for 1RTT packets requires examining both packet numbers
and the ID of the path on which the
packet was sent. The required logging information is defined in this document.

This document defines how a path ID can be added to event headers in {{path-id-def}},
adds events describing the management of paths in {{mp-events}},
explains the effects of adding a path ID to events defined in
{{QLOG-QUIC}} in {{add-path-id}},
specifies the logging of multipath specific frames in {{multipath-frames}},
and that of the multipath transport parameter in {{mp-parameters}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Event Schema Definition {#event-schema}

This document extends the `quic` namespace of
{{QLOG-QUIC}} to cover the multipath extension
defined in {{QUIC-MULTIPATH}}. The event schema URN for this extension is
`urn:ietf:params:qlog:events:quic#multipath`. The fragment `multipath`
identifies the extension within the `quic` namespace, as specified in
{{Section 8.1 of QLOG-MAIN}}.


## Draft Event Schema Identification

This section is to be removed before publishing as an RFC.

Only implementations of the published RFC can use the unversioned URN
defined in {{event-schema}}. Until such an RFC exists,
implementations MUST NOT identify themselves as using this URI.

Implementations of draft versions of this document MUST
append a hyphen and the corresponding draft number to that URN. For
exampledraft 00 of this document is identified using the URI
`urn:ietf:params:qlog:events:quic#multipath-00`.

# Adding a path identifier to event headers {#path-id-def}

The top-level `path_id` identifies the path on which a packet was sent or
received. It appears alongside `time`, `name`, and `data` in the qlog
`Event` map ({{Section 7 of QLOG-MAIN}}). {{add-path-id}} specifies when
to include it.

~~~ cddl
MultipathEventFields = (? path_id: uint32)
~~~

`path_id` is the Path ID defined in {{QUIC-MULTIPATH}}. It
is scoped to the connection and represented by a `uint32` in the log.

`path_id` is distinct from qlog's `tuple` / `tuple_id`. A path ID identifies a
logical QUIC path; multiple path IDs can use the same 4-tuple, and one Path ID
can use a new 4-tuple after migration. Implementations MUST NOT infer a path
ID from a tuple identifier.  Tuple details can
be recorded with `quic:tuple_assigned` (see {{QLOG-QUIC}}).

For example, a `packet_lost` event can identify the original sending path
even when it omits the optional packet header. The base qlog schema also
uses an optional top-level `tuple` field to associate an event with a network tuple
({{Section 7.2 of QLOG-MAIN}}).


# Multipath specific events and event extensions {#mp-events}

The three events in this section have Base importance. They record
changes to the paths of a connection as seen by the endpoint. Packet
events record the frames that carry path management signals. A single
change can involve several frames, and a local decision can precede the
first frame that signals it.

Each of these events MUST identify the affected path in `data.path_id`.
If receiving a packet triggers the event, the top-level `path_id`
identifies the path on which that packet arrived, as specified in
{{add-path-id}}. A local decision made independently of a packet has
only the affected Path ID in `data`.

The event data types extend the qlog main schema as follows. The
optional `error_code` and `sequence_number` fields use the qlog
`uint64` type. Their values are limited to 0 through 2^62 - 1 by
{{Sections 4.2 and 4.3 of QUIC-MULTIPATH}}.

~~~ cddl
$ProtocolEventData /= QUICPathCreated / QUICPathAbandoned /
                      QUICPathStatusUpdated
~~~

## path_created

`quic:path_created` is emitted once when the endpoint starts using a Path ID
for a new path. For an additional path, the client logs the event when it
initiates the path and the server logs it when it accepts the attempt and
sets up the path locally ({{Section 3.1 of QUIC-MULTIPATH}}). Validation can
still be pending. For the initial path, the endpoint logs the event with
`data.path_id` 0 when the handshake completes with multipath negotiated.
Issuing connection IDs for future paths alone does not trigger this event.
Migration to a new tuple retains the existing Path ID and can be logged
with `quic:migration_state_updated`.

~~~ cddl
QUICPathCreated = {
    path_id: uint32
    * $$quic-pathcreated-extension
}
~~~

## path_abandoned

`quic:path_abandoned` is emitted once when the endpoint abandons the
indicated Path ID under {{Section 3.4 of QUIC-MULTIPATH}}. The `initiator` is
`local` when the endpoint decides to close the path and `remote` when it
acts on the peer's PATH_ABANDON. These values use the `Initiator` type
from {{QLOG-QUIC}}. If known, `error_code` records
the numeric PATH_ABANDON error code for that decision.

An unused Path ID can be abandoned without a preceding `path_created`.
A repeated or reciprocal PATH_ABANDON and the eventual release of retained
path state do not produce additional abandonment events.

~~~ cddl
QUICPathAbandoned = {
    path_id: uint32
    initiator: Initiator
    ? error_code: uint64 .le 4611686018427387903
    * $$quic-pathabandoned-extension
}
~~~

## path_status_updated

`quic:path_status_updated` is emitted when the endpoint changes the path
usage preference it advertises (`initiator` is `local`), or records a
change in the peer's advertised preference
(see {{Section 3.3 of QUIC-MULTIPATH}}). In the latter case,
`initiator` is `remote`.

`new` gives the updated preference. If included, `old` gives the previous
preference for the same Path ID and initiator. Omit `old` when no previous
preference is known. The first advertisement can precede path creation.
`sequence_number`, if present, is the Path Status Sequence Number of the
advertisement. An ignored out-of-order or duplicate advertisement, or a
newer advertisement that repeats the current preference, produces no
status change event ({{Section 4.3 of QUIC-MULTIPATH}}).

~~~ cddl
QUICPathStatusUpdated = {
    path_id: uint32
    initiator: Initiator
    ? old: QUICPathStatus
    new: QUICPathStatus
    ? sequence_number: uint64 .le 4611686018427387903
    * $$quic-pathstatusupdated-extension
}

QUICPathStatus = "available" / "backup"
~~~

## Example

In this excerpt from a client's event list, the client opens path 1.
It then receives the peer's first preference for path 1 in a packet on
path 0. Later the client abandons path 1 at the application's request.

~~~ json
[
  { "time": 12.0, "name": "quic:path_created",
    "data": { "path_id": 1 } },
  { "time": 17.9, "name": "quic:packet_received", "path_id": 0,
    "data": {
      "header": { "packet_type": "1RTT", "packet_number": 8 },
      "frames": [ { "frame_type": "path_status_backup",
                    "path_id": 1, "sequence_number": 0 } ] } },
  { "time": 18.0, "name": "quic:path_status_updated", "path_id": 0,
    "data": { "path_id": 1, "initiator": "remote",
              "new": "backup", "sequence_number": 0 } },
  { "time": 97.0, "name": "quic:path_abandoned",
    "data": { "path_id": 1, "initiator": "local",
              "error_code": 62 } }
]
~~~

# Add a Path ID to existing QUIC events {#add-path-id}

Once the handshake completes with multipath negotiated
({{Section 2 of QUIC-MULTIPATH}}), an event that describes 1-RTT packets
on a single known path MUST include that path in the top-level `path_id`,
including path 0. For an event that records a state change, this rule
applies to the packet that triggered the change. Implementations MUST
omit the field in all other cases, including for Initial, Handshake,
0-RTT, and Retry packets and for local actions independent of any packet.

Omission does not mean path 0. An implementation MAY
place `path_id` in `common_fields` only when the same value applies to every
event in that trace, following the common-field rules in {{QLOG-MAIN}}.

A logged frame MUST retain any Path ID it carries on the wire
({{multipath-frames}}). For example, when a PATH_ACK arrives on path 0
and acknowledges packets sent on path 1, the `packet_received` event has
`path_id: 0` and the frame in `data.frames` has `path_id: 1`.

For the following events in the `quic` namespace, the packet path is
determined as follows:

| Event | Packet path |
| --- | --- |
| `packet_sent`, `packet_received` | Path carrying the packet. |
| `packet_lost` | Path carrying the lost packet. |
| `marked_for_retransmit` | Original sending path of the packet whose data is marked for retransmission (see {{cross-path-retransmission}}). |
| `packet_dropped`, `packet_buffered` | Path associated with the packet, if it can be identified at that point. |
| `packets_acked` | Receiving path of the packet carrying the ACK or PATH_ACK. |
| `frames_parsed`, `frames_created` | Path associated with the packet or packets. Implementations MUST split an event if its packet numbers span different paths. Omit `path_id` for frames created before a path is selected. |

In `packets_acked`, `data.path_id` identifies the original sending path of
the acknowledged 1-RTT packets. Implementations MUST include it when that
path is known and multipath has been negotiated. Otherwise, they MUST
omit it. Implementations MUST group acknowledged packets into separate
events if the value or presence of `data.path_id` differs between packets.
In the PATH_ACK example above, the resulting `packets_acked` event has top-level
`path_id: 0` and `data.path_id: 1`.

For the state updates listed below, `data.path_id` identifies the path
whose state is reported. Implementations MUST include it when the update
applies to one known path and MUST omit it for connection-wide updates or
when the affected path is unknown. For example, when a local timer for
path 1 expires, `timer_updated` has `data.path_id: 1` and no top-level
`path_id`.

~~~ cddl
$$quic-packetsacked-extension           //= (path_id: uint32)
$$quic-recoverymetricsupdated-extension //= (path_id: uint32)
$$quic-recoveryparametersset-extension  //= (path_id: uint32)
$$quic-congestionstateupdated-extension //= (path_id: uint32)
$$quic-ecnstateupdated-extension        //= (path_id: uint32)
$$quic-timerupdated-extension           //= (path_id: uint32)
$$quic-mtuupdated-extension             //= (path_id: uint32)
$$quic-spinbitupdated-extension         //= (path_id: uint32)
$$quic-connectionidupdated-extension    //= (path_id: uint32)
$$quic-tupleassigned-extension          //= (path_id: uint32)
$$quic-migrationstateupdated-extension  //= (path_id: uint32)
~~~

## Correlating 1-RTT packet events across paths

To match send, receive, and loss events for a 1-RTT packet within one
connection and sending direction, use the top-level `path_id` and the
packet number ({{Section 1 of QUIC-MULTIPATH}}). For packet numbers in
`packets_acked`, use `data.path_id` to select their packet number space.

The client logs two sends and one loss.

~~~ json
[
  { "time": 10.0, "name": "quic:packet_sent", "path_id": 0,
    "data": { "header": {
      "packet_type": "1RTT", "packet_number": 42 } } },
  { "time": 11.0, "name": "quic:packet_sent", "path_id": 1,
    "data": { "header": {
      "packet_type": "1RTT", "packet_number": 42 } } },
  { "time": 90.0, "name": "quic:packet_lost", "path_id": 0,
    "data": { "header": {
      "packet_type": "1RTT", "packet_number": 42 } } }
]
~~~

The server logs one receive event.

~~~ json
{ "time": 25.0, "name": "quic:packet_received", "path_id": 1,
  "data": { "header": {
    "packet_type": "1RTT", "packet_number": 42 } } }
~~~

The server's receive event matches the client's send event for `(1, 42)`.
The client's loss event matches its send event for `(0, 42)`.

## Cross-path retransmission {#cross-path-retransmission}

In `quic:marked_for_retransmit`, `path_id` identifies the path that carried
the lost packet, and in `quic:packet_sent`, it identifies the path carrying
the new packet with the retransmitted data.

For example, if a packet sent on path 1 is declared lost and its STREAM data is
retransmitted on path 0, the `quic:packet_lost` and `quic:marked_for_retransmit`
events both use `path_id` 1, while the `quic:packet_sent` event for the
retransmission uses `path_id` 0.

# Multipath transport parameter {#mp-parameters}

In `quic:parameters_set`, the `initial_max_path_id` field records the value
of the transport parameter defined in {{Section 2.1 of QUIC-MULTIPATH}}. Include
this field only if the endpoint identified by `initiator` advertised the
parameter.

~~~ cddl
$$quic-parametersset-extension //= (
    ? initial_max_path_id: uint32
)
~~~

## Example

In this excerpt from a client's event list, the client advertises a
maximum Path ID of 3 and the server advertises 1.

~~~ json
[
  { "time": 0.0, "name": "quic:parameters_set",
    "data": { "initiator": "local", "initial_max_path_id": 3 } },
  { "time": 31.2, "name": "quic:parameters_set",
    "data": { "initiator": "remote", "initial_max_path_id": 1 } }
]
~~~

# Multipath specific frames {#multipath-frames}

The following CDDL extends the `$QuicFrame` type socket defined in
{{Section 8.13 of QLOG-QUIC}}. These frame objects
can appear wherever a QUIC event records frames, including the `frames`
array of `quic:packet_sent` and `quic:packet_received`. The optional
`raw` field uses `RawInfo` from {{QLOG-MAIN}}.

Within a frame, `path_id` is the Path Identifier carried on the wire.
For example, a PATH_ACK sent on path 0 can acknowledge packets on path 1.
In that case the PATH_ACK object contains `"path_id": 1`. Path IDs and
path ID limits are unsigned 32-bit values. QUIC variable-length integer
fields are logged as decoded values within the limits of {{QUIC-MULTIPATH}}.

~~~ cddl
$QuicFrame /= PathAckFrame / PathAbandonFrame / PathStatusFrame /
              PathNewConnectionIDFrame /
              PathRetireConnectionIDFrame /
              MaxPathIDFrame / PathsBlockedFrame /
              PathCIDsBlockedFrame
~~~

## PATH_ACK

Both wire types in {{Section 4.1 of QUIC-MULTIPATH}} use the frame type
`path_ack`. The `path_id` selects the packet number space being
acknowledged. As in the base `AckFrame`, `ack_delay` is expressed in
milliseconds after applying the ACK delay exponent. The `acked_ranges`
field contains decoded, inclusive packet number ranges. For example,
`[[40, 42], [48]]` acknowledges packets 40 through 42 and packet 48.

For wire type 0x3e, omit the ECN counts. For 0x3f, include all three
cumulative counts for the acknowledged path, including counts of zero.

~~~ cddl
PathAckFrame = { PathAckFields } /
               { PathAckFields, ect0: uint64, ect1: uint64,
                 ce: uint64 }

PathAckFields = (
    frame_type: "path_ack"
    path_id: uint32
    ? ack_delay: float32
    ? acked_ranges: [+ AckRange]
    ? raw: RawInfo
)
~~~

## PATH_ABANDON

For PATH_ABANDON (0x3e75), `path_id` identifies the path being closed.
The `error` field uses qlog transport error names, including `no_error`
for code 0. The four additional names below correspond to the codes in
{{Section 4.2.1 of QUIC-MULTIPATH}}. For a code with no known name, log
`"error": "unknown"` and include its numeric value in `error_code`.

~~~ cddl
PathAbandonFrame = {
    frame_type: "path_abandon"
    path_id: uint32
    error: $TransportError / CryptoError
    ? error_code: uint64
    ? raw: RawInfo
}

$TransportError /= "application_abandon_path" /      ; 0x3e
                   "path_resource_limit_reached" /   ; 0x3e75
                   "path_unstable_or_poor" /         ; 0x3e76
                   "no_cid_available_for_path"       ; 0x3e77
~~~

## PATH_STATUS_AVAILABLE and PATH_STATUS_BACKUP

The `frame_type` identifies the preference advertised for `path_id`.
Use `path_status_available` for 0x3e77 and `path_status_backup` for 0x3e76
({{Section 4.3 of QUIC-MULTIPATH}}).

~~~ cddl
PathStatusFrame = {
    frame_type: "path_status_available" / "path_status_backup"
    path_id: uint32
    sequence_number: uint64
    ? raw: RawInfo
}
~~~

## PATH_NEW_CONNECTION_ID and PATH_RETIRE_CONNECTION_ID

These frames use the fields of the base `NewConnectionIDFrame` and
`RetireConnectionIDFrame`, with an additional `path_id`. Both the
connection ID sequence number and the retirement threshold apply to the
indicated path ({{Sections 4.4 and 4.5 of QUIC-MULTIPATH}}). For
PATH_NEW_CONNECTION_ID (0x3e78), `connection_id_length`, when present,
is the length in bytes of the connection ID on the wire. For
PATH_RETIRE_CONNECTION_ID (0x3e79), the pair (`path_id`,
`sequence_number`) identifies the connection ID to retire.

~~~ cddl
PathNewConnectionIDFrame = {
    frame_type: "path_new_connection_id"
    path_id: uint32
    sequence_number: uint64
    retire_prior_to: uint64
    ? connection_id_length: uint8
    connection_id: ConnectionID
    ? stateless_reset_token: StatelessResetToken
    ? raw: RawInfo
}

PathRetireConnectionIDFrame = {
    frame_type: "path_retire_connection_id"
    path_id: uint32
    sequence_number: uint64
    ? raw: RawInfo
}
~~~

## MAX_PATH_ID, PATHS_BLOCKED, and PATH_CIDS_BLOCKED

MAX_PATH_ID (0x3e7a) advertises the largest acceptable Path ID in
`maximum`. PATHS_BLOCKED (0x3e7b) reports the Path ID limit in effect
at the sender in `limit`. Both fields represent the Maximum Path
Identifier on the wire. PATH_CIDS_BLOCKED (0x3e7c) identifies the path
that lacks an unused connection ID and reports the next connection ID
sequence number expected from the peer
({{Sections 4.6 and 4.7 of QUIC-MULTIPATH}}).

~~~ cddl
MaxPathIDFrame = {
    frame_type: "max_path_id"
    maximum: uint32
    ? raw: RawInfo
}

PathsBlockedFrame = {
    frame_type: "paths_blocked"
    limit: uint32
    ? raw: RawInfo
}

PathCIDsBlockedFrame = {
    frame_type: "path_cids_blocked"
    path_id: uint32
    next_sequence_number: uint64
    ? raw: RawInfo
}
~~~

## Example

In this packet sent on path 0, the PATH_ACK acknowledges packets on
path 1 and the PATH_CIDS_BLOCKED reports that the sender has run out
of connection IDs for path 2.

~~~ json
{
  "time": 16.5,
  "name": "quic:packet_sent",
  "path_id": 0,
  "data": {
    "header": { "packet_type": "1RTT", "packet_number": 90 },
    "frames": [
      { "frame_type": "path_ack", "path_id": 1,
        "acked_ranges": [[40, 42], [48]],
        "ect0": 4, "ect1": 0, "ce": 0 },
      { "frame_type": "path_cids_blocked", "path_id": 2,
        "next_sequence_number": 3 }
    ]
  }
}
~~~

# Security Considerations {#security}

The security and privacy considerations discussed in {{Section 14 of QLOG-MAIN}}
apply to this document as well.

# IANA Considerations {#iana}

This document registers a new entry in the "qlog event schema URIs"
registry defined in {{Section 15 of QLOG-MAIN}}.

Event schema URI:  `urn:ietf:params:qlog:events:quic#multipath`

Namespace:  `quic`

Event Types:  `path_created`, `path_abandoned`, `path_status_updated`

Description:  Event definitions related to the extensions to
the QUIC transport protocol defined in {{QUIC-MULTIPATH}}.

Reference:  {{mp-events}} of this document.

This document modifies the following QUIC events: `packet_sent`, `packet_received`, `packet_dropped`,
`packet_buffered`, `packets_acked`, `packet_lost`, `marked_for_retransmit`,
`recovery_metrics_updated`,
`recovery_parameters_set`, `congestion_state_updated`, `ecn_state_updated`,
`timer_updated`, `mtu_updated`, `spin_bit_updated`, `connection_id_updated`,
`tuple_assigned`, `migration_state_updated`, `frames_parsed`, and
`frames_created`.

This document adds the `initial_max_path_id` field to the
`parameters_set` event ({{mp-parameters}}).

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
