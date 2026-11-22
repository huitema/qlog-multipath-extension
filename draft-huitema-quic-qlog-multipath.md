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

   RFC9000:
   RFC10069:
   I-D.ietf-quic-qlog-main-schema:
   I-D.ietf-quic-qlog-quic-events:


informative:
  DebuggingQuic:
    title: "Debugging QUIC and HTTP/3 with qlog and qvis"
    date: "2020"
    seriesinfo: "NRW '20: Proceedings of the 2020 Applied Networking Research Workshop"
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

The QLOG specifications defines structured logs for QUIC connections,
and has proven very useful to debug QUIC implementations and deployments.
This draft extends the QLOG specification for QUIC events. It add a
Path ID element to event headers, defines new events to log the
creation and management of multiple paths, and specifies the logging format
of new frames introduced in RFC 10069.

--- middle

# Introduction {#intro}

The QLOG drafts define the syntax of QLOG {{I-D.ietf-quic-qlog-main-schema}}
and the events and frames for logging QUIC events {{I-D.ietf-quic-qlog-quic-events}}
found in QUIC connections per {{RFC9000}}. The extensions to {{RFC9000}} for
managing multiple paths for a QUIC connections are defined in {{RFC10069}},
which introduces new concepts such as path, new events such as creating or
abandoning paths, and new frames for managing traffic when using multiple
paths.

This draft defines an extension to the QUIC events specification in
{{I-D.ietf-quic-qlog-quic-events}} to enable logging of QUIC connections that manage
multiple paths according to {{RFC10069}}.

Debugging tools like qvis have proven very useful when debugging QUIC implementations
{{DebuggingQuic}}. This document provides a standard way to add specific information
in QLOG traces so updated tools can properly represent QUIC connections that use
multiple paths according to {{RFC10069}}. For example, a key function of these
tools is to correlate packet departures and the arrival of the corresponding
acknowledgement. For plain QUIC connections, this can be done by examining
packet numbers. With multipath, there is a packet number space per path, and
correlation requires examining both packet numbers and the ID of the path on which the
packet was sent. The required logging information is defined in this document.




This document defines how a `path_id` can be added to event headers in {{path-id-def}},
adds events describing the management of paths in {{mp-events}},
explains the effects of adding a `path_id` to events defined in
{{I-D.ietf-quic-qlog-quic-events}} in {{add-path-id}},
specifies the logging of multipath specific frames in {{multipath-frames}},
and that of the multipath transport parameter in {{mp-parameters}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Event Schema Definition {#event-schema}

This document extends the `quic` namespace of
{{I-D.ietf-quic-qlog-quic-events}} to cover the multipath extension
defined in {{RFC10069}}. The event schema URN for this extension is
`urn:ietf:params:qlog:events:quic#multipath`. The fragment `multipath`
identifies the extension within the `quic` namespace, as specified in
{{Section 8.1 of I-D.ietf-quic-qlog-main-schema}}.


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

Managing multiple paths per connection requires identifying these paths.
Per {{RFC10069}}, this is done using a path ID. This path ID
will be added to events as specified in {{mp-events}} and {{add-path-id}}.
Each of these events MUST include the following field in its top-level
`Event` map alongside `time`, `name`, and `data`
({{Section 7 of I-D.ietf-quic-qlog-main-schema}}).

~~~ cddl
MultipathEventFields = (path_id: uint32)
~~~

`path_id` identifies the affected path within the connection. It is an
unsigned 32-bit integer in the range 0 to 4294967295 inclusive
({{Section 2.1 of RFC10069}}).

The value of `path_id` is the Path ID defined in {{RFC10069}}. It is scoped to
the QUIC connection, is shared by both directions, starts at 0 for the
initial path, increases monotonically, and is not reused. The range is
0..2^32-1, inclusive. The field is unsigned: negative values are invalid.

`path_id` is distinct from qlog's `tuple` / `tuple_id`. A Path ID identifies a
logical QUIC path; multiple Path IDs can use the same 4-tuple, and one Path ID
can use a new 4-tuple after migration. Implementations MUST NOT infer a Path
ID from a tuple identifier.  Tuple details can
be recorded with `quic:tuple_assigned` (see {{I-D.ietf-quic-qlog-quic-events}}).

Adding a path ID to an event header indicates that this event is related to
a specific path. This is explained in {{add-path-id}}.

# Multipath specific events and event extensions {#mp-events}

The three events in this section have Base importance. They record
changes to the paths of a connection as seen by the endpoint. Packet
events record the frames that carry path management signals. A single
change can involve several frames, and a local decision can precede the
first frame that signals it.

All the events described in this sections affect a single path. The
corresponding `path_id` MUST be included in the respective event headers,
as specified in {{path-id-def}}.

The event data types extend the qlog main schema as follows. The
optional `error_code` and `sequence_number` fields use the qlog
`uint64` type. Their values are limited to 0 through 2^62 - 1 by
{{Sections 4.2 and 4.3 of RFC10069}}.

~~~ cddl
$ProtocolEventData /= QUICPathCreated / QUICPathAbandoned /
                      QUICPathStatusUpdated
~~~

## path_created

`quic:path_created` is emitted once when the endpoint starts using a Path ID
for a new path. For an additional path, the client logs the event when it
initiates the path and the server logs it when it accepts the attempt and
sets up the path locally ({{Section 3.1 of RFC10069}}). Validation can
still be pending. For the initial path, the endpoint logs the event with
`path_id` 0 when the handshake completes with multipath negotiated.
Issuing connection IDs for future paths alone does not trigger this event.
Migration to a new tuple retains the existing Path ID and can be logged
with `quic:migration_state_updated`.

~~~ cddl
QUICPathCreated = {
    * $$quic-pathcreated-extension
}
~~~

## path_abandoned

`quic:path_abandoned` is emitted once when the endpoint abandons the
indicated Path ID under {{Section 3.4 of RFC10069}}. The `initiator` is
`local` when the endpoint decides to close the path and `remote` when it
acts on the peer's PATH_ABANDON. These values use the `Initiator` type
from {{I-D.ietf-quic-qlog-quic-events}}. If known, `error_code` records
the numeric PATH_ABANDON error code for that decision.

An unused Path ID can be abandoned without a preceding `path_created`.
A repeated or reciprocal PATH_ABANDON and the eventual release of retained
path state do not produce additional abandonment events.

~~~ cddl
QUICPathAbandoned = {
    initiator: Initiator
    ? error_code: uint64 .le 4611686018427387903
    * $$quic-pathabandoned-extension
}
~~~

## path_status_updated

`quic:path_status_updated` is emitted when the endpoint changes the path
usage preference it advertises (`initiator` is `local`), or records a
change in the peer's advertised preference (`initiator` is `remote`).
Each endpoint expresses its own preference for how its peer should use
the path ({{Section 3.3 of RFC10069}}). A sender can choose how to
schedule packets independently of the advertised preference.

`new` gives the updated preference. If included, `old` gives the previous
preference for the same Path ID and initiator. Omit `old` when no previous
preference is known. The first advertisement can precede path creation.
`sequence_number`, if present, is the Path Status Sequence Number of the
advertisement. An ignored out-of-order or duplicate advertisement, or a
newer advertisement that repeats the current preference, produces no
status change event ({{Section 4.3 of RFC10069}}).

~~~ cddl
QUICPathStatusUpdated = {
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
    "path_id": 1, "data": {} },
  { "time": 18.0, "name": "quic:path_status_updated",
    "path_id": 1, "data": { "initiator": "remote", "new": "backup",
                            "sequence_number": 0 } },
  { "time": 97.0, "name": "quic:path_abandoned",
    "path_id": 1, "data": { "initiator": "local",
                            "error_code": 62 } }
]
~~~


# Add a Path identifier to existing QUIC events {#add-path-id}

When a QUIC event describes an operation or state for one known path,
implementations MUST include `path_id`, including when its value is 0.
(The `path_id` field is defined in {{path-id-def}}.)
Implementations
MUST omit the field when the event is connection-wide or when the associated
Path ID is not known. Omission does not mean path 0. An implementation MAY
place `path_id` in `common_fields` only when the same value applies to every
event in that trace, following the common-field rules in {{I-D.ietf-quic-qlog-main-schema}}.

The event-level field identifies the path on which the event's packet or
operation applies. A Path ID carried inside a frame remains a frame field and
can identify a different path. For example, a packet sent on path 0 can carry
a PATH_ACK frame acknowledging packets from path 1; the event has
`path_id: 0`, while the PATH_ACK frame has `path_id: 1`.

The field applies to the following existing events in the `quic` namespace
when the stated path context is known:

| Event | Meaning of `path_id` |
| --- | --- |
| `packet_sent`, `packet_received` | Path carrying the packet. |
| `packet_lost` | Path carrying the lost packet. |
| `packet_dropped`, `packet_buffered` | Path associated with the packet, if it can be identified at that point. |
| `packets_acked` | Path of the acknowledged packets, not the path carrying the ACK or PATH_ACK frame. Implementations MUST split events when acknowledged packets belong to different paths. |
| `recovery_metrics_updated`, `recovery_parameters_set` | Path whose recovery or congestion-control values are reported. |
| `congestion_state_updated`, `ecn_state_updated`, `timer_updated` | Path whose state or timer changed. Omit it for a connection-wide update. |
| `mtu_updated`, `spin_bit_updated` | Path whose MTU or spin-bit state changed. |
| `connection_id_updated` | Path associated with the connection ID being updated. |
| `tuple_assigned` | Path only when the tuple assignment is for one specific Path ID; the tuple itself remains identified by `tuple_id`. |
| `migration_state_updated` | Path being migrated. Its Path ID remains the same as the tuple changes during migration. |
| `frames_parsed`, `frames_created` | Path associated with the packet or packets. Implementations MUST split an event if its packet numbers span different paths. Omit `path_id` for frames created before a path is selected. |

Path-management events defined by this document MUST include `path_id` when
they refer to one path. For a frame whose RFC 10069 definition contains a
Path ID, the frame's `data` representation MUST also retain that frame field;
the event-level `path_id` does not replace it. For other QUIC events,
implementations MUST include `path_id` when they describe a single known
path, and MUST omit it when they are connection-wide or the path is unknown.

The field is outside `data`, so existing QUIC event data definitions do not
need to be copied or redefined for each event. The table above specifies the
event-by-event semantics and applicability of the common field.
{{mp-events}} defines the new events. {{multipath-frames}}
defines the frame encodings.

## Correlating 1-RTT packet events across paths

Every event that identifies a specific 1-RTT packet MUST carry its
event-level `path_id`, even on path 0. For the packet number spaces
in {{Section 1 of RFC10069}}, match 1-RTT events by
(`path_id`, `packet_number`) within one connection and sending direction.

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

The `path_id` is set to 0 for Initial, Handshake, 0-RTT, and Retry packet events.

# Multipath transport parameter {#mp-parameters}

The `initial_max_path_id` transport parameter (0x3e) is logged in
`quic:parameters_set`. The `initiator` field of that event identifies
the endpoint that advertised the value. The multipath extension is
enabled only if both endpoints advertise the parameter
({{Section 2 of RFC10069}}).

~~~ cddl
$$quic-parametersset-extension //= (
    ? initial_max_path_id: uint32
)
~~~

A value of 0 enables the extension without allowing extra paths
immediately. It is distinct from an absent field, which means that the
endpoint did not advertise the parameter.

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
{{Section 8.13 of I-D.ietf-quic-qlog-quic-events}}. These frame objects
can appear wherever a QUIC event records frames, including the `frames`
array of `quic:packet_sent` and `quic:packet_received`. The optional
`raw` field uses `RawInfo` from {{I-D.ietf-quic-qlog-main-schema}}.

Within a frame, `path_id` is the Path Identifier carried on the wire.
For example, a PATH_ACK sent on path 0 can acknowledge packets on path 1.
In that case the PATH_ACK object contains `"path_id": 1`. Path IDs and
path ID limits are unsigned 32-bit values. QUIC variable-length integer
fields are logged as decoded values within the limits of {{RFC10069}}.

~~~ cddl
$QuicFrame /= PathAckFrame / PathAbandonFrame / PathStatusFrame /
              PathNewConnectionIDFrame /
              PathRetireConnectionIDFrame /
              MaxPathIDFrame / PathsBlockedFrame /
              PathCIDsBlockedFrame
~~~

## PATH_ACK

Both wire types in {{Section 4.1 of RFC10069}} use the frame type
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
{{Section 4.2.1 of RFC10069}}. For a code with no known name, log
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
Use `path_status_available` for 0x3e77 and `path_status_backup` for 0x3e76.
Both use the same per-path status sequence number space
({{Section 4.3 of RFC10069}}). The frame object records the received
sequence number even when the receiver ignores an outdated advertisement.

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
indicated path ({{Sections 4.4 and 4.5 of RFC10069}}). For
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
({{Sections 4.6 and 4.7 of RFC10069}}).

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

TODO Security


# IANA Considerations {#iana}

This document registers a new entry in the "qlog event schema URIs"
registry defined in {{Section 15 of I-D.ietf-quic-qlog-main-schema}}.

Event schema URI:  `urn:ietf:params:qlog:events:quic#multipath`

Namespace:  `quic`

Event Types:  `path_created`, `path_abandoned`, `path_status_updated`

Description:  Event definitions related to the extensions to
the QUIC transport protocol defined in {{RFC10069}}.

Reference:  {{mp-events}} of this document.

This document modifies the following QUIC events: `packet_sent`, `packet_received`, `packet_dropped`,
`packet_buffered`, `packets_acked`, `packet_lost`, `recovery_metrics_updated`,
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
