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
area: INT
workgroup: Internet Area Working Group
keyword:
 - IP enclave
 - delay/disruption links
 - ICMP
 - ICMPv4
 - ICMPv6
venue:
  group: INTAREA
  type: Working Group
  mail: int-area@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/int-area/
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
  RFC5905:

informative:
  RFC1122:
  RFC1812:
  RFC4890:
  RFC4950:
  RFC5461:
  RFC5482:
  RFC5837:
  RFC5927:
  RFC8799:
  RFC8877:
  RFC8883:
  RFC9171:
  RFC9293:

...

--- abstract

This document defines two ICMP Destination Unreachable codes generated
by the gateway at the boundary between a low-delay IP network and a
constrained link, such as a scheduled deep-space link, whose usability
is intermittent and whose use is governed by admission policy. Link
Temporarily Unavailable reports that an otherwise admissible packet
was not forwarded because the required link is not presently usable,
and may carry optional predictive metadata including the expected time
at which the link will become usable. Denied by Link Policy reports
that the packet was not forwarded because the link's admission policy
does not admit it. Both convey information the gateway already holds
to senders at local timescales, allowing transports and applications
to defer traffic, fail fast, or select another communication mechanism
rather than discover the condition through timer expiry.


--- middle

# Introduction {#intro}

In networks that include constrained links -- deep-space links,
windowed satellite reachback, periodic store-and-forward contacts --
an IP sender has no visibility into the condition of the path beyond
its own timers. The router at the boundary between the sender's
low-delay network and the constrained link knows more: it holds a
contact plan describing when the link will next carry traffic, and it
enforces an admission policy describing whose traffic the link will
carry. Today that knowledge shapes forwarding at the boundary but is
not communicated to senders, whose transports retransmit into
unavailable or unauthorized links until timers expire and whose
applications receive a failure indistinguishable from a routing fault
or a crashed peer.

This document defines two ICMP Destination Unreachable codes, for both
ICMPv4 and ICMPv6, that move that knowledge to senders. Link
Temporarily Unavailable reports that a packet which was otherwise
admissible was not forwarded because the constrained link required to
reach its destination is not presently usable. The gateway MAY attach
predictive metadata in ICMP extension objects {{RFC4884}}: the time at
which it generated the message, the time at which it expects the link
to become usable, and the approximate one-way delay of the link. Denied
by Link Policy reports that the packet was not forwarded because the
link's admission policy does not admit it, and MAY identify the policy
dimension on which admission failed. Neither code describes what
traffic would be admitted, and neither reports on a packet that was
forwarded.

The mechanism depends on the boundary being close and the condition
being long-lived: gateway feedback reaches senders in local round-trip
times, while the unavailability it reports lasts minutes to hours
({{applicability}}). Receivers can use it to defer traffic until the
link is expected to be usable, to fail fast when the expected wait
exceeds an application's tolerance, or to select a different
communication mechanism such as a Bundle Protocol {{RFC9171}} agent for
store-and-forward delivery. This document does not prescribe any of
these responses; the codes carry information, and endpoint and
application policy determine its use.

Where an application can learn in advance which communication
mechanisms suit a destination -- through configuration, provisioning,
DNS, or orchestration -- selecting an appropriate mechanism before
initiating unsuitable IP communication is preferable to discovering
the condition reactively. Such discovery and selection mechanisms are
out of scope for this document, which addresses only what a gateway
reports once a packet has been sent.

These codes are specified for use within administered limited domains
{{RFC8799}}. {{experiment}} defines the experiment this document
proposes.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Constrained link:
: a link whose usability is scheduled, intermittent, or otherwise
  restricted, and whose use is governed by admission policy.

Contact:
: an interval during which a constrained link is planned to carry
  traffic between a gateway and its remote counterpart.

Contact plan:
: the schedule of contacts for a link or set of links, distributed to
  gateways in advance of execution.

Low-delay domain:
: an administered IP network whose internal round-trip times are
  negligible relative to the unavailability durations of an adjacent
  constrained link. The definition is relative, not absolute
  ({{applicability}}).

Enclave:
: a low-delay domain reachable from other domains only across a
  constrained link.

Gateway:
: the router at the boundary between a low-delay domain and a
  constrained link, holding the contact plan and admission policy for
  that link and generating the codes defined in this document.

Signaling domain:
: the set of gateways and paths whose aggregate ability to serve
  traffic toward a destination a signal describes. In this document
  it coincides with the low-delay domain under a single administrative
  authority.

Trans-boundary traffic:
: traffic whose path crosses a gateway onto a constrained link.

Admission:
: a gateway's determination, against local policy, that particular
  traffic may use a constrained link.

Deferral cache:
: per-destination state a receiver maintains from Link Temporarily
  Unavailable metadata and consults before transmission
  ({{deferral-cache}}).


# Applicability {#applicability}

The mechanisms in this document apply at a boundary between a
low-delay domain and a constrained link whose unavailability lasts
long enough that feedback generated at the boundary reaches senders in
time to change their behavior.

Let R be the round-trip time between a sender and the gateway
generating these codes, D the duration of the unavailability being
reported, and T the timescale on which the sender's transport would
otherwise detect failure through retransmission timer growth and user
timeout. The codes are useful when R is small relative to T, so the
notification arrives while the sender still has options, and R is
small relative to D, so the reported condition is still in effect when
the notification arrives. In the Mars enclave of {{conops}}, R is
milliseconds to seconds, T is seconds to minutes, and D is minutes to
hours.

When R approaches D, a notification describes a link state that may no
longer exist on arrival. Gateways therefore generate these codes only
into their local low-delay domain and never across the constrained
link: an Earth-side gateway signals terrestrial senders, a Mars-side
gateway signals enclave hosts, and neither signals across the link
between them. "Local" is consequently relative, not absolute. The
entire terrestrial internet, at sub-second internal round-trip times,
is a single low-delay domain relative to a link whose unavailability
is measured in minutes, and a vehicle-area network is one relative to a
SATCOM link whose availability is measured in scheduled windows of tens
of minutes.

The mechanism is further applicable only where the signaling domain
can determine that no path within it can serve the invoking traffic
toward its destination. The codes describe the domain's aggregate
ability to serve the traffic, not the local state of the gateway that
happened to receive the packet; a signal is meaningful only when
receiving it tells the sender something true about every path the
domain could offer. How the domain obtains this knowledge -- from its
routing architecture, orchestration, the contact plan, or other
domain-specific mechanisms -- is not specified here. The mechanism is
not applicable to arbitrary routing architectures in which a gateway
cannot make this determination ({{generation}}).

The same criteria identify deployments beyond the interplanetary case:
tactical edge networks behind scheduled reachback links, maritime and
airborne platforms with windowed satellite connectivity, sensor
networks served by periodic data-mule contacts, and remote ground
stations with pass-limited access. Conversely, the mechanism offers
nothing on paths that are merely lossy or congested; existing
congestion signaling and routing convergence address those conditions.

These codes are specified for use within limited domains {{RFC8799}}
under a single administrative authority. ICMP carries no
authentication, so forged notifications are contained only by
validation against sender state and filtering at domain boundaries,
and the Denied by Link Policy code discloses admission policy that is
diagnostic within a cooperative domain but reconnaissance outside it
({{security}}). Boundary routers discard these codes on ingress from
and egress to uncontrolled networks.


# Concept of Operations {#conops}

This section is informative. It describes a representative deployment:
an IP-based enclave in Mars orbit and on the Martian surface, connected
to terrestrial networks through scheduled deep-space contacts.

## Reference Scenario {#scenario}

The enclave consists of surface assets and orbiters interconnected by
conventional IP links, with propagation delays of milliseconds to low
seconds and continuous or near-continuous connectivity. Standard IP
routing and transports operate normally within it.

The enclave connects to Earth through gateway nodes on relay orbiters
equipped with deep-space links. These links are scheduled: contacts
with ground stations occur in windows determined by orbital geometry
and antenna allocation, known in advance through a contact plan. They
are high-delay: one-way light time to Earth is roughly 3 to 22
minutes. And they are policy-managed: contact capacity is allocated to
specific senders or traffic classes, and traffic outside those
allocations is not authorized to use the link.

The gateways run a Bundle Protocol version 7 {{RFC9171}} agent.
Trans-boundary data normally crosses as bundles, either because the
originating host is BP-capable or because the gateway encapsulates IP
traffic on its behalf.

The same structure exists on the Earth side. A terrestrial sender's
traffic crosses an Earth-side gateway that holds the same contact plan
and enforces its own admission policy, and that gateway generates the
same codes toward terrestrial senders. This section describes the Mars
side for concreteness.

## Roles {#roles}

Enclave hosts originate and receive application traffic. Some are
DTN-aware, running a BP agent or a stack extension that can hand
traffic to one; others are unmodified IP hosts, including legacy
instruments and commercial payloads whose stacks predate this
specification. Where a performance-enhancing proxy (PEP) terminates
transport sessions on behalf of such hosts, the PEP consumes these
codes for them, applying the behaviors of {{use-cases}} on their behalf
while the hosts see locally acknowledged sessions with enclave-scale
round-trip times. This is the expected deployment pattern for hosts
that cannot be modified, and it means the mechanism's benefit does not
depend on universal host adoption.

Gateways hold the contact plan for their deep-space links, enforce
admission and precedence policy on contact capacity, and generate the
codes defined in this document: Link Temporarily Unavailable when
admitted trans-boundary traffic arrives while the link is not usable,
and Denied by Link Policy when traffic arrives that policy does not
admit. In this deployment, enclave routing is driven by the same
contact plan the gateways hold, so traffic reaches a gateway that
cannot serve it only when no gateway can; this is what makes the
aggregate-domain invariant of {{applicability}} hold. The appropriate
response to either code is deferral, selection of another
communication mechanism, or provisioning, never a routing retry.

An orchestration function, which may be Earth-based, distributes
contact plans and admission policy to gateways in advance. Since
distribution itself crosses the deep-space link, gateways operate
autonomously between management contacts, and their local plan and
policy are authoritative for their links.

Intra-enclave traffic is unaffected. The codes are generated only for
traffic that requires a trans-boundary link, and hosts that never
originate such traffic need not implement them.

## Use Cases {#use-cases}

### Deferral Until the Link Is Usable {#uc-deferral}

A surface host begins a bulk transfer toward Earth between contacts.
The first packet reaches the gateway, which drops it and returns Link
Temporarily Unavailable carrying its generation time and the time at
which it expects the link to become usable. The host records the
notification against the destination, suppresses transmission until
the indicated time, and informs the application that the path is
deferred rather than failed. The entry expires at the expected
usability time, so one notification per destination per gap suffices,
which matters because the gateway rate-limits ICMP generation like any
router.

### Selection of Another Communication Mechanism {#uc-handoff}

A DTN-aware host receiving Link Temporarily Unavailable may, as a
matter of local policy, hand the affected traffic to its local BP
agent, which originates bundles toward the destination; these queue at
the gateway and are forwarded during the next contact. An application
with its own delay tolerance may instead compare the expected wait
against that tolerance and fail fast. The code does not prescribe
either response; it supplies the information on which the endpoint
decides.

### Traffic Denied by Link Policy {#uc-denied}

A misconfigured or newly integrated payload host sends best-effort
traffic toward Earth during a contact whose capacity is allocated to
command and telemetry classes. The gateway classifies the traffic
against its local policy, which may consider DSCP, addresses, protocol
and port, or any other criteria, and returns Denied by Link Policy
with a reason identifying the dimension that failed. The code is
diagnostic, not prescriptive: it does not describe what traffic would
be admitted, since admission policy is multidimensional and local, and
coaching senders on acceptable markings would invert an admission
model in which access is granted through the control plane ({{ado}}).
The host, or the operator debugging it, learns why the traffic was
refused; obtaining admission is a provisioning action.

A flow that policy admitted at contact start and later displaces in
favor of higher-precedence traffic falls under the same code. Whether
an implementation regards this internally as preemption, resource
allocation, or precedence enforcement is a matter of local policy;
the sender learns only that the traffic is not presently admitted.

### Plan/Reality Divergence {#uc-divergence}

In nominal operation, provisioned traffic should never elicit these
codes. Link Temporarily Unavailable generated during a planned
contact, or Denied by Link Policy generated for a flow the
orchestrator believes it admitted, indicates that the control plane's
model and the data plane's state have diverged: an unplanned outage, a
policy distribution failure, or traffic outside the provisioning
workflow. Gateways count and report generation of these codes so that
operators can use them as a diagnostic feed ({{operational}}).

## Receiver Processing Model {#receiver-model}

A receiving host processes these codes at three levels. The stack may
cache the expected usability time per destination and consult it
before transmitting. The transport may compare the expected wait
against the connection's own tolerance and treat the notification as a
soft or hard error accordingly; Denied by Link Policy indicates a
condition that retrying unchanged will not resolve. The API may expose
both conditions to applications with their metadata, so that
applications can defer or select another mechanism themselves.
Requirements and open questions are given in {{receiver}}.

## Interaction of the Two Codes {#interaction}

The two codes answer different questions and may both apply to one
flow: a link can be simultaneously restricted and unavailable.
Admission is evaluated first. Traffic that would be refused regardless
of link state receives Denied by Link Policy, which avoids disclosing
link schedule information to traffic that could not use the link
anyway; traffic that is admitted but whose link is not presently
usable receives Link Temporarily Unavailable. A sender whose denial is
resolved through provisioning may then receive Link Temporarily
Unavailable for the same flow: first become admissible, then wait for
the link.


# Message Formats and Extension Objects {#formats}

This section is normative. Code and object class values are shown as
TBD pending IANA assignment ({{iana}}); prior to assignment,
implementations use the experimental values of {{RFC4727}} as
described in {{exp-scope}}.

## Common Message Format {#common}

Both codes are carried in Destination Unreachable messages: ICMPv6
Type 1 {{RFC4443}} and ICMPv4 Type 3 {{RFC0792}}. The message layout is
the extended Destination Unreachable message of {{RFC4884}}, which
permits an ICMP Extension Structure on Destination Unreachable
messages of both versions; this document adds no fields. The Length
field, original-datagram truncation and padding, and extension
structure placement follow {{Section 4 of RFC4884}}. Attaching
extension objects to Destination Unreachable is an established pattern
({{RFC4950}}, {{RFC5837}}, {{RFC8883}}).

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |            Checksum           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Length     |                    Unused                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
~   As much of the invoking packet as possible, zero-padded    ~
|   per Section 4 of RFC 4884                                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
~              ICMP Extension Structure (RFC 4884)              ~
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-du title="Extended Destination Unreachable (ICMPv6 layout shown; ICMPv4 places Length in the second octet of the second word)"}

A receiver that does not recognize a code treats the message as a
generic Destination Unreachable; the extension structure lies beyond
the portion of the original datagram such receivers examine. Any
receiver that recognizes either code postdates this specification and
parses the extension structure unconditionally, so the
backwards-compatibility ambiguities of {{Section 5 of RFC4884}} do not
arise.

A receiver MUST ignore extension objects of the class defined here
with an unrecognized C-Type, and MUST process at most one object of
each C-Type per message, taking the first if several are present.

## Link Temporarily Unavailable {#ltu}

Link Temporarily Unavailable is code TBD1 under ICMPv6 Destination
Unreachable and code TBD2 under ICMPv4 Destination Unreachable. It
reports that the invoking packet was admissible for the constrained
link required to reach its destination, that no path in the signaling
domain could serve it, and that the packet was not forwarded because
that link is not presently usable. The condition concerns the link,
not ordinary routing failure or congestion.

The message MAY carry any of the Generation Time, Expected Link
Usability Time, and Expected Link Delay objects ({{objects}}). The
message is meaningful without any of them: it reports the condition,
and the objects add prediction where the gateway has it. If Expected
Link Usability Time is present, Generation Time MUST also be present.

## Denied by Link Policy {#dlp}

Denied by Link Policy is code TBD3 under ICMPv6 Destination
Unreachable and code TBD4 under ICMPv4 Destination Unreachable. It
reports that the invoking packet was not forwarded because the
admission policy governing the constrained link required to reach its
destination does not admit it, independent of the link's current
usability.

The outcome resembles that of the existing administratively prohibited
codes, and those codes are capable of representing the condition. This
code differs in identifying refusal by the admission policy of a
specific constrained link, and in carrying a structured reason.
Whether that distinction warrants a dedicated code is left for review
and experimentation ({{experiment}}).

The message SHOULD include exactly one Admission Denial Object
({{ado}}). A message without one reports the denial with no reason,
which a gateway MAY choose where policy forbids disclosure
({{generation}}).

## Extension Objects {#objects}

The objects below share a single ICMP Extension Object class, the Link
Condition Object class (Class-Num TBD5), distinguished by C-Type. The
object header is as defined in {{Section 8 of RFC4884}}. Each object
is independently optional; presence conveys availability, and absence
means the gateway did not supply that information. No sentinel values
are defined for absent data.

### Generation Time {#gen-time}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Length = 12           |  Class-Num =  |  C-Type = 1   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                       Generation Time                         +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-gen-time title="Generation Time Object"}

Generation Time is the gateway's clock reading at the time it
generated the message, in the timestamp format of {{timestamps}}. It
dates any other metadata in the message and, together with Expected
Link Usability Time, allows a receiver to derive a predicted interval
without sharing the gateway's time reference.

### Expected Link Usability Time {#elut}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Length = 12           |  Class-Num =  |  C-Type = 2   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                 Expected Link Usability Time                  +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-elut title="Expected Link Usability Time Object"}

Expected Link Usability Time is the earliest time, according to
information available to the signaling gateway, at or after which the
constrained link is expected to be usable for forwarding traffic, in
the timestamp format of {{timestamps}}. The value refers to data-plane
usability, not to the start of a scheduled contact or of preparation
for one: antenna pointing, modem configuration, acquisition,
synchronization, and analogous link-establishment operations occur
before the reported time and are not represented separately. The value
is predictive, not a guarantee that the link will be usable at that
instant.

This object MUST NOT be sent without a Generation Time object in the
same message. Both MUST be derived from the same clock. A receiver
that receives this object without Generation Time MUST ignore it.

### Expected Link Delay {#eld}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Length = 8           |  Class-Num =  |  C-Type = 3   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Expected Link Delay                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-eld title="Expected Link Delay Object"}

Expected Link Delay is the approximate one-way delay associated with
traversal of the constrained link whose unavailability prevented
forwarding, in milliseconds, as an unsigned 32-bit integer. It is an
estimate, not a bound; a quantity of the link, not an end-to-end path
delay; and independent of the waiting time until the link becomes
usable. This document does not require receivers to combine it with
Expected Link Usability Time or otherwise compute an end-to-end
delivery time; how the estimate is used is a matter of endpoint and
application policy.

### Admission Denial {#ado}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Length = 8           |  Class-Num =  |  C-Type = 4   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Reason             |            Reserved           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-ado title="Admission Denial Object"}

Reserved MUST be zero on transmission and ignored on receipt. Reason
identifies the policy dimension on which admission failed, from a new
IANA registry with the following initial values:

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

Reasons identify the failed dimension and never describe what would
be admitted ({{uc-denied}}). Values 5 and 6 cover refusals arising
from resource allocation and precedence, including conditions an
implementation may internally regard as preemption; they remain
policy refusals from the sender's perspective. Value 1 allows a
gateway to deliver a definite denial while disclosing nothing, as an
alternative to silent discard. Registration policy for new values is
Specification Required.

## Timestamp Format {#timestamps}

Generation Time and Expected Link Usability Time are 64-bit values in
the format of the NTP timestamp of {{Section 6 of RFC5905}}: 32 bits
of seconds followed by 32 bits of fraction, with resolution 2^-32
seconds and wrapping every 2^32 seconds. Per the timestamp
specification template of {{RFC8877}}: the epoch is the NTP epoch, 1
January 1900 UTC, where the gateway's clock is expressed in the NTP
timescale; no era number is carried.

This document does not require that the gateway's clock be
synchronized to any reference shared with receivers. Both timestamps
in one message MUST be derived from the same clock, so that their
difference is meaningful. A receiver that can relate the transmitted
values to its own time reference MAY interpret Expected Link
Usability Time directly. A receiver that cannot MAY instead derive the
predicted interval (Expected Link Usability Time - Generation Time),
computed with unsigned modular arithmetic, and apply it from the time
of receipt. Applying the interval from receipt is conservative when
delivery of the notification was delayed; this document does not
assume that ICMP delivery is prompt.


# Generation Rules {#generation}

The rules in this section govern gateways. General ICMP generation
rules apply in addition: {{Section 2.4 of RFC4443}} for ICMPv6,
including its rate-limiting requirements, and {{RFC1812}} for ICMPv4.
Nothing in this document permits generating an error in response to
an ICMP error, to a packet addressed to a multicast destination, or
in the other cases those documents prohibit.

On receiving trans-boundary traffic, a gateway proceeds as follows:

1. Determine whether another path in the signaling domain can serve
   the traffic. If so, use that path and generate neither code.

2. Apply the constrained link's admission policy.

3. If the traffic is not admitted, drop it and, where configured,
   send Denied by Link Policy.

4. If the traffic is admitted but the required link is not presently
   usable, drop it and send Link Temporarily Unavailable.

5. If the traffic is admitted and the link is usable, forward it
   normally. A forwarded packet MUST NOT elicit either code.

A gateway MUST NOT generate either code if another path within the
signaling domain can provide service inconsistent with the signal.
The codes describe the domain's aggregate ability to serve the traffic
toward the destination ({{applicability}}). A gateway that cannot
determine this property for a destination MUST NOT generate these
codes for it.

A gateway MUST NOT populate Expected Link Usability Time with a value
not derived from its current contact plan or equivalent information.
A gateway sending Expected Link Usability Time MUST also send
Generation Time set from the same clock.

Generation of both codes MUST be configurable per policy. An operator
MUST be able to configure, per source, prefix, or policy class,
whether Denied by Link Policy is generated at all (the alternative
being silent discard) and whether its Admission Denial Object carries
a specific reason or the value 1, dimension not disclosed. An operator
SHOULD be able to configure omission of the predictive metadata
objects from Link Temporarily Unavailable ({{security}}).

A gateway MUST NOT generate these codes toward the constrained link
and MUST NOT generate them in response to traffic arriving from it
({{applicability}}).

The absence of these codes promises nothing. A gateway without current
contact plan information does not send Expected Link Usability Time.


# Receiver Requirements {#receiver}

The receiver behaviors in this section have not been fully reviewed
against the current message architecture. The requirements retained
here are those needed for coherent processing; transport reaction
semantics, cache scope, and application interface are left as open
questions for review and experimentation ({{exp-questions}}).

## Validation {#validation}

A receiver MUST validate these messages as ICMP errors per
{{RFC4443}} and the mitigations of {{RFC5927}}: the invoking packet
excerpt is matched against existing connection or flow state, and a
message matching nothing the receiver sent is discarded. A receiver
SHOULD discard these messages when received on an interface facing
outside its administered domain.

## Deferral Cache {#deferral-cache}

A host processing Link Temporarily Unavailable MAY maintain a deferral
cache keyed by the exact destination address of the invoking packet. A
newer notification replaces an older entry for the same key. An entry
records the metadata received and an expiry.

An entry created from a message carrying Expected Link Usability Time
expires at that time, interpreted per {{timestamps}}; an entry created
from a message without it expires after an implementation-chosen
conservative interval. Entry lifetime MUST be bounded by a
configurable maximum; a default of 24 hours is RECOMMENDED, and
deployments whose planned gaps exceed it configure it accordingly. The
bound limits the effect of a forged notification ({{security}}).

Entries are advisory: a gateway cannot revoke a notification whose
plan has since changed. A host MAY probe before expiry, SHOULD NOT
sustain transmission against an unexpired entry, and MUST NOT treat
expiry as an assurance of service; traffic sent at expiry that finds
the link still unusable elicits a fresh notification.

Whether entries may safely cover a prefix rather than a single
destination is not resolved by this document.

## Transport Processing {#transport}

The information in these messages is available to transports
according to local policy; this document does not mandate a
particular transport reaction. The following describes an expected
approach whose suitability is an experiment question
({{exp-questions}}).

A transport may compare the expected wait, derived from a deferral
cache entry, against the connection's tolerance. For TCP {{RFC9293}},
the tolerance might be the user timeout, as adjusted by {{RFC5482}}
where negotiated, or an application-supplied value. If the expected
wait falls within the tolerance, the notification can be treated as a
soft error, with retransmission suspended until the expected usability
time rather than counted toward connection failure; if it exceeds the
tolerance, or none can be computed, the connection can be aborted and
the reason surfaced to the application. The remote endpoint receives
no corresponding notification and continues to run its own timers; a
receiver that suspends a connection beyond what it knows of the peer's
tolerance preserves nothing.

Denied by Link Policy reports a condition that retrying unchanged will
not resolve. A transport SHOULD treat it as a hard error for matching
connections and flows, and a receiver SHOULD NOT automatically retry
the traffic unchanged ({{uc-denied}}).

## Application Interface {#api}

Hosts SHOULD make both codes available to applications as conditions
distinguishable from each other and from other unreachable errors,
including any metadata received, so that applications can implement
their own deferral or mechanism-selection logic. Whether such an
interface should be standardized is not addressed here.


# Legacy Host and Middlebox Behavior {#legacy}

Both codes are new codes under existing Destination Unreachable types,
so legacy behavior is whatever a stack does with an unrecognized
Destination Unreachable code. {{Section 4.2.3.9 of RFC1122}}
partitions the codes it enumerates into hard errors (codes 2 through
4), which abort TCP connections, and soft errors, but prescribes
nothing for other codes; {{RFC4443}} is likewise silent, and
{{RFC5461}} documents how divergent soft-error handling already is
among implementations. Deployed stacks variously treat an unrecognized
code as a generic unreachable, as a soft error, as a hard error, or
discard it on a range check. This document does not assume that all
stacks behave alike; recording actual behavior across the stacks
present in target environments is an experiment question
({{exp-questions}}).

For Denied by Link Policy, each of these behaviors is acceptable: an
abort is the intended outcome for unauthorized traffic, soft-error
handling reaches the same outcome through timer expiry, and discard
leaves the sender where a silently discarding gateway leaves it today.

For Link Temporarily Unavailable, soft-error handling and discard
reproduce the status quo. A stack that treats the unrecognized code
as a hard error aborts a connection that might have survived a short
unavailability; whether this occurs in practice, and whether it is
worse than the timeout the connection would otherwise have suffered,
is an experiment question.

Firewalls and stateful middleboxes inside the deploying domain may
discard unfamiliar Destination Unreachable codes under default-deny
policy. This fails safe but silently defeats the mechanism, so
deployment includes updating ICMP filtering policy on the paths
between gateways and the hosts they serve; {{RFC4890}} provides the
ICMPv6 filtering framework. Legacy hosts generate nothing new and are
unaffected as senders.


# Operational Considerations {#operational}

Gateways SHOULD count generation of each code, per Admission Denial
reason where applicable, and expose the counters through network
management. In a correctly provisioned domain these codes are rare
for authorized traffic, so the counters serve as the divergence
telemetry of {{uc-divergence}}.

ICMP rate limiters deserve attention at contact boundaries. When a
link becomes unusable, every active trans-boundary flow can elicit
Link Temporarily Unavailable within a short interval, and a
token-bucket limiter tuned for steady-state error rates may suppress
most of the burst, leaving hosts without deferral entries for the
entire gap. Operators SHOULD size gateway ICMP rate limits for the
expected flow fan-out at gap onset; the deferral cache keeps the burst
brief.

Expected Link Usability Time is only as meaningful as the gateway's
knowledge of the link and, for absolute interpretation, the domain's
time synchronization. Receivers without a shared time reference fall
back to the interval interpretation of {{timestamps}}.

Hosts SHOULD expose their deferral caches to local diagnostics, since
a populated entry changes transmission behavior in ways otherwise
invisible to troubleshooting.


# Experimental Status and Goals {#experiment}

This document is published as Experimental. The message formats and
generation rules are specified with normative precision, but the value
of the mechanism rests on operational questions that only deployment
can answer, and the receiver reaction rules have not been validated.

## Scope of the Experiment {#exp-scope}

The experiment runs within administered limited domains meeting the
criteria of {{applicability}}: space networking testbeds and missions,
tactical and maritime networks with windowed reachback, and laboratory
emulations of the same topologies. The codes are not exchanged across
uncontrolled networks, and no behavior in this document affects hosts
whose traffic never requires a trans-boundary link.

Prior to IANA assignment, implementations use the experimental ICMP
values reserved by {{RFC4727}} and coordinate their interpretation
bilaterally, per that document's rules. Interoperability reports from
this phase are in scope for the experiment.

## Questions the Experiment Is Intended to Answer {#exp-questions}

The experiment is intended to produce evidence on the following
questions:

- Whether actionable Destination Unreachable signaling reduces futile
  retransmission and offered load during link-unavailable periods,
  measured as retransmission volume on trans-boundary paths, time from
  first send to application-visible disposition, and offered load at
  the gateway during gaps, with and without processing of the new
  codes.

- Whether Expected Link Usability Time is accurate and useful: how
  often a host that deferred to the advertised time found the link
  usable at that time, and the operational consequences when it did
  not.

- Whether the optional extension metadata carries the right
  information, whether defined objects go unused, and whether
  additional objects are needed. Unused objects should be removed on
  advancement rather than carried forward.

- How unmodified hosts respond to the new Destination Unreachable
  codes across the stacks present in target environments, including
  embedded and flight-heritage implementations, and in particular
  whether any stack's hard-error handling of Link Temporarily
  Unavailable is harmful ({{legacy}}).

- What transport reaction rules are appropriate, including whether
  the tolerance comparison described in {{transport}} is the right
  approach and whether suspended connections survive the peer's own
  timers in practice.

- Whether exact-destination deferral caching is sufficient, or whether
  prefix-scoped caching is needed and can be made safe.

- Whether gateway ICMP rate limiting, as deployed, allows one
  notification per destination per gap to reach senders under
  realistic fan-out.

- Whether divergence telemetry ({{uc-divergence}}) exposed real faults
  ahead of existing monitoring.

- Whether the information disclosed by Denied by Link Policy reasons
  and by predictive metadata is operationally acceptable, and whether
  the configurable disclosure controls of {{generation}} are
  sufficient.

- Whether Denied by Link Policy warrants a dedicated code, or whether
  an existing administratively prohibited code with the Admission
  Denial Object would serve.

## Criteria for Advancement {#exp-criteria}

Advancement to the standards track would be supported by at least two
independent interoperable implementations of both codes, including
extension object processing; deployment experience from at least one
operational, non-laboratory domain meeting the applicability
criteria; measurement showing that processing the codes reduces
retransmission load or time-to-disposition on trans-boundary paths
relative to timer-based behavior in the same environment; and no
evidence of harmful interaction with unmodified hosts or middleboxes
in the target environments. Findings on transport reaction rules,
cache scope, and extension object contents are expected to produce
revisions rather than block advancement. Reports of failed or
inconclusive experiments are requested to the same degree as
successful ones.


# Security Considerations {#security}

ICMP carries no authentication, and both codes are actionable, so the
principal threats are forgery by off-path attackers and information
disclosure to the senders these messages answer. The containment for
both is the limited-domain scoping of {{applicability}}: validation
against sender state ({{validation}}), boundary filtering
({{generation}}), rate limiting, and a single administrative
authority. None of these defenses is dependable across the open
internet.

A forged Link Temporarily Unavailable message is a traffic-suppression
attack: a sender that accepts one may pause trans-boundary traffic for
the advertised interval. The invoking-packet validation of {{RFC5927}}
forces the attacker to guess connection state, boundary filtering
confines injection to on-path or in-domain attackers, and the
deferral-cache lifetime bound of {{deferral-cache}} caps the damage of
a forged far-future usability time.

A forged Denied by Link Policy message is a connection-reset attack in
the class {{RFC5927}} analyzes for existing hard-error codes; it adds
no capability beyond forging the existing administratively prohibited
codes, and the same mitigations apply.

Denied by Link Policy discloses policy to the sender it refuses, and a
responsive gateway is a probing oracle: an attacker varying markings,
sources, and destinations can map which traffic the domain admits.
{{generation}} therefore requires per-policy control over generation
and reason granularity, including reason value 1, dimension not
disclosed, and silent discard; toward any sender the operator does not
trust, silent discard is the expected posture, consistent with
existing firewall practice.

The predictive metadata of Link Temporarily Unavailable discloses link
schedule information, which in some deployments is sensitive
operational information. {{generation}} permits omitting it per
policy; operators of such deployments should protect the metadata as
they protect the contact plan. Evaluating admission before link state
({{generation}}) prevents disclosure of schedule information to
traffic that would not be admitted regardless.

Neither code creates an amplification vector: each is generated only
toward the domain interior, is rate limited, and is no larger than
ordinary extended ICMP errors.


# IANA Considerations {#iana}

This document requests the following assignments, shown as TBD values
throughout. No numeric values are proposed.

From the "Type 1 - Destination Unreachable" code registry of the
"ICMPv6 Parameters" registry: code TBD1, Link Temporarily Unavailable,
and code TBD3, Denied by Link Policy. The registration procedure for
this registry is Standards Action or IESG Approval; this document
requests IESG Approval.

From the "Type 3 - Destination Unreachable" code registry of the
"ICMP Parameters" registry: code TBD2, Link Temporarily Unavailable,
and code TBD4, Denied by Link Policy. The registration procedure for
this registry is IESG Approval or Standards Action; this document
requests IESG Approval.

From the "ICMP Extension Object Classes and Class Sub-types" registry
{{RFC4884}}: a new class TBD5, Link Condition Object, with C-Type 1,
Generation Time; C-Type 2, Expected Link Usability Time; C-Type 3,
Expected Link Delay; and C-Type 4, Admission Denial, assigned by this
document. Further C-Types are Specification Required.

A new registry, "Admission Denial Reason Codes," initialized per
{{ado}}, registration policy Specification Required.

Publication as Experimental does not by itself satisfy the
registration procedures for the Destination Unreachable code
registries. Should the requested code assignments not be approved,
the allocation strategy is a matter for further review.


--- back

# Worked Example {#example}

A gateway with no usable deep-space link receives a packet for a
trans-boundary destination at 18:00:00 UTC on 16 September 2026. No
other path in the signaling domain can serve the traffic. The packet
is admissible under the link's policy, but the link is not presently
usable; the gateway's contact plan indicates that the link is expected
to be usable 42 minutes later, at 18:42:00 UTC, and its one-way delay
estimate for the link is 20 minutes. The gateway drops the packet and
returns Destination Unreachable, code Link Temporarily Unavailable,
with three extension objects. The Class-Num octet of each is TBD5 and
is omitted from the listing.

| Object | C-Type | Wire value | Decoded |
|:-------|-------:|:-----------|:--------|
| Generation Time | 1 | 0xEE5557A0 0x00000000 | 2026-09-16 18:00:00Z |
| Expected Link Usability Time | 2 | 0xEE556178 0x00000000 | 2026-09-16 18:42:00Z |
| Expected Link Delay | 3 | 0x00124F80 | 1200000 ms (20 min) |
{: #tab-example title="Worked Example Extension Objects"}

The timestamps are NTP-format 64-bit values: the seconds words are the
Unix times of the two instants plus 2208988800, and both fraction
words are zero.

A receiver whose clock is synchronized to the gateway's reference
caches a deferral entry for the destination expiring at 18:42:00Z. A
receiver that cannot relate the timestamps to its own clock computes
Expected Link Usability Time minus Generation Time, 0x9D8 = 2520
seconds, and defers for that interval from the time it received the
message; if the message was delayed in delivery, this errs toward
waiting slightly longer than necessary. Either receiver may, as a
matter of local policy, use the 20-minute link delay estimate in
deciding whether the destination is suitable for its traffic once the
link is usable.


# Acknowledgments
{:numbered="false"}

The author used AI-assisted tooling in preparing this document,
including drafting and editing text and converting it to the
kramdown-rfc source format. All technical content, design decisions,
and the final text are the responsibility of the author.
