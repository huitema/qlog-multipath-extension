---
title: "QLOG extensions for Multiple Paths for a QUIC Connection"
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
    fullname: Altanai Bisht
    organization: Cisco Systems
    email: albisht@cisco.com


normative:

   RFC9000:
   RFC10069:
   I-D.ietf-quic-qlog-quic-events:
   I-D.ietf-quic-qlog-main-schema:


informative:

...

--- abstract

This draft define the QLOG elements that have to be added to
the definition of QLOG events for QUIC connections.


--- middle

# Introduction {#intro}

The QLOG drafts define the syntax of QLOG {{I-D.ietf-quic-qlog-main-schema}}
and the events and frames for logging QUIC events {{I-D.ietf-quic-qlog-quic-events}}
found in QUIC connections per {{RFC9000}}. The extensions to {{RFC9000}} for
managing multiple paths for a QUIC connections are defined in {{RFC10069}},
which introduces new concepts such as path, new events such as creating or
abandoning paths, and new frames for managing traffic when using multiple
paths. This draft defines the corresponding QLOG extensions.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Event Schema Definition

This document describes how the extend extensions to the core QUIC protocol
defined in {{RFC10069}} can be expressed by extending the schema
defined in {{I-D.ietf-quic-qlog-quic-events}}.
Per the requirements in Section 8 of {{I-D.ietf-quic-qlog-main-schema}},
this document registers extensions to the quic namespace.
The event schema URI is: urn:ietf:params:qlog:events:quic-multipath.


## Draft Event Schema Identification

This section is to be removed before publishing as an RFC.

   Only implementations of the final, published RFC can use the events
   belonging to the event schema with the URI
   urn:ietf:params:qlog:events:quic-multipath.  Until such an RFC exists,
   implementations MUST NOT identify themselves using this URI.

   Implementations of draft versions of the event schema MUST append the
   string "-" and the corresponding draft number to the URI.  For
   example, draft 00 of this document is identified using the URI
   urn:ietf:params:qlog:events:quic-multipath-00.

# Multipath specific events and event extensions

TODO.

# Multipath specific frames

TODO. Need to extract the list of frames from RFC 10069.

# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}


This document registers a new entry in the "qlog event schema URIs"
registry, which was created in Section 15 of [QLOG-MAIN]:

   Event schema URI:  urn:ietf:params:qlog:events:quic-multipath

Event-types: TODO. List the new events that are created here.

Description:  Event definitions related to the extensions to
the QUIC transport protocol defined in {{RFC10069}}.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
