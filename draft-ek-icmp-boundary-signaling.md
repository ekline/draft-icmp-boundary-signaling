---
title: "ICMP Enclave Boundary Signaling"
abbrev: "ICMP Enclave Boundary Signaling"
category: exp

docname: draft-ek-icmp-boundary-signaling-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: INT
# workgroup: WG
keyword:
 - IP enclave
 - delay/distruption links
 - ICMP
 - ICMPv4
 - ICMPv6
venue:
#  group: INTAREA
#  type: Working Group
#  mail: intarea@ietf.org
#  arch: https://ietf.org/wg/intarea
  github: "ekline/draft-icmp-boundary-signaling"
  latest: "https://ekline.github.io/draft-icmp-boundary-signaling/draft-ek-icmp-boundary-signaling.html"

author:
 -
    fullname: "Erik Kline"
    organization: Aalyria Technologies, Inc.
    email: "ek.ietf@gmail.com"

normative:
  RFC0792:
  RFC4443:
  RFC4727:
  RFC4884:
  RFC5482:
  RFC5905:
  RFC9293:

informative:
  RFC1122:
  RFC1812:
  RFC4890:
  RFC4950:
  RFC5461:
  RFC5837:
  RFC5927:
  RFC8799:
  RFC8877:
  RFC8883:
  RFC9171:

...

--- abstract

This document defines two ICMP messages generated at the boundary
between a low-delay IP network and a scheduled, high-delay or
disruption-prone segment such as a deep-space link. The Path Delay
Notification reports that the path to a destination is awaiting a
scheduled contact, subject to a stated delay, or preempted, and
carries the expected delay and next contact time. The Denied by Link
Policy code, under Destination Unreachable, reports that the sender's
traffic is denied admission to the link the path requires. The
messages move information that already exists at the boundary, the
contact plan and the admission policy, to senders at local
timescales, allowing transports and applications to defer traffic,
fail fast, or hand traffic to a bundle service rather than discover
path conditions through timer expiry. The messages are specified for
use within administered limited domains, and this document is
published as Experimental.


--- middle

# Introduction {#intro}

In networks that include scheduled, high-delay segments -- deep-space
links, windowed satellite reachback, periodic store-and-forward
contacts -- an IP sender has no visibility into the condition of the
path beyond its own timers. The router at the boundary between the
sender's low-delay network and the high-delay segment knows a great
deal more: it holds a contact plan describing when the segment will
next carry traffic, and it enforces an admission policy describing
whose traffic the segment will carry. Today that knowledge shapes
forwarding at the boundary but is not communicated to senders, whose
transports retransmit into dead or unauthorized links until timers
expire, and whose applications receive an undifferentiated failure
indistinguishable from a routing fault or a crashed peer.

This document defines two ICMP messages that move boundary knowledge
to senders. The Path Delay Notification (PDN), a new ICMPv4 and
ICMPv6 type, reports that the path to a destination is awaiting a
scheduled contact, subject to a stated delay, or preempted, and
carries the expected delay and next contact time in an ICMP extension
object {{RFC4884}}. Disruption is expressed as the limiting case: a
path with no active contact is reported with a next-contact time and
no finite delay bound. The Denied by Link Policy code, a new code
under Destination Unreachable in both IP versions, reports that the
sender's traffic is denied admission to the link the path requires
under local policy. It is diagnostic, identifying the policy
dimension on which admission failed, and never prescriptive; it does
not describe what traffic would be admitted.

The mechanism rests on an asymmetry stated precisely in
{{applicability}}: the boundary is close and the path is far.
Feedback from the gateway reaches senders in local round-trip times,
milliseconds to seconds, while the conditions reported last minutes to
hours. Receivers use the messages to defer traffic until the next
contact, to fail fast when a path cannot meet an application's delay
tolerance, and to hand traffic to a Bundle Protocol {{RFC9171}} agent
for store-and-forward delivery. The messages complement rather than
compete with the Bundle Protocol; in many deployments the intended
effect of a PDN is precisely to trigger handoff to a bundle service at
flow establishment rather than after a timeout budget has been spent.
Neither message reports congestion, and neither substitutes for
congestion signaling or routing convergence ({{applicability}}).

These messages are specified for use within administered limited
domains {{RFC8799}}. This document is published as Experimental;
{{experiment}} defines the experiment, the questions it is intended to
answer, and the evidence that would support advancement.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Contact:
: an interval during which a high-delay link is planned to carry
  traffic between a gateway and its remote counterpart.

Contact plan:
: the schedule of contacts for a link or set of links, distributed to
  gateways in advance of execution.

High-delay segment:
: a network segment whose delay or scheduled unavailability is large
  relative to the round-trip times of the domains it connects.

Low-delay domain:
: an administered IP network whose internal round-trip times are
  negligible relative to the delays and gap durations of an adjacent
  high-delay segment. The definition is relative, not absolute
  ({{applicability}}).

Enclave:
: a low-delay domain reachable from other domains only across a
  high-delay segment.

Gateway:
: the router at the boundary between a low-delay domain and a
  high-delay segment, holding the contact plan and admission policy
  for that segment and generating the messages defined in this
  document.

Trans-boundary traffic:
: traffic whose path crosses a gateway onto a high-delay segment.

Admission:
: a gateway's determination, against local policy, that particular
  traffic may use a high-delay link.

Deferral cache:
: per-destination state a receiver maintains from PDN contents and
  consults before transmission ({{deferral-cache}}).


# Applicability {#applicability}

The mechanisms in this document are applicable at a boundary between
two network segments whose characteristic delays differ by enough that
feedback generated at the boundary reaches senders in time to change
their behavior. This section states that condition precisely, since it
both motivates the mechanism and bounds where deployment makes sense.

Let R be the round-trip time between a sender and the gateway
generating these messages, D be the duration of the condition being
reported (a contact gap, a deferral interval, or the path delay
itself), and T be the timescale on which the sender's transport would
otherwise detect failure on its own through retransmission timer growth
and eventual user timeout. The messages defined here are useful when
R is small relative to T, so the notification arrives while the sender
still has options, and R is small relative to D, so the reported
condition is still in effect when the notification arrives and the cost
of reacting to it is justified. In the Mars enclave of {{conops}}, R is
milliseconds to seconds, T is seconds to minutes, and D is minutes to
hours; both conditions hold by several orders of magnitude.

When R approaches D, the mechanism degrades: a notification that takes
as long to deliver as the condition it reports lasts describes a path
state that may no longer exist on arrival. This is why gateways
generate these messages only into their local low-delay domain and
never across the high-delay segment. Each boundary serves its own
side. An Earth-side gateway signals terrestrial senders; a Mars-side
gateway signals enclave hosts; neither attempts to signal across the
segment between them. The definition of "local" that follows from this
is relative, not absolute: the entire terrestrial internet, at
sub-second internal round-trip times, constitutes a single local
low-delay domain relative to a segment whose delays are measured in
minutes, and a vehicle-area network constitutes one relative to a
SATCOM link whose availability is measured in scheduled windows of tens
of minutes. No fixed delay threshold defines the boundary; the ratios
do.

The same criterion identifies deployments beyond the interplanetary
case. Tactical edge networks behind scheduled or intermittently
available reachback links, maritime and airborne platforms with
windowed satellite connectivity, sensor networks served by periodic
data-mule or store-and-forward contacts, and polar or remote ground
stations with pass-limited access all present a low-delay local domain
against a scheduled high-delay segment, and all have a boundary node
positioned to know the schedule and the admission policy. Conversely,
the mechanism offers nothing on paths that are merely lossy or
congested at uniform delay; existing congestion signaling and routing
convergence address those conditions, and these messages are not a
substitute for either.

These messages are specified for use within limited domains {{RFC8799}}
under a single administrative authority, for two reasons beyond the
delay criterion. First, ICMP carries no authentication, and the only
practical containment of forged notifications is validation against
sender state combined with filtering at domain boundaries; both are
enforceable within an administered domain and neither is dependable
across the open internet. Second, the Denied by Link Policy message
discloses admission policy to the sender it refuses. Within a
cooperative domain this is diagnostic information; toward an untrusted
sender it is reconnaissance, and gateways default to silent discard
rather than generating this message for traffic from outside the
domain. Boundary routers discard these ICMP types on ingress from and
egress to uncontrolled networks.


# Concept of Operations {#conops}

This section is informative. It describes the operational context in
which the messages defined in this document are generated and consumed,
using a representative deployment: an IP-based enclave network in Mars
orbit and on the Martian surface, connected to terrestrial networks
through scheduled deep-space contacts.

## Reference Scenario {#scenario}

The enclave consists of surface assets and orbiters interconnected by
conventional IP links: surface LANs, proximity RF links between landers
and relay orbiters, and inter-orbiter crosslinks. Within the enclave,
propagation delays are milliseconds to low seconds and connectivity is
continuous or near-continuous. Standard IP routing and transport
protocols operate normally.

The enclave connects to Earth through one or more gateway nodes on
relay orbiters equipped with deep-space links. These links have three
properties that motivate this document. First, they are scheduled:
contacts with Earth ground stations occur in windows determined by
orbital geometry and antenna allocation, known in advance through a
contact plan. Second, they are high-delay: one-way light time to Earth
ranges from roughly 3 to 22 minutes, so no interactive end-to-end IP
exchange is practical across them regardless of contact state. Third,
they are expensive and policy-managed: capacity during a contact may
be constrained and allocated to specific senders or traffic classes
according to local policy, and traffic outside those allocations is
not authorized to use the link.

The gateway nodes run a Bundle Protocol version 7 {{RFC9171}} agent and
act as the boundary between the enclave's IP domain and the
store-and-forward interplanetary segment. Trans-enclave data normally
crosses this boundary as bundles, either because the originating host
is BP-capable or because the gateway encapsulates IP traffic on the
host's behalf.

The essential asymmetry in this scenario is that the boundary is close
and the path is far. A host attempting to send toward Earth receives
feedback from the gateway within enclave round-trip times, on the order
of milliseconds to seconds, even though the path being reported on has
delays of minutes to hours. The messages defined here exploit that
asymmetry: they move knowledge that already exists at the boundary (the
contact plan, the authorization policy) to the hosts whose traffic is
affected by it, at local timescales.

The asymmetry is not specific to the Mars side. A terrestrial sender
directing traffic toward the enclave crosses an Earth-side gateway that
holds the same contact plan and enforces its own admission policy; that
gateway generates the same messages toward terrestrial senders at
terrestrial timescales. The mechanisms in this document apply
symmetrically at any boundary between a low-delay IP domain and a
scheduled, high-delay segment; this section describes the Mars-side
enclave for concreteness.

## Roles {#roles}

Enclave hosts originate and receive application traffic. Some are
DTN-aware, running a BP agent or a stack extension that can hand
traffic to one. Others are unmodified IP hosts, including legacy
instruments and COTS payloads whose stacks predate this specification.

Gateway nodes hold the contact plan for their deep-space links,
enforce per-class authorization and precedence policy on contact
capacity, and generate the messages defined in this document. A
gateway generates a PDN when it receives IP traffic for a trans-enclave
destination while no contact is active, or while the active contact
cannot meet the delay characteristics implied by the traffic. It
generates a Denied by Link Policy message when traffic arrives that
is not admitted under the link's policy, independent of contact
state.

Implicit in this architecture, and load-bearing for the semantics of
both messages, is that generation occurs only when no alternative
trans-enclave path could serve the traffic. Enclave routing, driven by
the same contact plan, already steers trans-enclave traffic to
whichever gateway and Direct-With-Earth link can serve it; if an
alternate DWE contact were available, the traffic would have been
routed there and no message generated. A PDN or a Denied by Link
Policy message therefore asserts a condition of the enclave's
aggregate connectivity toward the destination, not of one interface,
and the correct receiver response is deferral, handoff, or escalation
to provisioning as described in {{use-cases}}, never a routing retry.
A message generated while another gateway did have service for the
traffic indicates that routing and the contact plan have diverged,
which is the fault condition of {{uc-divergence}}, not a situation the
sender can resolve.

An orchestration or network management function, which may be
Earth-based, distributes contact plans and authorization policy to
gateways in advance. Because plan distribution itself crosses the
deep-space link, gateways are provisioned to operate autonomously
between management contacts; the messages they generate reflect their
current local plan and policy, which is authoritative for their links.

Intra-enclave communication is unaffected by any of this. Hosts
exchanging traffic entirely within the enclave use standard IP,
routing, and transports unmodified; the messages defined here are
generated only for traffic that requires a trans-enclave link, and
hosts that never originate such traffic need not implement or be aware
of them.

Optionally, a performance-enhancing proxy (PEP) or enclave-side
application gateway terminates transport sessions on behalf of legacy
hosts. Where present, the PEP is the consumer of these messages for
the hosts behind it.

## Use Cases {#use-cases}

### Deferral Until the Next Contact {#uc-deferral}

A surface host begins a bulk transfer toward an Earth destination
between contacts. The first packets reach the gateway, which has no
active contact and generates a PDN carrying the start time and duration
of the next contact from its plan. The host's stack records the
notification against the destination prefix, analogous to a path MTU
cache entry, and suppresses further transmission until the indicated
time. The application is informed through the transport API that the
path is deferred rather than failed. Without the notification, the
host would retransmit into a dead link until its transport gave up, and
the application would see an undifferentiated timeout indistinguishable
from a routing failure or a crashed peer.

The cache entry has a natural lifetime: the next-contact time itself.
Negative caching driven by the contact plan means one notification per
destination prefix per gap is sufficient, which matters because the
gateway rate-limits ICMP generation like any router.

### Delay Tolerance Decisions by Transports and Applications {#uc-tolerance}

A PDN carries the expected path delay whether or not a contact is
active; during a contact the deep-space path is up but still has
minutes of one-way delay. A transport or application compares the
advertised delay against its own tolerance. A telemetry archiver with
a delivery deadline of hours proceeds. A request/response exchange
with a ten-second application timeout learns immediately that the
exchange cannot succeed over this path and fails fast, or switches to a
messaging pattern suited to the delay. The quantitative field, rather
than a bare unreachable indication, is what allows this decision to be
made by the endpoint instead of being guessed at through timer
expiration.

### Handoff to the Bundle Service {#uc-handoff}

A DTN-aware host receiving a PDN for a destination hands the affected
traffic to its local BP agent, which originates bundles toward the
destination's EID; the bundles are queued at the gateway and forwarded
during the next contact. Where the PDN's extension object identifies a
BP gateway, a host without a local agent can instead direct traffic to
that gateway for encapsulation. In either case the application's data
survives the disruption rather than being retransmitted against it.
This handoff is the intended terminal state for most trans-enclave
traffic; the PDN's role is to trigger it at flow establishment rather
than after a timeout budget has been spent.

### Traffic Denied by Link Policy {#uc-denied}

A payload host, misconfigured or newly integrated, sends
best-effort-marked traffic toward Earth during an active contact whose
capacity is allocated to command and telemetry classes. The gateway
classifies the traffic against its local policy, which may consider
DSCP, source and destination address, transport protocol and port, or
any other classification criteria the policy defines, and determines
that the traffic is not admitted. It generates a Denied by Link
Policy message with a reason identifying the policy dimension that
failed. The message is diagnostic, not prescriptive: it carries no
description of what traffic would be admitted. Admission policy is
multidimensional and local, no compact wire encoding of it exists, and
a message that coached senders on markings the gateway would accept
would invert the admission model, in which access is granted through
the control plane rather than obtained by senders adapting traffic to
gateway responses. The host, or the operator debugging it, learns why
the traffic was refused rather than merely that it was; obtaining
admission is a provisioning action. A legacy host that does not
implement this specification processes the message as a generic
Destination Unreachable and aborts, which is the correct outcome for
unauthorized traffic.

### Preemption During a Contact {#uc-preemption}

A flow authorized at routine precedence is admitted at contact start
and later displaced by higher-precedence traffic. This condition is
temporal, not an authorization failure: the sender's class remains
permitted and service can resume within the same contact. The gateway
therefore reports it with a PDN carrying an expected resumption time
where one can be estimated, not with a Denied by Link Policy message.
Reporting preemption as prohibition would drive senders to abandon
paths they should merely pause on.

### Proxy Operation for Legacy Hosts {#uc-proxy}

Where a PEP fronts legacy instruments, the PEP terminates their
transport sessions locally, consumes PDN and Denied by Link Policy
messages itself, and applies the deferral and handoff behaviors above
on the instruments' behalf. The instruments see locally-acknowledged
transport sessions with intra-enclave-scale RTTs and no visibility into
contact structure. This is the expected deployment pattern for hosts
that cannot be modified, and it means the specification's benefit does
not depend on universal host adoption within the enclave.

### Plan/Reality Divergence as an Operational Signal {#uc-divergence}

The orchestration function provisions flows against the contact plan;
in nominal operation, authorized traffic should not encounter these
messages at all. A PDN generated during a planned contact, or a
Denied by Link Policy message generated for a flow the orchestrator
believes it admitted, indicates divergence between the control plane's
model and the data plane's state: an unplanned outage, a policy
distribution failure, or traffic originating outside the provisioning
workflow. Gateways count and report generation of these messages
through network management, and enclave telemetry consumers treat them
as a diagnostic feed independent of their effect on senders.

## Receiver Processing Model {#receiver-model}

A conforming host processes these messages at three levels. The stack
maintains a per-destination (or per-prefix) deferral cache populated
from PDN extension data, consulted before transmission, and expired at
the indicated contact time. The transport maps a PDN to a soft or
hard error depending on the deferral duration. If the indicated
resumption time falls within the connection's user timeout ({{RFC9293}},
{{RFC5482}} where negotiated) or an application-configured tolerance,
TCP treats the PDN as a soft error, suspending retransmission for the
deferral interval rather than counting it toward connection failure.
If the resumption time exceeds that tolerance, or the PDN carries no
resumption time, TCP aborts the connection immediately: the connection
was going to fail at timer expiry regardless, and delivering the
failure now, with the reason attached, is the point of the mechanism.
A PDN received in SYN-SENT is treated as a hard error unless the
application has indicated delay tolerance for the connection, since no
transfer state is preserved by waiting and the application can elect a
bundle-service path instead. Note that the peer receives no
corresponding notification and runs its own timers; suspending a
connection locally for longer than the peer will tolerate preserves
nothing. QUIC surfaces the PDN as a path event and applies the same
tolerance comparison per connection. The API surfaces both messages
to applications as distinguishable conditions, so an application can
implement its own deferral or handoff logic where the stack defaults
are insufficient. A Denied by Link Policy message is surfaced as a
hard error carrying the structured reason; the connection is not
retried unchanged.

Validation follows {{RFC4443}} and {{RFC5927}}: the receiver matches
the embedded packet against existing state and discards notifications
that do not correspond to traffic it sent. Within the enclave this
check, combined with boundary filtering that discards these ICMP types
arriving from outside the domain, bounds exposure to off-path
injection. These messages are specified for use within a limited
domain {{RFC8799}}; gateways do not generate them toward the
deep-space link, and enclave boundaries do not admit them from it.

## Interaction of the Two Messages {#interaction}

The two messages answer different questions and may both apply to one
flow: a link can be simultaneously expensive and down. Authorization
is evaluated first. Traffic that would be refused regardless of
contact state receives a Denied by Link Policy message; deferring
such traffic to the next contact would only result in it being
refused again. Traffic that is authorized but cannot currently be
served receives a PDN. A sender whose denial is resolved through
provisioning may subsequently receive a PDN for the same flow, which
is the expected sequence: first become admissible, then wait for
service.


# Message Formats and Extension Objects {#formats}

This section is normative. Type, code, and object class values are
shown as TBD pending IANA assignment ({{iana}}); prior to assignment,
implementations use the experimental values of {{RFC4727}} as
described in {{exp-scope}}.

## Path Delay Notification {#pdn}

The PDN is a new ICMPv6 error message, Type TBD1. The ICMPv4
equivalent is a new type, TBD2, with the same layout except as noted
in {{pdn-v4}}. The message reuses the extended-message structure of
{{RFC4884}} so that existing extension-aware parsers process it
without modification.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |            Checksum           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Length     |                   Reserved                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
~   As much of the invoking packet as possible, zero-padded    ~
|   to a multiple of 8 octets, per Section 4 of RFC 4884       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
~              ICMP Extension Structure (RFC 4884)              ~
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-pdn title="Path Delay Notification Message Format"}

The Length field gives the size of the invoking packet portion in
8-octet units, per {{RFC4884}}. The Reserved field MUST be set to zero
on transmission and ignored on receipt.

The Code field distinguishes the condition being reported:

| Code | Name |
|-----:|:-----|
| 0 | No contact active; path awaiting next contact |
| 1 | Contact active; path subject to the indicated delay |
| 2 | Contact active; sender's traffic preempted |
{: #tab-pdn-codes title="Path Delay Notification Codes"}

Code 2 reports the preemption condition of {{uc-preemption}}: the
sender's traffic remains admitted but is displaced within the current
contact. It is distinguished from code 0 because the reported times
refer to expected resumption within a contact rather than to the
contact plan, and receivers MAY apply a shorter deferral-cache
lifetime to it.

A PDN MUST include exactly one Path Schedule Object ({{pso}}) in its
extension structure. A receiver processing a PDN without one MUST
treat all schedule fields as unknown. Generation of PDNs is
rate-limited per {{Section 2.4 of RFC4443}}; the deferral cache of
{{receiver-model}} is what makes one notification per destination
prefix per gap sufficient.

### ICMPv4 Differences {#pdn-v4}

The ICMPv4 PDN (Type TBD2) carries the Length field in 32-bit words
and follows the invoking-packet inclusion and padding rules of
{{Section 5.1 of RFC4884}}. All other fields and all extension objects
are identical.

## Denied by Link Policy {#dlp}

Denied by Link Policy is code TBD3 under ICMPv6 Destination
Unreachable (Type 1) and code TBD4 under ICMPv4 Destination
Unreachable (Type 3). The message layout is that of the extended
Destination Unreachable message defined by {{RFC4884}}; this document
adds no fields to it.

{{RFC4884}} updates {{RFC0792}} and {{RFC4443}} to permit an ICMP
Extension Structure on Destination Unreachable messages of both
versions, and its requirements apply here by inheritance: the message
carries the Length field of {{RFC4884}} bounding the original-datagram
portion, that portion is truncated and zero-padded per Sections 4.4
and 5.1 of {{RFC4884}} (including the 128-octet original-datagram
length for ICMPv4), and the extension structure follows it. New
object classes under this mechanism are an established pattern;
{{RFC8883}} defines a Destination Unreachable code carrying ancillary
information in an extension object in the same way, and {{RFC5837}}
and {{RFC4950}} attach objects to Destination Unreachable in wide
deployment.

A receiver that predates this specification treats the message as a
generic Destination Unreachable: it does not recognize the code, and
the extension structure is invisible to it, appearing only as
trailing octets beyond the portion of the original datagram such
receivers examine. The classic-versus-compliant message ambiguity
discussed in {{Section 5.3 of RFC4884}} cannot arise for this code,
because any receiver that recognizes the code postdates this
specification and parses the extension structure unconditionally.

A Denied by Link Policy message SHOULD include exactly one Admission
Denial Object ({{ado}}); a message without one reports the denial with
no reason, which a gateway MAY choose deliberately where policy
forbids disclosure ({{applicability}}).

## Path Schedule Object {#pso}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Length = 28           |  Class-Num =  |  C-Type = 1   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Generation Time                          +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                     Next Contact Time                         +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Expected Delay                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Contact Duration                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-pso title="Path Schedule Object"}

The object header is as defined in {{Section 8 of RFC4884}}; the class
is the Path Signaling Object class (Class-Num TBD5), C-Type 1.

Generation Time and Next Contact Time are 64-bit NTP timestamps as
defined in {{Section 6 of RFC5905}} and recommended by {{RFC8877}}: 32
bits of seconds and 32 bits of fraction, in the UTC timescale of the
NTP era containing the time of generation, seconds excluding leap
second adjustments. A gateway whose contact plan is expressed in
another timescale (TAI, GPS) MUST convert at generation; no timescale
indicator is carried. The value 0 indicates an unknown or unset
timestamp, consistent with NTP convention.

Generation Time MUST be set to the gateway's current time when the
message is generated. Next Contact Time carries the start of the
next contact serving the destination (code 0), the present time
(code 1), or the expected resumption time (code 2); it is 0 where the
gateway has no prediction.

Per the timestamp specification template of {{RFC8877}}: the epoch is
the NTP epoch, 1 January 1900 UTC; resolution is 2^-32 seconds;
the format wraps every 2^32 seconds (one NTP era, approximately 136
years). No era number is carried. Both timestamps in one object MUST
be generated from the same clock, and receivers resolve the era of
each absolute timestamp as the era placing it nearest the receiver's
current time. A receiver that does not trust its own synchronization
MAY instead use the two timestamps differentially, treating (Next
Contact Time - Generation Time), computed with unsigned modular
arithmetic, as an offset from message receipt; the error of this
interpretation is bounded by the in-domain transit time of the
message, which the applicability criterion of {{applicability}} makes
negligible against the durations reported.

Expected Delay is the anticipated one-way path delay to the
destination, in milliseconds, as an unsigned 32-bit integer. The
value 0xFFFFFFFF indicates no finite bound, which is how the
disrupted case is expressed; the value 0 indicates unknown.

Contact Duration is the expected remaining duration of the contact
identified by Next Contact Time, in milliseconds. The value 0
indicates unknown.

A receiver MUST ignore Path Signaling objects with an unrecognized
C-Type and MUST process at most one Path Schedule Object per message,
taking the first if several are present.

## Admission Denial Object {#ado}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Length = 8           |  Class-Num =  |  C-Type = 2   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Reason             |            Reserved           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-ado title="Admission Denial Object"}

The object uses the Path Signaling Object class (Class-Num TBD5),
C-Type 2. Reserved MUST be zero on transmission and ignored on
receipt.

Reason identifies the policy dimension on which admission failed,
from a new IANA registry with the following initial values:

| Value | Name |
|------:|:-----|
| 0 | Reserved |
| 1 | Denied; dimension not disclosed |
| 2 | Traffic classification not admitted |
| 3 | Source not admitted |
| 4 | Destination not admitted |
| 5 | Allocation exhausted for admitted classification |
| 6 | Precedence below admission threshold |
| 7-65279 | Unassigned |
| 65280-65534 | Experimental use |
| 65535 | Reserved |
{: #tab-ado-reasons title="Admission Denial Reason Codes"}

Consistent with {{uc-denied}}, reasons identify the failed dimension
and never describe what would be admitted. Value 1 exists so that a
gateway can generate the message while disclosing nothing, as an
alternative to the silent discard of {{applicability}} where the
operator wants the sender to receive a definite denial. Registration
policy for new values is Specification Required.


# Generation Rules {#generation}

The rules in this section govern gateways. General ICMP generation
rules apply in addition: {{Section 2.4 of RFC4443}} for ICMPv6,
including its rate-limiting requirements, and {{RFC1812}} for ICMPv4.
Nothing in this document permits generating an error in response to
an ICMP error, to a packet addressed to a multicast destination, or
in the other cases those documents prohibit.

A gateway MUST NOT generate a PDN or a Denied by Link Policy message
for traffic that another path in the domain could serve. Domain
routing, driven by the same contact plan, is assumed to have steered
the traffic to the generating gateway because no alternative exists;
these messages assert a condition of the domain's aggregate
connectivity toward the destination ({{roles}}). A gateway that
cannot make this determination for a destination MUST NOT generate
these messages for it.

For trans-boundary traffic a gateway cannot currently serve,
admission is evaluated first ({{interaction}}). Traffic that is denied
admission regardless of contact state is answered with a Denied by
Link Policy message. Traffic that is admitted but cannot currently
be served is answered with a PDN: code 0 when no contact is active,
code 1 when a contact is active and the stated path delay applies,
and code 2 when the sender's traffic is preempted within an active
contact.

A gateway generating a PDN MUST set Generation Time to its current
time and MUST populate the remaining Path Schedule Object fields from
its contact plan where predictions exist, using the unknown values of
{{pso}} where they do not. A gateway MUST NOT populate Next Contact
Time with a value not derived from its current plan.

Generation of both messages MUST be configurable per policy. In
particular, an operator MUST be able to configure, per source,
prefix, or policy class: whether a Denied by Link Policy message is
generated at all, the alternative being silent discard, and whether
its Admission Denial Object carries a specific reason or the value 1,
dimension not disclosed. An operator SHOULD be able to configure
omission of Next Contact Time and Contact Duration, transmitted as
unknown, where the contact schedule itself is sensitive ({{security}}).

A gateway MUST NOT generate these messages toward the high-delay
segment and MUST NOT generate them in response to traffic arriving
from it. Each gateway signals only into its local low-delay domain
({{applicability}}).

These messages report conditions known at the boundary; their absence
promises nothing. A gateway restarting without a current contact
plan does not generate PDNs until it holds one.


# Receiver Requirements {#receiver}

## Validation {#validation}

A receiver MUST validate these messages as ICMP errors per
{{RFC4443}} and the mitigations of {{RFC5927}}: the invoking packet
excerpt is matched against existing connection or flow state, and a
message matching nothing the receiver sent is discarded. A receiver
SHOULD discard these messages when received on an interface facing
outside its administered domain.

## Deferral Cache {#deferral-cache}

A host implementing PDN processing SHOULD maintain a deferral cache
keyed by destination address or covering prefix. A PDN inserts or
replaces the entry for the invoking packet's destination; a newer PDN
always replaces an older entry for the same key. An entry records
the code, the schedule fields, and an expiry.

An entry created from a code 0 PDN expires at Next Contact Time; from
code 2, at the expected resumption time; from code 1, after an
implementation-chosen interval, since an active high-delay contact
offers no gap to key expiry to. An entry whose PDN carried no usable
time expires after an implementation-chosen conservative interval.
In all cases entry lifetime MUST be bounded by a configurable
maximum; a default of 24 hours is RECOMMENDED, and deployments whose
planned gaps exceed it, such as solar conjunction, configure it
accordingly. The bound limits the effect of a forged PDN carrying a
far-future contact time ({{security}}).

Entries are advisory. A gateway cannot revoke a PDN whose plan has
since changed, so an entry is a hint about the plan as of generation,
not a guarantee in either direction. A host MAY probe before expiry,
SHOULD NOT sustain transmission against an unexpired entry, and MUST
NOT treat expiry as an assurance of service; traffic sent at expiry
that finds the path still unserved elicits a fresh PDN.

## Transport Processing {#transport}

A transport maps a PDN to a soft or hard error by comparing the
entry's expiry against the connection's tolerance. For TCP
{{RFC9293}}, the tolerance is the user timeout, as adjusted by
{{RFC5482}} where negotiated, or an application-supplied value. If the
expiry falls within the tolerance, the PDN is a soft error: TCP
SHOULD suspend retransmission until the expiry rather than counting
the interval toward connection failure. If the expiry exceeds the
tolerance, or none could be computed, TCP SHOULD abort the connection
and surface the reason to the application. A PDN received in
SYN-SENT SHOULD be treated as a hard error unless the application has
indicated delay tolerance for the connection. The remote endpoint
receives no corresponding notification and continues to run its own
timers; a receiver MUST NOT suspend a connection beyond what it knows
or has negotiated of the peer's tolerance.

Other transports apply the same comparison; QUIC surfaces the PDN as
a path event on the connection.

A Denied by Link Policy message MUST be surfaced as a hard error for
matching connections and flows. A receiver SHOULD NOT automatically
retry the traffic unchanged; obtaining admission is a provisioning
action ({{uc-denied}}).

## Application Interface {#api}

Hosts SHOULD surface both messages to applications as conditions
distinguishable from each other and from other unreachable errors,
including the schedule fields of a PDN and the reason of a Denied by
Link Policy message, so that applications can implement their own
deferral or handoff logic where transport defaults are insufficient.


# Legacy Host and Middlebox Behavior {#legacy}

This section analyzes hosts and middleboxes that predate this
specification. The two messages were deliberately given opposite
legacy failure modes, and the design claim of this section is that
every plausible legacy behavior is acceptable and none is worse than
the status quo, in which senders learn nothing and time out.

## Path Delay Notification {#legacy-pdn}

The PDN is a new type, so legacy behavior is whatever a stack does
with an unknown ICMP type, and the two IP versions specify this
differently. For ICMPv4, {{Section 3.2.2 of RFC1122}} requires that a
message of unknown type be silently discarded; the PDN is a specified
no-op. For ICMPv6, {{Section 2.4 of RFC4443}} requires that an
unknown message with the error bit set be passed to the upper-layer
process that originated the invoking packet. The PDN is therefore not
discarded at the ICMPv6 layer but delivered to a transport with no
defined handling for it. Deployed transports ignore ICMPv6 error
types they do not classify, so the practical outcome is a no-op as
well, but that is an empirical claim about implementations rather
than a specified behavior, and {{exp-questions}} lists confirming it
across the stacks present in target environments, including embedded
and flight-heritage stacks, as an experiment question.

In either version, a legacy host experiences exactly today's
behavior: it retransmits into the gap and times out.

## Denied by Link Policy {#legacy-dlp}

Legacy handling of a new code under an existing type is the least
specified corner of ICMP processing. {{Section 3.2.2.1 of RFC1122}}
partitions the Destination Unreachable codes it enumerates into hard
errors, codes 2 through 4, which abort TCP connections, and soft
errors, but prescribes nothing for codes it does not enumerate;
{{RFC4443}} likewise does not prescribe unknown-code mapping; and
{{RFC5461}} documents how divergent soft-error handling already is
among implementations for the codes that are specified. Deployed
stacks variously treat an unrecognized Destination Unreachable code
as a generic unreachable, as a soft error, as a hard error, or
discard it on a code range check.

Every point in that space is acceptable for this code. An abort is
the intended outcome for unauthorized traffic. Soft-error handling
reaches the same outcome through timer expiry, which is the status
quo. Discard on a range check leaves the sender exactly where a
silently discarding gateway leaves it today. There is no legacy
behavior in which a host handles unauthorized traffic worse than it
does now. {{exp-questions}} nonetheless lists recording actual
behavior across target-environment stacks as an experiment question,
since a stack whose unknown-code handling proved disruptive in some
unanticipated way would be a significant finding.

## Middleboxes {#middleboxes}

Stateful middleboxes and firewalls inside the deploying domain may
discard an unknown ICMPv6 type or an unfamiliar Destination
Unreachable code under default-deny policy. This fails safe, in that
senders revert to status-quo behavior, but silently defeats the
mechanism. Deployment of this specification therefore includes
updating ICMP filtering policy on the paths between gateways and the
hosts they serve, which is feasible precisely because
{{applicability}} scopes the mechanism to administered domains;
{{RFC4890}} provides the relevant ICMPv6 filtering framework.
Boundary filtering requirements appear in {{generation}} and
{{security}}.

## Legacy Senders {#legacy-senders}

Hosts that predate this specification generate nothing new and are
unaffected as senders. Where such hosts sit behind a
performance-enhancing proxy, the proxy consumes these messages on
their behalf ({{uc-proxy}}), and the mechanism's benefit does not
depend on host adoption.


# Operational Considerations {#operational}

Gateways SHOULD count generation of each message, per code and per
Admission Denial reason, and expose the counters through network
management. In a correctly provisioned domain these messages are
rare for authorized traffic, so the counters function as the
divergence telemetry of {{uc-divergence}}: a PDN generated during a
planned contact, or a denial generated for a flow the orchestrator
believes it admitted, indicates that the control plane's model and
the data plane's state disagree, independent of any effect on
senders.

ICMP rate limiters on gateways deserve attention at contact
boundaries. When a contact ends, every active trans-boundary flow
can elicit a PDN within a short interval; a token-bucket limiter
tuned for steady-state error rates may suppress most of the burst,
leaving many hosts without deferral entries for the entire gap.
Operators SHOULD size gateway ICMP rate limits for the expected flow
fan-out at gap onset; the per-destination amortization of the
deferral cache keeps the burst brief.

The timestamps of {{pso}} are only as meaningful as the domain's
time synchronization. Gateways require a synchronized clock to
generate them, which the contact-plan architecture already demands;
hosts with poor synchronization fall back to the differential
interpretation of {{pso}}. Operators measuring deferral accuracy
({{exp-questions}}) should account for host clock quality before
attributing misses to plan divergence.

Hosts SHOULD expose their deferral caches to local diagnostics, since
a populated entry changes transmission behavior in ways otherwise
invisible to troubleshooting.


# Experimental Status and Goals {#experiment}

This document is published as Experimental. The message formats and
generation rules are specified with normative precision, but the value
of the mechanism rests on operational questions that can only be
answered by deployment. This section defines the experiment: where it
runs, what it is intended to find out, and what evidence would support
advancing the mechanism on the standards track.

## Scope of the Experiment {#exp-scope}

The experiment runs within administered limited domains satisfying the
applicability criterion of {{applicability}}: a low-delay IP domain
adjacent to a scheduled or disruption-prone high-delay segment, with
gateways that hold the contact plan and admission policy for that
segment. Expected venues include space networking testbeds and
missions, tactical and maritime networks with windowed reachback, and
laboratory emulations of the same topologies. The messages are not
exchanged across uncontrolled networks, and no behavior in this
document affects hosts whose traffic never requires a trans-boundary
link.

Prior to IANA assignment of the type and code defined here,
implementations use the experimental ICMP values reserved by
{{RFC4727}} and coordinate their interpretation bilaterally, per that
document's rules. Interoperability reports from this phase are in
scope for the experiment.

## Questions the Experiment Is Intended to Answer {#exp-questions}

The experiment is intended to produce evidence on the following
questions.

Whether PDN-driven behavior outperforms timer expiry. The premise of
the PDN is that deferral and fail-fast, driven by boundary knowledge,
beat retransmission into a dead or preempted link followed by an
undifferentiated timeout. Deployments can measure this directly:
retransmission volume on trans-boundary paths, time from first send to
application-visible disposition, and offered load at the gateway during
contact gaps, each with and without PDN processing enabled.

Whether contact-plan predictions are reliable enough to act on. A
deferral cache is only as good as the next-contact times populating it.
The experiment should measure how often a host that deferred to an
advertised contact time found service available at that time, and the
operational consequences when plan and reality diverged.

How unmodified hosts respond to the new code. The Denied by Link
Policy code is assigned under Destination Unreachable on the
expectation that legacy stacks degrade to generic unreachable
handling, which is the intended outcome for unauthorized traffic. The
experiment should confirm this across the stacks actually present in
target environments, including embedded and flight-heritage
implementations, and record any stack whose handling of an
unrecognized code is disruptive.

Whether the conditional transport reaction is the right one.
{{transport}} maps a PDN to a soft or hard error by comparing the
deferral against the connection's tolerance. Deployment may show that
a simpler rule is sufficient, that the tolerance comparison needs
different inputs, or that suspended connections rarely survive the
peer's own timers in practice; any of these findings should reshape
the normative text before advancement.

Whether one notification per destination per gap is sufficient. The
rate-limiting and negative-caching design assumes a small number of
notifications amortize across a gap. Environments with large host
counts, short flows, or address-diverse traffic may stress this
assumption, and the experiment should record gateway ICMP generation
rates and cache hit behavior under realistic fan-out.

Whether the extension objects carry the right information. Field
experience may show that additional data (residual contact capacity,
multiple future contacts, confidence indicators on predicted times) is
needed, or that defined fields go unused. Unused fields are a finding,
not a failure; they should be removed on advancement rather than
carried forward.

Whether divergence telemetry is operationally useful. {{uc-divergence}}
positions these messages as a data-plane check on control-plane state.
Operators should report whether counting and reporting their
generation exposed real faults ahead of existing monitoring.

## Criteria for Advancement {#exp-criteria}

Advancement to the standards track would be supported by: at least two
independent interoperable implementations of both messages, including
extension object processing; deployment experience from at least one
operational (non-laboratory) domain meeting the applicability
criterion; measurement showing that PDN processing reduces
retransmission load or time-to-disposition on trans-boundary paths
relative to timer-based behavior in the same environment; and no
evidence of harmful interaction with unmodified hosts or with
middleboxes present in the target environments. Findings on the
transport reaction rule and extension object contents are expected to
produce revisions, not to block advancement, provided the revised
behavior is itself supported by the collected data.

If deployment experience shows the mechanism is not useful, the
honest outcome is to document that result. Reports of failed or
inconclusive experiments are requested to the same degree as
successful ones, through the DTN working group or its successor.


# Security Considerations {#security}

ICMP carries no authentication, and both messages are actionable, so
the principal threats are forgery by off-path attackers and
information disclosure to the senders these messages answer. The
containment for all of them is the limited-domain scoping of
{{applicability}}: validation against sender state ({{validation}}),
boundary filtering ({{generation}}), and a single administrative
authority. None of these defenses is dependable across the open
internet, which is among the reasons the mechanism is not specified
for it.

A forged PDN is a traffic-suppression attack: a sender that accepts
one pauses trans-boundary traffic for the advertised interval. The
invoking-packet validation of {{RFC5927}} forces the attacker to guess
connection state, boundary filtering confines injection to on-path or
in-domain attackers, and the deferral-cache lifetime bound of
{{deferral-cache}} caps the damage of a far-future forged contact
time. An in-domain attacker positioned to forge these convincingly is
generally positioned to drop traffic outright, which bounds the
marginal capability the message adds.

A forged Denied by Link Policy message is a connection-reset attack
in the class {{RFC5927}} analyzes for existing hard-error codes; it
adds no capability beyond forging the existing administratively
prohibited codes, and the same mitigations apply.

The Denied by Link Policy message discloses policy to the sender it
refuses, and a responsive gateway is a probing oracle: an attacker
varying markings, sources, and destinations can map which traffic the
domain admits. {{generation}} therefore requires per-policy control
over generation and reason granularity, including reason value 1,
dimension not disclosed, and silent discard; toward any sender the
operator does not trust, silent discard is the expected posture,
consistent with existing firewall practice.

The PDN's schedule fields disclose contact times, and in some
deployments the contact schedule is itself sensitive operational
information. {{generation}} permits omitting the time fields per
policy; operators of such deployments should protect schedule fields
as they protect the contact plan.

Neither message creates an amplification vector: each is generated
only toward the domain interior, is rate limited, and is no larger
than ordinary extended ICMP errors.


# IANA Considerations {#iana}

This document requests the following assignments, shown as TBD
values throughout.

From the ICMPv6 "Type" registry: a new error message type TBD1, Path
Delay Notification, with a code sub-registry initialized per {{pdn}}
(0, no contact active; 1, contact active, stated delay applies; 2,
preempted within contact), registration policy Specification
Required.

From the ICMPv4 "Type" registry: a new type TBD2, Path Delay
Notification, with the same code sub-registry.

From the code registry for ICMPv6 Type 1, Destination Unreachable:
code TBD3, Denied by Link Policy.

From the code registry for ICMPv4 Type 3, Destination Unreachable:
code TBD4, Denied by Link Policy.

From the "ICMP Extension Object Classes and Class Sub-types" registry
{{RFC4884}}: a new class TBD5, Path Signaling Object, with C-Type 1,
Path Schedule, and C-Type 2, Admission Denial, assigned by this
document; further C-Types are Specification Required.

A new registry, "Admission Denial Reason Codes," initialized per
{{ado}}, registration policy Specification Required.


--- back

# Worked Example {#example}

A gateway with no active contact receives a packet for a
trans-boundary destination at 18:00:00 UTC on 16 September 2026. Its
contact plan shows the next contact serving the destination starting
42 minutes later, at 18:42:00 UTC, lasting 15 minutes. It
generates a PDN, code 0, whose Path Schedule Object decodes as
follows. The object's Class-Num octet is TBD5 and is omitted from
the field listing.

| Field | Wire value | Decoded |
|:------|:-----------|:--------|
| Generation Time | 0xEE5557A0 0x00000000 | 2026-09-16 18:00:00Z |
| Next Contact Time | 0xEE556178 0x00000000 | 2026-09-16 18:42:00Z |
| Expected Delay | 0xFFFFFFFF | no finite bound |
| Contact Duration | 0x000DBBA0 | 900000 ms (15 min) |
{: #tab-example title="Worked Example Path Schedule Object"}

The timestamps are NTP 64-bit values: the seconds words above are the
Unix times of the two instants plus 2208988800, and both fraction
words are zero. A synchronized receiver caches a deferral entry for
the destination expiring at 18:42:00Z. A receiver that does not
trust its clock computes Next Contact Time minus Generation Time,
2520 seconds, and defers for that interval from message receipt.
The all-ones Expected Delay is the disrupted case: no finite delay
bound exists until the contact opens.


# Acknowledgments
{:numbered="false"}

TODO acknowledge.
