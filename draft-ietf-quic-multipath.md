---
title: Multipath Extension for QUIC
abbrev: Multipath QUIC
docname: draft-ietf-quic-multipath-latest
date: {DATE}
category: std

ipr: trust200902
area: Transport
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
-
  fullname:
      :: "刘彦梅"
      ascii: "Yanmei Liu"
  role: editor
  org: Alibaba Inc.
  email: miaoji.lym@alibaba-inc.com
-
   fullname:
     :: "马云飞"
     ascii: "Yunfei Ma"
   org: Alibaba Inc.
   email: yunfei.ma@alibaba-inc.com
-
   ins: Q. De Coninck
   name: Quentin De Coninck
   role: editor
   org: UCLouvain
   email: quentin.deconinck@uclouvain.be
-
   ins: O. Bonaventure
   name: Olivier Bonaventure
   org: UCLouvain and Tessares
   email: olivier.bonaventure@uclouvain.be
-
   ins: C. Huitema
   name: Christian Huitema
   org: Private Octopus Inc.
   email: huitema@huitema.net
-
   ins: M. Kuehlewind
   name: Mirja Kuehlewind
   role: editor
   org: Ericsson
   email: mirja.kuehlewind@ericsson.com


normative:
  RFC2119:
  QUIC-TRANSPORT: rfc9000
  QUIC-TLS: rfc9001
  QUIC-RECOVERY: rfc9002

informative:
  RFC6356:
  QUIC-Timestamp: I-D.huitema-quic-ts
  OLIA:
    title: "MPTCP is not pareto-optimal: performance issues and
a possible solution"
    date: "2012"
    seriesinfo: "Proceedings of the 8th international conference on
Emerging networking experiments and technologies, ACM"
    author:
    -
      ins: R. Khalili
    -
      ins: N. Gast
    -
      ins: M. Popovic
    -
      ins: U. Upadhyay
    -
      ins: J.-Y. Le Boudec


--- abstract

This document specifies a multipath extension for the QUIC protocol to
enable the simultaneous usage of multiple paths for a single connection.

--- middle

# Introduction

This document specifies an extension to QUIC version 1 {{QUIC-TRANSPORT}}
to enable the simultaneous usage of multiple paths for a single
connection.

This proposal is based on several basic design points:

  * Re-use as much as possible mechanisms of QUIC version 1. In
particular, this proposal uses path validation as specified for QUIC
version 1 and aims to re-use as much as possible of QUIC's connection
migration.
  * Use the same packet header formats as QUIC version 1 to minimize
    the difference between multipath and non-multipath traffic being
    exposed on wire.
  * Congestion Control must be per-path (following {{QUIC-TRANSPORT}})
which usually also requires per-path RTT measurements
  * PMTU discovery should be performed per-path
  * The use of this multipath extension requires the use of non-zero
Connection IDs in both directions.
  * A path is determined by the 4-tuple of source and destination IP
address as well as source and destination port. Therefore, there can be
at most one active paths/connection ID per 4-tuple.
  * If the 4-tuple changes without the use of a new connection ID (e.g.
due to a NAT rebinding), this is considered as a migration event.

The path management specified in {{Section 9 of QUIC-TRANSPORT}}
fulfills multiple goals: it directs a peer to switch sending through
a new preferred path, and it allows the peer to release resources
associated with the old path. Multipath requires several changes to
that mechanism:

  *  Allow simultaneous transmission of non-probing frames on multiple
paths.
  *  Continue using an existing path even if non-probing frames have
been received on another path.
  *  Manage the removal of paths that have been abandoned.

As such, this extension specifies a departure from the specification of
path management in {{Section 9 of QUIC-TRANSPORT}} and therefore
requires negotiation between the two endpoints using a new transport
parameter, as specified in {{nego}}.

This extension uses multiple packet number spaces.
When multipath is negotiated,
each destination connection ID is linked to a separate packet number space.
Using multiple packet number spaces enables direct use of the
loss recovery and congestion control mechanisms defined in
{{QUIC-RECOVERY}}.

This specification
requires the sender to use a non-zero connection ID when opening an
additional path.
Some deployments of QUIC use zero-length connection IDs.
However, when a node selects to use zero-length connection IDs, it is not
possible to use different connection IDs for distinguishing packets
sent to that node over different paths.

Each endhost may use several IP addresses to serve the connection. In
particular, the multipath extension supports the following scenarios.

  * The client uses multiple IP addresses and the server listens on only
    one.
  * The client uses only one IP address and the server listens on
    several ones.
  * The client uses multiple IP addresses and the server listens on
    several ones.
    * The client uses only one IP address and the server
      listens on only one.

Note that in the last scenario, it still remains possible to have
multiple paths over the connection, given that a path is not only
defined by the IP addresses being used, but also the port numbers.
In particular, the client can use one or several ports per IP
address and the server can listen on one or several ports per IP
address.

This proposal does not cover address discovery and management. Addresses
and the actual decision process to setup or tear down paths are assumed
to be handled by the application that is using the QUIC multipath
extension. This is sufficient to address the first aforementioned
scenario. However, this document does not prevent future extensions from
defining mechanisms to address the remaining scenarios.
Further, this proposal only specifies a simple basic packet
scheduling algorithm, in order to provide some basic implementation
guidance. However, more advanced algorithms as well as potential
extensions to enhance signaling of the current path state are expected
as future work.

## Conventions and Definitions {#definition}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all
capitals, as shown here.

We assume that the reader is familiar with the terminology used in
{{QUIC-TRANSPORT}}. When this document uses the term "path", it refers
to the notion of "network path" used in {{QUIC-TRANSPORT}}.

# High-level overview {#overview}

The multipath extensions to QUIC proposed in this document enable the simultaneous utilization of
different paths to exchange non-probing QUIC frames for a single connection. This contrasts with
the base QUIC protocol {{QUIC-TRANSPORT}} that includes a connection migration mechanism that
selects only one path to exchange such frames.

A multipath QUIC connection starts with a QUIC handshake as a regular QUIC connection.
See further {{nego}}.
The peers use the enable_multipath transport parameter during the handshake to
negotiate the utilization of the multipath capabilities.
The active_connection_id_limit transport parameter limits the maximum number of active paths
that can be used during a connection. A multipath QUIC connection is thus an established QUIC
connection where the enable_multipath transport parameter
has been successfully negotiated.

To add a new path to an existing multipath QUIC connection, a client starts a path validation on
the chosen path, as further described in {{setup}}.
In this version of the document, a QUIC server does not initiate the creation
of a path, but it can validate a new path created by a client.
A new path can only be used once the associated 4-tuple has been validated
by ensuring that the peer is able to receive packets at that address
(see {{Section 8 of QUIC-TRANSPORT}}). The Destination Connection ID is used
to associate a packet to a packet number space that is used on a valid path. Further, the
sequence number of Destination Connection ID is used as numerical identifier
in control frames. E.g. an endpoint sends a PATH_ABANDON frame to request its peer to
abandon the path on which the sender uses the Destination Connection ID
with the sequence number contained in the PATH_ABANDON frame.

In addition to these core features, an application using Multipath QUIC will typically
need additional algorithms to handle the number of active paths and how they are used to
send packets. As these differ depending on the application's requirements, their
specification is out of scope of this document.

Using multiple packet number spaces requires changes in the way AEAD is
applied for packet protection, as explained in {{multipath-aead}},
and tighter constraints for key updates, as explained in {{multipath-key-update}}.

# Handshake Negotiation and Transport Parameter {#nego}

This extension defines a new transport parameter, used to negotiate
the use of the multipath extension during the connection handshake,
as specified in {{QUIC-TRANSPORT}}. The new transport parameter is
defined as follows:

- enable_multipath (current version uses 0x0f739bbc1b666d05): the
  enable_multipath transport parameter is included if the endpoint supports
  the multipath extension as defined in this document. This parameter has
  a zero-length value.

If any of the endpoints does not advertise the enable_multipath transport
parameter, then the endpoints MUST NOT use any frame or
mechanism defined in this document.

When advertising the enable_multipath transport parameter, the endpoint
MUST use non-zero source and destination connection IDs.
If an enable_multipath transport
parameter is received and the carrying packet contains a zero
length connection ID, the receiver MUST treat this as a connection error of type
MP_PROTOCOL_VIOLATION and close the connection.

The enable_multipath parameter MUST NOT be remembered
({{Section 7.4.1 of QUIC-TRANSPORT}}).
New paths can only be used after handshake completion.

This extension does not change the definition of any transport parameter
defined in {{Section 18.2. of QUIC-TRANSPORT}}.

The transport parameter "active_connection_id_limit"
{{QUIC-TRANSPORT}} limits the number of usable Connection IDs, and also
limits the number of concurrent paths. However, endpoints might prefer to retain
spare Connection IDs so that they can respond to unintentional migration events
({{Section 9.5 of QUIC-TRANSPORT}}).

Cipher suites with nonce shorter than 12 bytes cannot be used together with
the multipath extension. If such cipher suite is selected and the use of the
multipath extension is negotiated, endpoints MUST abort the handshake with a
TRANSPORT_PARAMETER error.


# Path Setup and Removal {#setup}

After completing the handshake, endpoints have agreed to enable
multipath feature. They can also start using multiple paths, unless both
server preferred addresses and a disable_active_migration transport parameter
were provided by the server, in which case a client is forbidden to establish
new paths until "after a client has acted on a preferred_address transport
parameter" ({{Section 18.2. of QUIC-TRANSPORT}}).

This document
does not specify how an endpoint that is reachable via several addresses
announces these addresses to the other endpoint. In particular, if the
server uses the preferred_address transport parameter, clients
cannot assume that the initial server address and the addresses
contained in this parameter can be simultaneously used for multipath
({{Section 9.6.2 of QUIC-TRANSPORT}}).
Furthermore, this document
does not discuss when a client decides to initiate a new path. We
delegate such discussion in separate documents.

To open a new path, an endpoint SHALL use different Connection IDs on different paths.
Still, the receiver may observe the same Connection ID used on different
4-tuples due to, e.g., NAT rebinding. In such case, the receiver reacts
as specified in {{Section 9.3 of QUIC-TRANSPORT}}.

This proposal adds two multipath control frames for path management:

- PATH_ABANDON frame for the receiver side to abandon the path
(see {{path-abandon-frame}})
- PATH_STATUS frame to express a preference in path usage
(see {{path-status-frame}}

All the new frames are sent in 1-RTT packets {{QUIC-TRANSPORT}}.

## Path Initiation

Connection IDs cannot be reused, thus opening a new path requires the
use of a new Connection ID (see {{Section 9.5 of QUIC-TRANSPORT}}).
Following {{QUIC-TRANSPORT}}, each endpoint uses NEW_CONNECTION_ID frames
to issue usable connections IDs to reach it. As such to open
a new path by initiating path validation, both sides need at least
one unused Connection ID (see {{Section 5.1.1 of QUIC-TRANSPORT}}).

If the transport parameter "active_connection_id_limit" is negotiated as N,
the server provided N Connection IDs, and the client is already actively
using N paths, the limit is reached. If the client wants to start
a new path, it has to retire one of the established paths.

When the multipath option is negotiated, clients that want to use an
additional path MUST first initiate the Address Validation procedure
with PATH_CHALLENGE and PATH_RESPONSE frames described in
{{Section 8.2 of QUIC-TRANSPORT}}. After receiving packets from the
client on a new path, if the server decides to use the new path,
the server MUST perform path validation ({{Section 8.2 of QUIC-TRANSPORT}})
unless it has previously validated that address.

If validation succeed, the client can send non-probing, 1-RTT packets
on the new paths.  In contrast with the specification in
{{Section 9 of QUIC-TRANSPORT}}, the server MUST NOT assume that
receiving non-probing packets on a new path indicates an attempt
to migrate to that path.  Instead, servers SHOULD consider new paths
over which non-probing packets have been received as available
for transmission.

As specified in {{Section 9.3 of QUIC-TRANSPORT}}, the server is expected to send a new
address validation token to a client following the successful validation of a
new client address. In situations where multiple paths are activated, the
client may be recipient of several tokens, each tied to a different address.
When considering using a token for subsequent connections, the client ought to
carefully select the token to use, due to the inherent ambiguity associated
with determining the exact address to which a token is bound. To alleviate such a
token ambiguity issue, a server may issue a token that is capable of validating
any of the previously validated addresses.

## Path State Management

An endpoint uses PATH_STATUS frames to inform that the peer should
send packets in the preference expressed by these frames.
Notice that the endpoint might not follow the peer’s advertisements,
but the PATH_STATUS frame is still a clear signal of suggestion
for the preference of path usage by the peer.

PATH_STATUS frame describes 2 kinds of path states:

- Mark a path as "available", i.e., allow the peer to use its own logic
  to split traffic among available paths.
- Mark a path as "standby", i.e., suggest that no traffic should be sent
  on that path if another path is available.

Endpoints use Destination Connection ID Sequence Number field
in PATH_STATUS frame to identify which path state is going to be
changed. Notice that PATH_STATUS frame
can be sent via a different path.

If all available path are marked as "standby", no guidance is provided about
which path should be used preferably.


## Path Close

Each endpoint manages the set of paths that are available for
transmission. At any time in the connection, each endpoint can decide to
abandon one of these paths, following for example changes in local
connectivity or changes in local preferences. After an endpoint abandons
a path, the peer will not receive any more non-probing packets on
that path. Non-probing packets are defined in {{Section 9.1 of QUIC-TRANSPORT}}.

An endpoint that wants to close a path SHOULD use explicit request to
terminate the path by sending the PATH_ABANDON frame (see
{{path-abandon-close}}). Note that while abandoning a path will cause
Connection ID retirement, only retiring the associated Connection ID
does not necessarily advertise path abandon (see {{retire-cid-close}}).
However, implicit signals such as idle time or packet losses might be
the only way for an endhost to detect path closure (see
{{idle-time-close}}).

Note that other explicit closing mechanisms of {{QUIC-TRANSPORT}} still
apply on the whole connection. In particular, the reception of either a
CONNECTION_CLOSE ({{Section 10.2 of QUIC-TRANSPORT}}) or a Stateless
Reset ({{Section 10.3 of QUIC-TRANSPORT}}) closes the connection.

### Use PATH_ABANDON Frame to Close a Path {#path-abandon-close}

Both endpoints, namely the client and the server, can initiate path closure,
by sending a PATH_ABANDON frame (see {{path-abandon-frame}}) which
requests the peer to stop sending packets with the corresponding Destination Connection ID.

When sending or receiving a PATH_ABANDON frame, endpoints SHOULD wait for at
least three times the current Probe Timeout (PTO) interval as defined in
{{Section 6.2 of QUIC-RECOVERY}} after the last packet was sent on the path,
before sending the RETIRE_CONNECTION_ID frame for the corresponding Connection
ID. This is inline with the requirement of {{Section 10.2 of QUIC-TRANSPORT}}
to ensure that paths close cleanly and that delayed or reordered packets
are properly discarded.
The effect of receiving a RETIRE_CONNECTION_ID frame is specified in the
next section.

Usually, it is expected that the PATH_ABANDON frame is used by the client
to indicate to the server that path conditions have changed such that
the path is or will be not usable anymore, e.g. in case of a mobility
event. The PATH_ABANDON frame therefore recommends to the receiver
that no packets should be sent on that path anymore.
In addition, the RETIRE_CONNECTION_ID frame is used indicate to the receiving peer
that the sender will not send any packets associated to the
Connection ID used on that path anymore.
The receiver of a PATH_ABANDON frame MAY also send
a PATH_ABANDON frame to indicate its own unwillingness to receive
any packet on this path anymore.

PATH_ABANDON frames can be sent on any path,
not only the path that is intended to be closed. Thus, a path can
be abandoned even if connectivity on that path is already broken.

If a PATH_ABANDON frame is received for the only active path of a QUIC
connection, the receiving peer SHOULD send a CONNECTION_CLOSE frame
and enter the closing state. If the client received a PATH_ABANDON
frame for the last open path, it MAY instead try to open a new path, if
available, and only initiate connection closure if path validation fails
or a CONNECTION_CLOSE frame is received from the server. Similarly
the server MAY wait for a short, limited time such as one PTO if a path
probing packet is received on a new path before sending the
CONNECTION_CLOSE frame.

### Refusing a New Path

An endpoint may deny the establishment of a new path initiated by its
peer during the address validation procedure. According to
{{QUIC-TRANSPORT}}, the standard way to deny the establishment of a path
is to not send a PATH_RESPONSE in response to the peer's PATH_CHALLENGE.

### Effect of RETIRE_CONNECTION_ID Frame {#retire-cid-close}

Receiving a RETIRE_CONNECTION_ID frame causes an endpoint to discard
the resources associated with that Connection ID. Specifically, the endpoint
should not use the sequence number of the retired Connection ID anymore in
any control frames, as the peer will not be able to associate those frames to
a path and will therefore ignore them. This means an endpoint is also not required
to acknowledge any late packets carrying that Connection ID and, hence,
it can remove the list of received packets used to send acknowledgements after
receiving the RETIRE_CONNECTION_ID frame.

The peer, that sent RETIRE_CONNECTION_ID frame, can keep sending data using
the same IP addresses and UDP ports previously associated with
that Connection ID, but MUST use a different connection ID when doing so.
If no new connection ID is available anymore, the endpoint cannot send on
this path. This can happen if, e.g., the Connection ID issuer requests retirement of a
Connection ID using the Retire Prior To field in the NEW_CONNECTION_ID frame but does
provide sufficient new CIDs.

Note that even if a peer cannot send on a path anymore because it does not have
a valid Connection ID to use, it can still acknowledge packets received on the path,
by sending ACK_MP frames on another path, if available. But also note that,
as there is no valid CID associated with the path, the other end cannot send
multipath control frames that contain the sequence number of a Connection ID, such
as PATH_ABANDON or PATH_STATUS.

If the peer cannot send on a path and no data is received on the path, the idle time-out will close
the path. If, before the idle timer expires, a new Connection ID gets issued
by its peer, the endpoint can re-activate the path by
sending a packet with a new Connection ID on that path.

If the sender retires a Connection ID that is still used by
in-flight packets, it may receive ACK_MP frames referencing the retired
Connection ID. If the sender stops tracking sent packets with retired
Connection ID, these would be spuriously marked as lost. To avoid such
performance issue without keeping retired Connection ID state, an
endpoint should first stop sending packets with the to-be-retired
Connection ID, then wait for all in-flight packets to be either
acknowledged or marked as lost, and finally retire the Connection ID.

### Idle Timeout {#idle-time-close}

{{QUIC-TRANSPORT}} allows for closing of connections if they stay idle
for too long. The connection idle timeout in multipath QUIC is defined
as "no packet received on any path for the duration of the idle timeout".
When only one path is available, servers MUST follow the specifications
in {{QUIC-TRANSPORT}}.

When more than one path is available, hosts shall monitor the arrival of
non-probing packets and the acknowledgements for the packets sent over each
path. Hosts SHOULD stop sending traffic on a path if for at least the period of the
idle timeout as specified in {{Section 10.1. of QUIC-TRANSPORT}}
(a) no non-probing packet was received or (b) no
non-probing packet sent over this path was acknowledged, but MAY ignore that
rule if it would disqualify all available paths. To avoid idle timeout of a path, endpoints
can send ack-eliciting packets such as packets containing PING frames
({{Section 19.2 of QUIC-TRANSPORT}}) on that path to keep it alive.  Sending
periodic PING frames also helps prevent middlebox timeout, as discussed in
{{Section 10.1.2 of QUIC-TRANSPORT}}.

Server MAY release the resource associated with paths for
which no non-probing packet was received for a sufficiently long
path-idle delay, but SHOULD only release resource for the last
available path if no traffic is received for the duration of the idle
timeout, as specified in {{Section 10.1 of QUIC-TRANSPORT}}.
This means if all paths remain idle for the idle timeout, the connection
is implicitly closed.

Server implementations need to select the sub-path idle timeout as a trade-
off between keeping resources, such as connection IDs, in use
for an excessive time or having to promptly reestablish a path
after a spurious estimate of path abandonment by the client.

## Path States

{{fig-path-states}} shows the states that an endpoint's path can have.

~~~
       o
       | PATH_CHALLENGE sent/received on new path
       v
 +------------+    Path validation abandoned
 | Validating |----------------------------------+
 +------------+                                  |
       |                                         |
       | PATH_RESPONSE received                  |
       |                                         |
       v                                         |
 +------------+     Path blackhole detected      |
 |   Active   |----------------------------------+
 +------------+                                  |
       |                                         |
       | PATH_ABANDON sent/received              |
       v                                         |
 +------------+                                  |
 |   Closing  |                                  |
 +------------+                                  |
       |                                         |
       | RETIRE_CONNECTION_ID sent && received   |
       | or                                      |
       | Path's draining timeout                 |
       | (at least 3 PTO)                        |
       v                                         |
 +------------+                                  |
 |   Closed   |<---------------------------------+
 +------------+
~~~
{: #fig-path-states title="States of a path"}

In non-final states, hosts have to track the following information.

- Associated 4-tuple: The tuple (source IP, source port, destination IP,
destination port) used by the endhost to send packets over the path.

- Associated Destination Connection ID: The Connection ID used to send
packets over the path.

In Active state, hosts MUST also track the following information:

- Associated Source Connection ID: The Connection ID used to receive
packets over the path. The endpoint relies on its sequence number to
send path control information and specifically acknowledge packets belonging to that Connection ID-specific
packet number space.

A path in the "Validating" state performs path validation as described
in {{Section 8.2 of QUIC-TRANSPORT}}. An endhost should not send
non-probing frames on a path in "Validating" state, as it has no
guarantee that packets will actually reach the peer.

The endhost can use all the paths in the "Active" state, provided
that the congestion control and flow control currently allow sending
of new data on a path. Note that if a path became idle due to a timeout,
endpoints SHOULD send PATH_ABANDON frame before closing the path.

In the "Closing" state, the endhost SHOULD NOT send packets on this
path anymore, as there is no guarantee that the peer can still map
the packets to the connection. The endhost SHOULD wait for
the acknowledgment of the PATH_ABANDON frame before moving the path
to the "Closed" state to ensure a graceful termination of the path.

When a path reaches the "Closed" state, the endhost releases all the
path's associated resources, including the associated Connection IDs.
Endpoints SHOULD send RETIRE_CONNECTION_ID frames for releasing the
associated Connection IDs following {{QUIC-TRANSPORT}}. Considering
endpoints are not expected to send packets on the current path in the "Closed"
state, endpoints can send RETIRE_CONNECTION_ID frames on other
available paths. Consequently, the endhost is not able to send nor
receive packets on this path anymore.

# Multipath Operation with Multiple Packet Number Spaces

The QUIC multipath extension uses different packet number spaces for each path.
This also means that the same packet number can occur on each path and the
packet number is not a unique identifier anymore. This requires changes to
the ACK frame as well as packet protection as described in the following subsections.

When multipath is negotiated,
each Destination Connection ID is linked to a separate packet number space.
Each CID-specific packet number space starts at packet number 0. When following
the packet number encoding algorithm described in {{Section A.2 of QUIC-TRANSPORT}},
the largest packet number (largest_acked) that has been acknowledged by the
peer in this new CID's packet number space is initially set to "None".

## Sending Acknowledgements

The ACK_MP frame, as specified in {{ack-mp-frame}}, is used to
acknowledge 1-RTT packets.
Compared to the QUIC version 1 ACK frame, the ACK_MP frame additionally
contains the receiver's Destination Connection ID Sequence Number field
to distinguish the Connection ID-specific packet number space.

Acknowledgements of Initial and Handshake packets MUST be carried using
ACK frames, as specified in {{QUIC-TRANSPORT}}. The ACK frames, as defined
in {{QUIC-TRANSPORT}}, do not carry the Destination Connection ID
Sequence Number field to identify the packet number space.
If the multipath extension has been successfully
negotiated, ACK frames in 1-RTT packets acknowledge packets sent with
the Connection ID having sequence number 0.

As soon as the negotiation of multipath support is completed,
endpoints SHOULD use ACK_MP frames instead of ACK frames to acknowledge application
data packets, including 0-RTT packets, using the initial Connection ID with
sequence number 0 after the handshake concluded.

ACK_MP frames (defined in {{ack-mp-frame}}) can be returned on any path.

## Packet Protection {#multipath-aead}

Packet protection for QUIC version 1 is specified in {{Section 5 of QUIC-TLS}}.
The general principles of packet protection are not changed for
QUIC Multipath. No changes are needed for setting packet protection keys,
initial secrets, header protection, use of 0-RTT keys, receiving
out-of-order protected packets, receiving protected packets,
or retry packet integrity. However, the use of multiple number spaces
for 1-RTT packets requires changes in AEAD usage.

{{Section 5.3 of QUIC-TLS}} specifies AEAD usage, and in particular
the use of a nonce, N, formed by combining the packet protection IV
with the packet number. When multiple packet number spaces are used,
the packet number alone would not guarantee the uniqueness of the nonce.

In order to guarantee the uniqueness of the nonce, the nonce N is
calculated by combining the packet protection IV with the packet number
and with the least significant 32 bits of the Destination Connection ID
sequence number.

{{Section 19 of QUIC-TRANSPORT}} encodes the Connection ID Sequence
Number as a variable-length integer,
allowing values up to 2^62-1; in this specification, a range of less than 2^32-1
values MUST be used before updating the packet protection key.

To calculate the nonce, a 96 bit path-and-packet-number is composed of the least
significant 32 bits of the Connection ID Sequence Number in network byte order,
two zero bits, and the 62 bits of the reconstructed QUIC packet number in
network byte order. If the IV is larger than 96 bits, the path-and-packet-number
is left-padded with zeros to the size of the IV. The exclusive OR of the padded
packet number and the IV forms the AEAD nonce.

For example, assuming the IV value is `6b26114b9cba2b63a9e8dd4f`,
the Connection ID Sequence Number is `3`, and the packet number is `aead`,
the nonce will be set to `6b2611489cba2b63a9e873e2`.

Due to the way the nonce is constructed, endpoints MUST NOT use more than 2^32
Connection IDs without a key update.

## Key Update {#multipath-key-update}

The Key Phase bit update process for QUIC version 1 is specified in
{{Section 6 of QUIC-TLS}}.
The general principles of key update are not changed in this
specification. Following QUIC version 1, the Key Phase bit is used to
indicate which packet protection keys are used to protect the packet.
The Key Phase bit is toggled to signal each subsequent key update.

Because of network delays, packets protected with the older key might
arrive later than the packets protected with the new key, however receivers
can solely rely on the Key Phase bit to determine the corresponding packet
protection key, assuming that there is sufficient interval between two
consecutive key updates ({{Section 6.5 of QUIC-TLS}}).

When this specification is used, endpoints SHOULD wait for at least three times
the largest PTO among all the paths before initiating a new key update
after receiving an acknowledgement that confirms receipt of the previous key
update. This interval is different from that of QUIC version 1 which used three times the PTO of the only one active path.

Following {{Section 5.4 of QUIC-TLS}}, the Key Phase bit is protected,
so sending multiple packets with Key Phase bit flipping at the same time
should not cause linkability issue.

# Examples

## Path Establishment

{{fig-example-new-path}} illustrates an example of new path establishment
using multiple packet number spaces.

~~~
   Client                                                  Server

   (Exchanges start on default path)
   1-RTT[]: NEW_CONNECTION_ID[C1, Seq=1] -->
                       <-- 1-RTT[]: NEW_CONNECTION_ID[S1, Seq=1]
                       <-- 1-RTT[]: NEW_CONNECTION_ID[S2, Seq=2]
   ...
   (starts new path)
   1-RTT[0]: DCID=S2, PATH_CHALLENGE[X] -->
                   Checks AEAD using nonce(CID sequence 2, PN 0)
     <-- 1-RTT[0]: DCID=C1, PATH_RESPONSE[X], PATH_CHALLENGE[Y],
                                              ACK_MP[PID=2,PN=0]
   Checks AEAD using nonce(CID sequence 1, PN 0)
   1-RTT[1]: DCID=S2, PATH_RESPONSE[Y],
             ACK_MP[PID=1, PN=0], ... -->

~~~
{: #fig-example-new-path title="Example of new path establishment"}

In {{fig-example-new-path}}, the endpoints first exchange
new available Connection IDs with the NEW_CONNECTION_ID frame.
In this example, the client provides one Connection ID (C1 with
sequence number 1), and server provides two Connection IDs
(S1 with sequence number 1, and S2 with sequence number 2).

Before the client opens a new path by sending a packet on that path
with a PATH_CHALLENGE frame, it has to check whether there is
an unused Connection IDs available for each side.
In this example, the client chooses the Connection ID S2
as the Destination Connection ID in the new path.

If the client has used all the allocated CID, it is supposed to retire
those that are not used anymore, and the server is supposed to provide
replacements, as specified in {{QUIC-TRANSPORT}}.
Usually, it is desired to provide one more Connection ID as currently
in use, to allow for new paths or migration.

## Path Closure

In this example, the client detects the network environment change
(client's 4G/Wi-Fi is turned off, Wi-Fi signal is fading to a threshold,
or the quality of RTT or loss rate is becoming worse) and wants to close
the initial path.

{{fig-example-path-close1}} illustrates an example of path closing. For the first path, the
server's 1-RTT packets use DCID C1, which has a sequence number of 1; the
client's 1-RTT packets use DCID S2, which has a sequence number of 2. For the
second path, the server's 1-RTT packets use DCID C2, which has a sequence
number of 2; the client's 1-RTT packets use DCID S3, which has a sequence number
of 3. Note that the paths use different packet number spaces. In this case, the
client is going to close the first path. It identifies the path by the sequence
number of the DCID its peer uses for sending packets over that path,
hence using the DCID sequence number 1 (which relates to C1). Optionally, the
server confirms the path closure
by sending an PATH_ABANDON frame by indicating the sequence number of the DCID
the client uses to send over that path, which corresponds to the sequence number
2 (of S2). Both the client and
the server can close the path after receiving the RETIRE_CONNECTION_ID frame
for that path.

~~~
Client                                                      Server

(client tells server to abandon a path)
1-RTT[X]: DCID=S2 PATH_ABANDON[dcid_seq_num=1]->
                           (server tells client to abandon a path)
               <-1-RTT[Y]: DCID=C1 PATH_ABANDON[dcid_seq_num=2],
                                               ACK_MP[PID=2, PN=X]
(client retires the corresponding CID)
1-RTT[U]: DCID=S3 RETIRE_CONNECTION_ID[2], ACK_MP[PID=1, PN=Y] ->
                            (server retires the corresponding CID)
 <- 1-RTT[V]: DCID=C2 RETIRE_CONNECTION_ID[1], ACK_MP[PID=3, PN=U]
~~~
{: #fig-example-path-close1 title="Example of closing a path."}

# Implementation Considerations

## Number Spaces

As stated in {{introduction}}, when multipath is negotiated, each
Destination Connection ID is linked to a separate packet number space.
This a big difference between implementations of QUIC as specified in
{{QUIC-TRANSPORT}}, which only have to manage three number spaces for Initial,
Handshake and Application packets.

Implementation of multipath capable QUIC will need to carefully
model the relations between paths and number spaces, as shown
in {{fig-number-spaces}}.

~~~
   +-------------------------+
   | CID received from peer: |
   | Previous Sender Number  |
   | Space                   |-- - - - - - +
   +-------------------------+
   +-------------------------+             |
   | CID received from peer: |
   | Sender Number Space     |             |
   +-------------------------+             v
                      ^             +----------------+
                      |             | Path (4 tuple) |
                      +-------------| - RTT          |
                +------------------>| - Congestion   |
                |                   |   state        |
                v                   +----------------+
   +-------------------------+             ^
   | CID provided to peer:   |             |
   | Receiver Number Space   |
   +-------------------------+             |
   +-------------------------+
   | CID previously used by  |-- - - - - - +
   | Peer: old Receiver      |
   | Number Space            |
   +-------------------------+
~~~
{: #fig-number-spaces title="Send and Receive number spaces"}

The path is defined by the 4-tuple through which packets are
received and sent. Packets sent on the path will include the
Destination Connection ID currently used for that path, selected
from the list of CID provided by the peer. Packets received
on the path carry a Destination CID selected by the peer from
the list provided to that peer.

The relation between CIDs and paths is not fixed. A node may
decide to rotate the Destination CID it uses, a NAT may decide
to change the 4-tuple over which packets from that path will be
received.
Implementation will have to manage these evolving relations.

Data associated with the transmission and reception on a given
path can be associated to either the "path state", or to the
state of either the sender or receiver number spaces. For example:

* RTT measurements and congestion state are logically associated
  with the 4-tuple. They will remain unchanged if data starts
  being received or sent through the same 4-tuple using new CIDs.

* Implementations of loss recovery typically maintain lists of
  packets sent and not yet acknowledged. Such information, along
  with the value of the next PN to use for sending, is
  logically associated with the "Sender Number Space", and
  with the peer-provided CID used for sending on the path.

* Sending of acknowledgement requires keeping track of the PN of
  received packets and of acknowledgements previously sent. Such
  information is logically associated with the "Receiver Number Space",
  and with the CID used by the peer for sending on the path.

When the link between paths and CID changes, the information tied
to the now unused CID remains useful for some time. For example,
the list of packet numbers to acknowledge maintained in the old
receiver number space could still be used to send ACK_MP frames
for that number space. Similarly, the list of packets sent but
not yet acknowledged with an old sender number space can be used
when processing incoming ACK_MP frames for that number space. Such
data should not be discarded immediately after a CID change, but
only later, for example when the CID is retired.

## Congestion Control {#congestion-control}

When the QUIC multipath extension is used, senders manage per-path
congestion status as required in {{Section 9.4 of QUIC-TRANSPORT}}.
However, in {{QUIC-TRANSPORT}} only one active path is assumed and as such
the requirement is to reset the congestion control status on path migration.
With the multipath extension, multiple paths can be used simultaneously,
therefore separate congestion control state is maintained for each path.
This means a sender is not allowed to send more data on a given path
than congestion control for that path indicates.

When a Multipath QUIC connection uses two or more paths, there is no
guarantee that these paths are fully disjoint. When two (or more paths)
share the same bottleneck, using a standard congestion control scheme
could result in an unfair distribution of the bandwidth with
the multipath connection getting more bandwidth than competing single
paths connections. Multipath TCP uses the LIA congestion control scheme
specified in {{RFC6356}} to solve this problem.  This scheme can
immediately be adapted to Multipath QUIC. Other coupled congestion
control schemes have been proposed for Multipath TCP such as {{OLIA}}.

## Computing Path RTT {#compute-rtt}

Acknowledgement delays are the sum of two one-way delays, the delay
on the packet sending path and the delay on the return path chosen
for the acknowledgements.  When different paths have different
characteristics, this can cause acknowledgement delays to vary
widely.  Consider for example a multipath transmission using both a
terrestrial path, with a latency of 50ms in each direction, and a
geostationary satellite path, with a latency of 300ms in both
directions.  The acknowledgement delay will depend on the combination
of paths used for the packet transmission and the ACK transmission,
as shown in {{fig-example-ack-delay}}.


ACK Path \ Data path         | Terrestrial   | Satellite
-----------------------------|-------------------|-----------------
Terrestrial | 100ms  | 350ms
Satellite   | 350ms  | 600ms
{: #fig-example-ack-delay title="Example of ACK delays using multiple paths"}

The ACK_MP frames describe packets that were sent on the specified path,
but they may be received through any available path. There is an
understandable concern that if successive acknowledgements are received
on different paths, the measured RTT samples will fluctuate widely,
and that might result in poor performance. While this may be a concern,
the actual behavior is complex.

The computed values reflect both the state of the network path and the
scheduling decisions by the sender of the ACK_MP frames. In the example
above, we may assume that the ACK_MP will be sent over the terrestrial
link, because that provides the best response time. In that case, the
computed RTT value for the satellite path will be about 350ms. This
lower than the 600ms that would be measured if the ACK_MP came over
the satellite channel, but it is still the right value for computing
for example the PTO timeout: if an ACK_MP is not received after more
than 350ms, either the data packet or its ACK_MP were probably lost.

The simplest implementation is to compute smoothedRTT and RTTvar per
{{Section 5.3 of QUIC-RECOVERY}} regardless of the path through which MP_ACKs are
received. This algorithm will provide good results,
except if the set of paths changes and the ACK_MP sender
revisits its sending preferences. This is not very
different from what happens on a single path if the routing changes.
The RTT, RTT variance and PTO estimates will rapidly converge to
reflect the new conditions.
There is however an exception: some congestion
control functions rely on estimates of the minimum RTT. It might be prudent
for nodes to remember the path over which the ACK MP that produced
the minimum RTT was received, and to restart the minimum RTT computation
if that path is abandoned.

## Packet Scheduling

The transmission of QUIC packets on a regular QUIC connection is regulated
by the arrival of data from the application and the congestion control
scheme. QUIC packets that increase the number of bytes in flight can only be sent
when the congestion window allows it.
Multipath QUIC implementations also need to include a packet scheduler
that decides, among the paths whose congestion window is open, the path
over which the next QUIC packet will be sent. Most frames, including
control frames (PATH_CHALLENGE and PATH_RESPONSE being the notable
exceptions), can be sent and received on any active path. The scheduling
is a local decision, based on the preferences of the application and the
implementation.

Note that this implies that an endpoint may send and receive ACK_MP
frames on a path different from the one that carried the acknowledged
packets. As noted in {{compute-rtt}} the values computed using
the standard algorithm reflect both the characteristics of the
path and the scheduling algorithm of ACK_MP frames. The estimates will converge
faster if the scheduling strategy is stable, but besides that
implementations can choose between multiple strategies such as sending
ACK_MP frames on the path they acknowledge packets, or sending
ACK_MP frames on the shortest path, which results in shorter control loops
and thus better performance.

## Retransmissions

Simultaneous use of multiple paths enables different
retransmission strategies to cope with losses such as:
a) retransmitting lost frames over the
same path, b) retransmitting lost frames on a different or
dedicated path, and c) duplicate lost frames on several paths (not
recommended for general purpose use due to the network
overhead). While this document does not preclude a specific
strategy, more detailed specification is out of scope.

## Handling different PMTU sizes

An implementation should take care to handle different PMTU sizes across
multiple paths. One simple option if the PMTUs are relatively similar is to apply the minimum PMTU of all paths to
each path. The benefit of such an approach is to simplify retransmission
processing as the content of lost packets initially sent on one path can be sent
on another path without further frame scheduling adaptations.

## Keep Alive

The QUIC specification defines an optional keep alive process, see {{Section 5.3 of QUIC-TRANSPORT}}.
Implementations of the multipath extension should map this keep alive process to a number of paths.
Some applications may wish to ensure that one path remains active, while others could prefer to have
two or more active paths during the connection lifetime. Different applications will likely require different strategies.
Once the implementation has decided which paths to keep alive, it can do so by sending Ping frames
on each of these paths before the idle timeout expires.

## Connection ID Changes and NAT Rebindings

{{Section 5.1.2 of QUIC-TRANSPORT}} indicates that an endpoint
can change the Connection ID it uses for to another available one
at any time during the connection. As such a sole change of the Connection
ID without any change in the address does not indicate a path change and
the endpoint can keep the same congestion control and RTT measurement state.

While endpoints assign a Connection ID to a specific sending 4-tuple,
networks events such as NAT rebinding may make the packet's receiver
observe a different 4-tuple. Servers observing a 4-tuple change will
performs path validation (see {{Section 9 of QUIC-TRANSPORT}}).
If path validation process succeeds, the endpoints set
the path's congestion controller and round-trip time
estimator according to {{Section 9.4 of QUIC-TRANSPORT}}.

{{Section 9.3 of QUIC-TRANSPORT}} allows an endpoint to skip validation of
a peer address if that address has been seen recently. However, when the
multipath extension is used and an endpoint has multiple addresses that
could lead to switching between different paths, it should rather maintain
multiple open paths instead.

If an endpoint uses a new Connection ID after an idle period
and a NAT rebinding leads to a 4-tuple changes on the same packet,
the receiving endpoint may not be able to associate the packet to
an existing path and will therefore consider this as a new path.
This leads to an inconsistent view of open paths at both peers,
however, as the "old" path will not work anymore, it will be silently
closed after the idle timeout expires.


# New Frames {#frames}

All the new frames MUST only be sent in 1-RTT packet.

If an endpoint receives a multipath-specific frame in a different packet type,
it MUST close the connection with an error of type FRAME_ENCODING_ERROR.

All multipath-specific frames relate to a Destination Connection
ID sequence number. If an endpoint receives a Destination Connection ID
sequence number greater than any previously sent to the peer, it MUST
treat this as a connection error of type MP_PROTOCOL_VIOLATION. If an
endpoint receives a multipath-specific frame
with a Destination Connection ID sequence number that it cannot process
anymore (e.g., because the Connection ID might have been retired), it
MUST silently ignore the frame.

## ACK_MP Frame {#ack-mp-frame}

The ACK_MP frame (types TBD-00 and TBD-01; experiments use 0xbaba00..0xbaba01)
is an extension of the ACK frame defined by {{QUIC-TRANSPORT}}. It is
used to acknowledge packets that were sent on different paths using
multiple packet number spaces. If the frame type is TBD-01, ACK_MP frames
also contain the sum of QUIC packets with associated ECN marks received
on the acknowledged packet number space up to this point.

ACK_MP frame is formatted as shown in {{fig-ack-mp-format}}.

~~~
  ACK_MP Frame {
    Type (i) = TBD-00..TBD-01
         (experiments use  0x15228c00-0x15228c01),
    Destination Connection ID Sequence Number (i),
    Largest Acknowledged (i),
    ACK Delay (i),
    ACK Range Count (i),
    First ACK Range (i),
    ACK Range (..) ...,
    [ECN Counts (..)],
  }
~~~
{: #fig-ack-mp-format title="ACK_MP Frame Format"}

Compared to the ACK frame specified in {{QUIC-TRANSPORT}}, the following
field is added.

Destination Connection ID Sequence Number:
: The sequence number of the Connection ID identifying the packet number
  space of the 0-RTT and 1-RTT packets which are acknowledged by the ACK_MP
  frame.

## PATH_ABANDON Frame {#path-abandon-frame}

The PATH_ABANDON frame informs the peer to abandon a path.

PATH_ABANDON frames are formatted as shown in {{fig-path-abandon-format}}.

~~~
  PATH_ABANDON Frame {
    Type (i) = TBD-02 (experiments use 0x15228c05),
    Destination Connection ID Sequence Number (i),
    Error Code (i),
    Reason Phrase Length (i),
    Reason Phrase (..),
  }
~~~
{: #fig-path-abandon-format title="PATH_ABANDON Frame Format"}

PATH_ABANDON frames contain the following fields:

Destination Connection ID Sequence Number:
: The sequence number of the Destination Connection ID used by the
  receiver of the frame to send packets over the path to abandon.

Error Code:
: A variable-length integer that indicates the reason for abandoning
  this path.

Reason Phrase Length:
: A variable-length integer specifying the length of the reason phrase
  in bytes. Because an PATH_ABANDON frame cannot be split between packets,
  any limits on packet size will also limit the space available for
  a reason phrase.

Reason Phrase:
: Additional diagnostic information for the closure. This can be
  zero length if the sender chooses not to give details beyond
  the Error Code value. This SHOULD be a UTF-8 encoded string {{!RFC3629}},
  though the frame does not carry information, such as language tags,
  that would aid comprehension by any entity other than the one
  that created the text.

PATH_ABANDON frames are ack-eliciting. If a packet containing
a PATH_ABANDON frame is considered lost, the peer SHOULD repeat it.

## PATH_STATUS frame {#path-status-frame}

PATH_STATUS Frame are used by endpoints to inform the peer of the current
status of one path, and the peer should send packets according to
the preference expressed in these frames.
PATH_STATUS frames are formatted as shown in {{fig-path-status-format}}.

~~~
  PATH_STATUS Frame {
    Type (i) = TBD-03 (experiments use 0x15228c06),
    Destination Connection ID Sequence Number (i),
    Path Status sequence number (i),
    Path Status (i),
  }
~~~
{: #fig-path-status-format title="PATH_STATUS Frame Format"}

PATH_STATUS Frames contain the following fields:

Destination Connection ID Sequence Number:
: The sequence number of the Destination Connection ID used by the
  receiver of this frame to send packets over the path the status update
  corresponds to.

Path Status sequence number:
: A variable-length integer specifying
  the sequence number assigned for this PATH_STATUS frame. The sequence
  number MUST be monotonically increasing generated by the sender of
  the PATH_STATUS frame in the same connection. The receiver of
  the PATH_STATUS frame needs to use and compare the sequence numbers
  separately for each Destination Connection ID Sequence
  Number.

Available values of Path Status field are:

- 1: Standby
- 2: Available

Endpoints use PATH_STATUS frame to inform the peer whether it prefer to
use this path or not. If an endpoint receives a PATH_STATUS frame
containing 1-Standby status, it SHOULD stop sending non-probing packets
on the corresponding path, until it receives a new PATH_STATUS frame
containing 2-Available status with a higher sequence number referring to
the same path.

Frames may be received out of order. A peer MUST ignore an incoming
PATH_STATUS frame if it previously received another PATH_STATUS frame
for the same Destination Connection ID Sequence Number with a
Path Status sequence number equal to or higher than the Path Status
sequence number of the incoming frame.

PATH_STATUS frames are ack-eliciting. If a packet containing a
PATH_STATUS frame is considered lost, the peer SHOULD resend the frame
only if it contains the last status sent for that path -- as indicated
by the sequence number.


# Error Codes {#error-codes}
Multipath QUIC transport error codes are 62-bit unsigned integers
following {{QUIC-TRANSPORT}}.

This section lists the defined multipath QUIC transport error codes
that can be used in a CONNECTION_CLOSE frame with a type of 0x1c.
These errors apply to the entire connection.

MP_PROTOCOL_VIOLATION (experiments use 0x1001d76d3ded42f3): An endpoint detected
an error with protocol compliance that was not covered by
more specific error codes.


# IANA Considerations

This document defines a new transport parameter for the negotiation of
enable multiple paths for QUIC, and three new frame types. The draft defines
provisional values for experiments, but we expect IANA to allocate
short values if the draft is approved.

The following entry in {{transport-parameters}} should be added to
the "QUIC Transport Parameters" registry under the "QUIC Protocol" heading.

Value                                         | Parameter Name.   | Specification
----------------------------------------------|-------------------|-----------------
TBD (current version uses 0x0f739bbc1b666d05) | enable_multipath  | {{nego}}
{: #transport-parameters title="Addition to QUIC Transport Parameters Entries"}


The following frame types defined in {{frame-types}} should be added to
the "QUIC Frame Types" registry under the "QUIC Protocol" heading.


Value                                              | Frame Name          | Specification
---------------------------------------------------|---------------------|-----------------
TBD-00 - TBD-01 (experiments use 0x15228c00-0x15228c01) | ACK_MP              | {{ack-mp-frame}}
TBD-02 (experiments use 0x15228c05)                  | PATH_ABANDON        | {{path-abandon-frame}}
TBD-03 (experiments use 0x15228c06)                  | PATH_STATUS         | {{path-status-frame}}
{: #frame-types title="Addition to QUIC Frame Types Entries"}

The following transport error code defined in {{tab-error-code}} should
be added to the "QUIC Transport Error Codes" registry under
the "QUIC Protocol" heading.

Value                       | Code                  | Description                   | Specification
----------------------------|-----------------------|-------------------------------|-------------------
TBD (experiments use 0x1001d76d3ded42f3)| MP_PROTOCOL_VIOLATION | Multipath protocol violation  | {{error-codes}}
{: #tab-error-code title="Error Code for Multipath QUIC"}


# Security Considerations

TBD

# Contributors

This document is a collaboration of authors that combines work from
three proposals. Further contributors that were also involved
one of the original proposals are:

* Qing An
* Zhenyu Li

# Acknowledgments

Thanks to Marten Seemann and Kazuho Oku for their thorough reviews and valuable contributions!
