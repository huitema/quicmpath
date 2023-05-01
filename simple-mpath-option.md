---
title: Simple QUIC Multipath Extension
abbrev: Simple QUIC Multipath
category: std
docName: draft-huitema-quic-mpath-option-02
    
stand_alone: yes

ipr: trust200902
area: Transport
kw: Internet-Draft

coding: us-ascii
pi: [toc, sortrefs, symrefs, comments]

author:
      -
        ins: C. Huitema
        name: Christian Huitema
        org: Private Octopus Inc.
        email: huitema@huitema.net

normative:
  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    seriesinfo:
      RFC: 9000
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

  QUIC-TLS:
    title: "Using TLS to Secure QUIC"
    date: {DATE}
    seriesinfo:
      RFC: 9001
    author:
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor
      -
        ins: S. Turner
        name: Sean Turner
        org: sn3rd
        role: editor


  QUIC-RECOVERY:
    title: "QUIC Loss Detection and Congestion Control"
    date: {DATE}
    seriesinfo:
      RFC: 9002
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: I. Swett
        name: Ian Swett
        org: Google
        role: editor

informative:

  QUIC-MP:
    title: "Multipath Extension for QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-multipath
    author:
      -
        ins: Y. Liu
        name: Yanmei Liu
        role: editor
        org: Alibaba Inc.
        email: miaoji.lym@alibaba-inc.com
      -
        ins: Y. Ma
        name: Yunfei Ma
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
    target: "https://datatracker.ietf.org/doc/draft-ietf-quic-multipath/"

  QUIC-DATAGRAM:
    title: "An Unreliable Datagram Extension to QUIC"
    date: {DATE}
    author:
      - 
        ins: T. Pauly
        name: Tommy Pauly
        org: Apple Inc. 
        email: tpauly@apple.com
      - 
        ins: E. Kinnear
        name: Eric Kinnear
        org: Apple Inc.
        email: ekinnear@apple.com
      - 
        ins: D. Schinazi
        name: David Schinazi
        org: Google LLC
        email: dschinazi.ietf@gmail.com
    target: "https://datatracker.ietf.org/doc/draft-ietf-quic-datagram/"

  QUIC-Timestamp:
    title: "QUIC Timestamps For Measuring One-Way Delays"
    author:
      - ins: C. Huitema
    date: 2020-8
    target: "https://datatracker.ietf.org/doc/draft-huitema-quic-ts/"

--- abstract

The initial version of QUIC provides support for path migration.
We propose a simple mechanism to support not just migration, but
also simultaneous usage of multiple paths. This mechanism
requires that multipath senders
keep track of which packet is sent on what path, and use that
information to manage congestion control and loss recovery. It works
better if multipath receivers tightly manage the formatting of
ACK frames. The mechanism does not require change to packet formats,
encryption methods, encryption key rotation, or the management of
connection identifiers. All software changes are beneficial to
both single path and multipath operation.

--- middle

# Introduction

The QUIC Working Group is debating how much priority should be given
to the support of multipath in QUIC. This is not a new debate. The
QUIC transport {{QUIC-TRANSPORT}} includes a number of mechanisms
to handle multiple paths, including the support for NAT rebinding
and for explicit migration.

The processing of received packets by QUIC nodes only requires
that the connection context can be retrieved using the combination
of IP addresses and UDP ports in the IP and UDP headers, and
connection identifier in the packet header. Once the connection context
is identified, the receiving node will verify if the packet can be
successfully decrypted. If it is, the packet will be processed and
eventually an acknowledgement will inform the sender that it was
received.

In theory, that property of QUIC implies that senders could send
packets using whatever IP addresses or UDP ports they see fit, as long
as they carry a valid connection ID and are
properly encrypted. Senders use the Path Challenge
and Response mechanism to verify that the path is valid before
sending packets, but even that is optional. The NAT rebinding mechanisms,
for example, rely on the possibility of receiving packets without
a preliminary Path Challenge. However, in practice, the sender's
choice of sending paths is limited by the path migration logic
in {{QUIC-TRANSPORT}}.

Path migration according to {{QUIC-TRANSPORT}} is always
initiated by the node in the client role. That node will test the validity
of paths using the path challenge response mechanism, and at some
point decide to switch all its traffic to a new path. The node in the
server role will detect that data traffic (i.e., non probing frames)
is sent on a new path, and on detecting that, will switch its own traffic
to that new path. After that, client and server can release the resource
tied to the old path, for example by retiring the connection identifiers,
releasing the memory context used for managing per path congestion,
or forgetting the IP addresses and ports associated with that path.
By sending data on a new path, the client provides an implicit signal to
discard the old path. If the client was to spread data on several paths,
the server would probably become confused.

This draft proposes a simple mechanism to enable transmission on
several paths simultaneously. This requires
updating the handling of paths specified in {{QUIC-TRANSPORT}} by
the proposed simple multipath management, see {{simple-multipath-specification}},
that multipath senders
keep track of which packet is sent on what path, that they use
this information to manage congestion control and loss recovery,
and that nodes take path information into account when sending
ACK_ECN frames. Simple multipath works better if receivers tightly manage
the formatting of ACK frames. 

This draft was initially developed as one of the alternatives that was
considered for QUIC Multipath Extension {{QUIC-MP}}, which evolved to
only support a more complex mechanism involving connection identifiers, number
spaces, path management, acknowledgements and explicit path management.
The QUIC working group can only adopt one solution, and it is extremely
unlikely that it would reverse course and adopt the Simple Multipath Extension
presented here instead of the current design. However, there is historic
and academic value in fully presenting the simple solution, such as for
example allowing comparison and informing future designs.

## Conventions and Definitions {#multipath-definition}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted
as described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all
capitals, as shown here.

# Simple Multipath Specification

The path management specified in section 9 of {{QUIC-TRANSPORT}} fulfills multiple goals:
it direct a peers to switch sending through a new preferred path, and it allows the peer
to release resources associated with the old path. Multipath requires several
changes to that mechanism:

* Allow simultaneous transmission of non probing frames on multiple paths.

* Continue using an existing path even if non probing frames have been received on
  another frame.

* Manage the removal of paths that have been abandoned.

* Use path information in congestion control and loss recovery.

* Change the handling of ACK-ECN frames to handle per path ECN

The multipath management is a departure from the specification of path management
in section 9 of {{QUIC-TRANSPORT}}. As such, it requires negotiation between
the two endpoints, as specified in {{enable-simple-multipath-extension}}. This negotiation will
also determine whether the endpoints will use simple or explicit multipath
management.

## Enable Simple Multipath Extension

The path management extension is negotiated by means of the 
"enable_simple_multipath" transport parameter:

* "enable_simple_multipath" (TBD)

The "enable_path_management" transport parameter is included if the endpoint
wants to receive or accepts  frames
for this connection. This parameter is encoded as a variable integer as specified in
section 16 of {{QUIC-TRANSPORT}}. It
can take one of the following values:

0. Default value, 0, implies that the simple multipath extension is not supported.

1. The simple path extension is supported, and is preferred over the official
   multipath support {{QUIC-MP}} if both options are available.

2. The simple path extension is supported, but the official
   multipath support {{QUIC-MP}} is preferred if both options are available.

Peers receiving another value SHOULD terminate the connection with a TRANSPORT
PARAMETER error.

The selected option depends on the proposed values, and also on whether the nodes
are capable of using official multipath support {{QUIC-MP}}:

* if both peers support the simple path extension and one of them does not
  enable the official multipath version, then the simple path extension
  is used.

* if both peers support the simple path extension and also enable the
  official multipath version, then the server preference take precedence.
  If the server selected the value 1, the simple multipath extension is
  used. If it selected the value is 2, the official multipath version is used.


## Path Creation and Path Validation

When the simple multipath option is negotiated, clients may initiate
transmission on multiple paths, using the mechanisms specified in
section 8.2 of {{QUIC-TRANSPORT}} to validate the new paths. After
receiving packets from the client on the new paths, the servers may in
turn attempt validate these paths using the same mechanisms.

Client may decide to send non-probing packets on validated paths. In
contrast with the specification in section 9 of {{QUIC-TRANSPORT}},
the server MUST NOT assume that receiving non-probing packets on a new path
indicates an attempt to migrate to that path. Instead, servers SHOULD
mark paths over which non-probing packets are received as available
for transmission.

## Path Removal

At any time in the connection, each endpoint manages a set of paths
that are available for transmission, as explained in
{{path-creation-and-path-validation}}. At any time, the client can
decide to abandon one of these paths, following for example changes in local
connectivity or changes in local preferences. After a client abandons
a path, the server will not receive any more non-probing packets
on that path.

When only one path is available, servers MUST follow the specifications in
{{QUIC-TRANSPORT}}. When more than one path is available,
servers shall monitor the arrival of non-probing packets on the
available paths. Servers SHOULD stop sending traffic on
paths through which no non-probing packet was received in the last 3 path RTTs,
but MAY ignore that rule if it would disqualify all available paths.
Server MAY release the resource associated with paths for which
no non-probing packet was received for a sufficiently long path-idle delay,
but SHOULD only release resource for the last available path if
no traffic is received for the duration of the idle timeout,
as specified in section 10.1 of {{QUIC-TRANSPORT}}.
Server implementations manage the value of the path-idle delay as a trade-off between
keeping resource such as Connection Identifiers in use for an excessive
time, and having to promptly reestablish a path after a spurious
estimate of path abandonment by the client.

## Packet Transmission

Once the connection is complete, if the simple multipath option is negotiated,
each endpoint can send 1RTT protected packets on any of the validated paths.
The packet sequence numbers are allocated from the common number space,
so that for example path number number N could be sent on one path and
packet number N+1 on another.

ACK frames report the packet numbers that have
been received so far, regardless of the path on which they have
been received. See {{acknowledgement-and-ranges}} for implementation recommendations on
the selection of packet ranges included in ACK frames. ACK_ECN frames
report these packet numbers as well, but see {{handling-of-ack-ecn-frames}}
for rules on reporting ECN numbers per path.

## Handling of ACK ECN Frames

When simple multipath extension is negotiated, the handling of ACK ECN
frames differs slightly from the specification in section 13.4 of
{{QUIC-TRANSPORT}}:

* implementations SHOULD count the numbers of received ECT(0),
ECT(1) and ECT-CE codepoints for each path, instead of for the whole
connection.
* the ACK ECN frames MUST carry the number of codepoints received on
the path over which the ACK_ECN frame is sent, instead of representing
the number received for all paths.

ECN validation is performed for each path, as specified in section
13.4.2 of {{QUIC-TRANSPORT}}. If validation fails, endpoints
that negotiate the simple multipath extension MUST
disable ECN reporting for that path. Similarly, endpoints that
negotiate the simple multipath extension and cannot
count ECN marks for a path MUST disable ECN reporting for that
path. Obviously, endpoints MAY send ACK frames.

Nodes may send ACK or ACK_ECN frames on any path. However, they SHOULD
ensure that ACK_ECN frames are regularly sent over each path on which
new ECN marks have been received.

## Per Path Information For Congestion Control

Senders MUST manage per-path congestion status, and MUST NOT send more
data on a given path than congestion control on that path allows. This
is already a requirement of {{QUIC-TRANSPORT}}.

In order to perform per-path congestion control, enpoints MUST maintain
per path information such as losses of packets, received ECN marks or
variations in measured round trip time.

In practice, maintaining information on losses and round trip times
requries that senders maintain
an association between previously sent packet numbers and the path
over which these packets were sent. When a packet is newly acknowledged,
the delay between the transmission of that packet and its first
acknowledgement is used to update the RTT statitics for the sending path,
and to update the state of the congestion control for that path.

## Path Information for Loss Recovery

The packet loss recovery algorithms specified in {{QUIC-RECOVERY}} include
two mechanisms: Acknowledgement Based Detection and Probe Timeout. Endpoints
that negotiate the simple multipath extension MUST adapt the loss recovery
implementation to account for transmission of packets on multiple paths:

* the round trip time estimation specified in section 5 of {{QUIC-RECOVERY}}
  MUST be performed per path, instead of globally for the whole connection.
  The values of min_rtt, smoothed_rtt and rttvar are computed independently
  for each path.

* per path RTT samples SHOULD be generated when an ACK frame newly acknowledges
  at least one packet sent on the path if that packet is ack-eliciting, even if
  the packet number is not the largest acknowledged inthe ACK frame. In that
  case, the acknowledgment delay SHOULD NOT be substracted from the RTT sample.

* when performing acknowledgement based detection, the packet number MUST be
  compared to packet numbers acknowledged on the same path, and the time
  threshold MUST be computed per path.

* the probe timeout algorithm SHOULD be executed per path, and MUST
  use PTO computed per path.

# Implementation Considerations

The guidance provided in this
section is based on early tests and early deployments.
We expect that as implementations and deployment progresses,
we will gain experience in the development and deployment of
the simple QUIC Multipath option. 

## Scheduling Transmissions

The Simple Multipath Specification in {{simple-multipath-specification}}
makes a number of recommendations such as only sending on paths
that are validated, or heeding per-path congestion control algorithms.
It stops short of mandating any specific scheduling algorithm. However,
based on early experience, we can make a set of implementations
recommendations.

### Tying Packet Scheduling and Loss Recovery

In principle, implementations that spread traffic on multiple paths
benefit from more bandwidth than using a single path, and are expected to
obtain better performance. However, early tests show that losses on one
path may cause delays until the lost data is retransmitted and finally
received. This can result in increased latency compared to transmission
on a single "non lossy" path.

In early tests, these effects were mitigated by tying packet scheduling
and loss recovery. If high losses were noticed on one path, the scheduling
algorithm would privilege transmission on alternate paths.

Of course, packet losses may be due to transient events. If losses are
experienced on a path, it is important to keep probing that path, and
to restart scheduling packets on that path if probing is successful. This
may be achieved by using redundant transmissions, see
{{redundant-transmissions}}.

### Multipath Scheduling and Slow-Start

Congestion control algorithms often rely on an initial phase such as
"slow-start" to assess the capacity of path, see for example
section 7.3 of {{QUIC-RECOVERY}}. The evaluation phase often ends
with the detection of packet losses. As discussed in
{{tying-packet-scheduling-and-loss-recovery}}, these losses
may have an adverse effect on the overall quality of experience.

In early tests, there were some benefits in using redundant transmissions
to avoid the adverse effects of packet losses during initial slow-start,
see {{redundant-transmissions}}.

### Redundant Transmissions

Redundant transmission is the process of sending the same data multiple
times. In early tests, this process was used to feed redundant packets to
paths that recently experienced losses, or to paths undergoing slow start.
Instead of probing the "dubious" path with dummy packets made of PING
and PAD frames, the implementation could send copies of frames previously
transmitted on other paths.

This mechanism is generally safe, because duplication of frames naturally
occurs as a side effect of "spurious" retransmissions. Implementations
of QUIC routinely filter out duplicate STREAM_DATA content, duplicate
stream and flow control management frames, or duplicate acknowledgments.
The datagram frames specified in {{QUIC-DATAGRAM}} are a notable
exception; including such frames in redundant packets could well result in
duplicate delivery.

Redundant transmission is a trade-off between risking delays due to losses
and risking delays due to inefficient use of the transmission capacity.
It is not always appropriate. One might also argue that duplicate
transmission is a simplistic form of forward error correction, and
that a more elaborate form of forward error correction might result
in better performance.

## Implementing Congestion Control and Loss Recovery

### Linking Packet Numbers and Paths

Per path congestion control and per path loss recovery defined in
{{per-path-information-for-congestion-control}} and
{{path-information-for-loss-recovery}} requires linking
packets and paths. One possible way is to remember three
variables for each packet awaiting retransmission:

* the global packet number,
* the local identifier of the path over which the packet was sent
* a "virtual sequence number" for the packet on the path.

The definition of these variables is illustrated in {{fig-path-virtual-number}}.

~~~
Path A      packet number       Path B

+-----+          +---+
| A,1 | <------- | 1 |
+-----+          +---+          +-----+
                 | 2 | -------> | B,1 |
+-----+          +---+          +-----+
| A,2 | <------- | 3 |
+-----+          +---+
| A,3 | <------- | 4 |
+-----+          +---+          +-----+
                 | 5 | -------> | B,2 |
+-----+          +---+          +-----+
| A,4 | <------- | 6 |
+-----+          +---+
| A,5 | <------- | 7 |
+-----+          +---+          +-----+
                 | 8 | -------> | B,3 |
                 +---+          +-----+
~~~
{: #fig-path-virtual-number title="Per path virtual packet number"}

The path identifier is used during the processing of acknowledgement to
retrieve the path over which the packet was sent, and thus update
the RTT or the congestion information for that path.

The virtual numbers can be directly used in the ACK based loss detection
algorithms: a packet can be deemed lost if later packet have been acknowledged
on the same path, and if the difference between the packet's virtual number is
larger than a threshold.

This data structure can be used even when the simple multipath option is
not negotiated. For example, it will automatically handle holes in the
packet number sequence caused by probes sent on other paths, or by
protections against optimistic ack attacks. 

### Acknowledgement and Ranges

If senders decide to send packets on paths with different transmission
delays, some packets will very probably be received out of order.
This will cause the ACK frames to carry multiple ranges of received
packets. The large number of range increases the size of ACK frames,
causing transmission and processing overhead.

Early tests showed that the size and overhead of the ACK frames could be
controlled by the combination of one or several of the following:

* Limiting the number of transmissions of a specific ACK range, on
the assumption that a sufficient number of transmissions almost
certainly ensures reception by the peer.

* Not transmitting again ACK ranges that were present in an ACK frame
acknowledged by the peer.

* Delay acknowledgement to allow for arrival of "hole filling" packets.

* Limit the total number of ranges sent in an ACK frame.

* Send multiple messages for a given path in a single socket operation,
so that a series of packets sent from a single path uses a series of
consecutive sequence numbers without creating holes.

This tight management of acknowledgement ranges is beneficial in both
multipath and single path operation. 

### Computing Path RTT

Acknowledgement delays are the sum of two one-way delays, the delay on the
packet sending path and the delay on the return path chosen for the
acknowledgements. When different paths have different characteristics,
there is an
understandable concern that if successive acknowledgements are received
on different paths, the measured RTT samples will fluctuate widely,
and that might result in poor performance. In fact, this concern is
probably not justified.

The computed values reflect both the state of the network path and the
scheduling decisions by the sender of the ACK or ACK_ECN frames. For
example, in a configuration mixing long delay satellite links with
a 300 ms transmission delay in each direction
and low bandwidth terrestrial links with 50 ms transmission delay,
we may assume that the ACK frames will be sent over the terrestrial
link, because that provides the best response time. In that case, the
computed RTT value for the satellite path will be about 350ms. This
lower than the 600ms that would be measured if the ACK came over
the satellite channel, but it is still the right value for computing
for example the PTO timeout: if an ACK_MP is not received after more
than 350ms, either the data packet or its ACK were probably lost.

In general, using the algorithm above will provide good results,
except if the set of path changes and the ACK sender
revisits its sending preferences. This is not very
different from what happens on a single path if the routing changes,
and the RTT, RTT variance and PTO estimates will rapidly converge to
the new values. There is however an exception: some congestion
control functions rely on estimates of the minimum RTT. It might be prudent
for nodes to remember the path over which the ACK MP that produced
the minimum RTT was received, and to restart the minimum RTT computation
if that path is abandoned.

# Comparison to the mainline QUIC Multipath draft

There were prolongated debates in the QUIC WG over whether to adopt a
single number space solution or a multiple number space solution. The
discussions considered 4 criteria, efficiency, code complexity,
handling of ACK, and support for zero-length CID, and the arguments were
summarized as follow:

* Efficiency:
    - Multiple PN space is more efficient, due to complete reuse of
      loss-recovery logic and no additional state
    - Single PN space is similarly efficient if loss detection is
      to manage a virtual order per path (see {{linking-packet-numbers-and-paths}})
* Code complexity:
    - If efficiency is required, Single PN space requries new code to
      to manage a virtual order per path
      (see {{linking-packet-numbers-and-paths}}) and manage ACK size
      (see {{acknowledgement-and-ranges}})
    - Multiple PN spaces require multiple instantiations of the loss recovery
      algorithm for each path
* ACK handling:
    - With single PN space,
      endpoints need to implement an algorithm to manage ACK sizes, (e.g.,
      {{linking-packet-numbers-and-paths}}), or risk seeing much larger
      overhead due to ACKs.
    - With multiple PN spaces, the new MP_ACK frame keeps small-sized ACK for
      each path; ACK-Delay and ACK-ECN work as expected
* Zero-length CID:
    - supported only with single PN space, not supported with multiple
      PN spaces.

Discussions also mentioned the lack of support for per path ECN in the
version of simple multipath presented at the time. With those arguments,
the working group concluded that the "multiple number space" solution was
a better choice.

A year later, several points surfaced that could well have changed this
evaluation.

## No change in encryption specification

The encryption specification in {{QUIC-TLS}} requries encrypting
packets using an AEAD algorithm. AEAD encryption requires a unique nonce
per packet, which is composed in {{QUIC-TLS}} by mixing an Initial Vector (IV)
with the 64 bit packet number. The simple multipath extension keeps
that specification unchanged, but this changes in {{QUIC-MP}},
because same packet number can be repeated in multiple spaces.In {{QUIC-MP}}
the nonce is obtained by
mixing the IV with the packet number and with an identifier of the
number space.

This requirement was easily met in the software implementations tested
a year ago, although we knew that it required changes in the API to the
cryptographic libraries, and that some implementations might find these changes
difficult to implement. In addition, we have since discovered that
this change makes hardware offload more complex, because the offload
engine have to keep track of the mapping between the connection ID in
the incoming packet and the identifier of the number space.

Hardware offload would probably be simpler with the simple multipath extension
presented here.

## No impact on key update

QUIC endpoints are supposed to update their encryption keys after a certain
number of packets have been sent. The update is managed is indicated by a phase
bit in the packet header. The algorithm uses the packet number to predict which
version of the encryption key is used for what packet. This is significantly
harder when using multiple number spaces as in {{QUIC-MP}}, because the key
rotation has to be coordinated across all these spaces. In contrast, the
simple multipath extension uses exactly the key mechanisms as specified in
{{QUIC-TLS}}, without requiring extra code.

## No impact on CID renewal

QUIC endpoints will renew the connection ID used on a given path on occasion,
in order to improve the privacy of transmissions. This is supported in both
the simple multipath extension and {{QUIC-MP}}, but there is an extra complexity
in {{QUIC-MP}}, as renewing the connection identifier also triggers the
start of a new packet number space. Endpoints need to manage that and somehow
tie the old and new packet number space together, to avoid a loss
of efficiency in loss recovery or in congestion control. This is extra complexity,
directly tied to the use of multiple number space.

## Support for zero length connection identifier

The simple multipath extension does not create any constraint on the use
of Connection Identifiers. There is no new frame type that would require
references to path identifiers on connection identifiers. This means
that the simple multipath extension fully supports the use of zero length
connection ID, which can significantly reduce the per packet overhead in
some scenarios. In contrast, the specification in {{QUIC-MP}}
requires the use of non-zero Connection IDs in both directions, which
will reduce efficiency in some scenarios.

## Summary of the comparison

One year later, it is pretty clear that the simple multipath extension is
much easier to implement and to manage than the use of multiple number
spaces in {{QUIC-MP}}. It does not require any new frame, does not change
encryption mechanisms, and does not constrain the use of connection identifiers.
Implementation aiming for efficiency will have to implement solutions similar to
those described in {{linking-packet-numbers-and-paths}} and
{{acknowledgement-and-ranges}}, but this extra software can be used
in both single-path and multipath modes, and provide efficiency gains in
both modes. Implementations that make the effort will find the efficiency
of the simple multipath extension very close to that of {{QUIC-MP}}.

Knowing what we now know, the working group may very well have made a different
decision a year ago. It is probably to late for that now.

# Security Considerations

The simple QUIC multipath extension adds very few mechanisms to {{QUIC-TRANSPORT}},
and thus expose very few new attacks. There is obviously an increased usage
of resource when managing multiple paths, which exposes to two new attacks:

* A node might try to increase the number of simulatneous paths in order to
  exhaust the peer's resource,
* On path attackers might try to modify the IP headers of QUIC packets in
  order to mimic NAT rebinding and try to increase the cost of path management.

Versions of these attacks are in fact already possible against {{QUIC-TRANSPORT}},
and generic mitigations apply, such as limiting the number of path that an endpoint
is willing to support.

# IANA Considerations

This document registers a new value in the QUIC Transport Parameter
Registry:


Value                        | Parameter Name.   | Specification
-----------------------------|-------------------|-----------------
TBD (experiments use 0x29e3d19e) | enable_simple_multipath  | {{enable-simple-multipath-extension}}
{: #transport-parameters title="Addition to QUIC Transport Parameters Entries"}

Note: the value 0x29e3d19e is picked as random, in accordance with
section 22.1.2 of {{QUIC-TRANSPORT}}. It is set to the first 32 bits
of the MD5 hash of "enable-simple-multipath-extension".

--- back

# Acknowledgements



