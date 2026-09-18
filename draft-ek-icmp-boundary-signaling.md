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
  RFC5837:
  RFC5927:
  RFC8799:
  RFC8877:
  RFC8883:
  RFC9171:

...

--- abstract

This document defines two ICMP Destination Unreachable codes generated
by the gateway at the boundary between a low-delay IP network and a
constrained link, such as a scheduled deep-space link, whose usability
is intermittent and whose use is governed by admission policy. Link
Temporarily Unavailable reports that an otherwise admissible packet
was not forwarded because the required link is presently unusable
under a transient condition known to the domain, and may carry
optional predictive metadata including the expected time until the
link becomes usable. Denied by Link Policy reports that the packet was
not forwarded because the link's admission policy does not admit it.
This document specifies the conditions under which the codes are
generated and the information they convey. It does not specify how
transports, applications, or hosts react to them; making the
boundary's knowledge available to senders is intended to enable
experimentation with such reactions.


--- middle

# Introduction {#intro}

In networks that include constrained links -- deep-space links,
satellite links available only during scheduled passes, periodic
store-and-forward contacts -- an IP sender has no visibility into the condition of the path beyond
its own timers. The router at the boundary between the sender's
low-delay network and the constrained link knows more: it knows, from
a contact plan or equivalent domain knowledge, when the link is
expected to carry traffic, and it enforces an admission policy
describing whose traffic the link will carry. Today that knowledge
shapes forwarding at the boundary but is not communicated to senders,
whose transports retransmit into unavailable or unauthorized links
until timers expire and whose applications receive a failure
indistinguishable from a routing fault or a crashed peer.

This document defines two ICMP Destination Unreachable codes, for both
ICMPv4 and ICMPv6, that move that knowledge to senders. Link
Temporarily Unavailable reports that a packet which was otherwise
admissible was not forwarded because the constrained link required to
reach its destination is presently unusable under a transient
condition the domain knows about. The gateway MAY attach predictive
metadata in ICMP extension objects {{RFC4884}}: the time at which it
generated the message, its estimate of the time until the link becomes
usable, and the approximate one-way delay of the link. Denied by Link
Policy reports that the packet was not forwarded because the link's
admission policy does not admit it, and MAY identify the policy
dimension on which admission failed. Neither code describes what
traffic would be admitted, and neither reports on a packet that was
forwarded.

This document specifies the network-layer signals: when a gateway
generates each code, what each means, and the semantics and encoding
of the optional metadata. It does not specify how a transport,
application, host cache, provisioning system, or orchestrator reacts
to them. Such reactions are not required for interoperability, and
they are not merely out of scope: a purpose of publishing the signals
is to enable experimentation with them. A host might defer further
attempts; a transport might use the information as an input to its
timeout and failure decisions; an application might expose the
condition, select another communication mechanism such as a Bundle
Protocol {{RFC9171}} agent for store-and-forward delivery, or trigger
provisioning. Experience with such reactions may motivate later
specifications ({{experiment}}).

The mechanism depends on the boundary being close and the condition
being long-lived: gateway feedback reaches senders in local round-trip
times, while the unavailability it reports lasts minutes to hours
({{applicability}}).

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
  restricted according to knowledge held by the domain, and whose use
  is governed by admission policy. Ordinary transient interface or
  carrier failure does not by itself make a link constrained
  ({{applicability}}).

Contact:
: an interval during which a constrained link is planned to carry
  traffic between a gateway and its remote counterpart.

Contact plan:
: the schedule of contacts for a link or set of links, distributed to
  gateways in advance of execution. A contact plan is one source of
  the domain knowledge this document relies on, not the only one.

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
  constrained link, holding the domain's knowledge of that link's
  availability and admission policy and generating the codes defined
  in this document.

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

Expected Time Until Link Usability (ETU):
: the gateway's estimate, at generation of a notification, of the
  interval from generation until the constrained link is expected to
  become usable for forwarding ({{etu}}).


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

The mechanism applies only to links whose availability the domain
deliberately models as a constrained resource: the gateway asserts a
transient service condition on the basis of domain or service
knowledge about the link, not on the basis of observed local carrier
or interface state. A router that merely notices its outgoing
interface is down has no basis to generate Link Temporarily
Unavailable, and ordinary transient link failures are not the
condition this document addresses. How the domain acquires its
knowledge -- a contact plan, orchestration, link-management
signaling, or other domain-specific means -- is not specified; the
codes encode what the gateway is entitled to assert, not how it
learned it.

The mechanism is further applicable only where the signaling domain
can determine that no path within it can serve the invoking traffic
toward its destination. The codes describe the domain's aggregate
ability to serve the traffic, not the local state of the gateway that
happened to receive the packet; a signal is meaningful only when
receiving it tells the sender something true about every path the
domain could offer. The mechanism is not applicable to arbitrary
routing architectures in which a gateway cannot make this
determination ({{generation}}).

The same criteria identify deployments beyond the interplanetary case,
wherever a domain manages a scheduled or intermittently available
link on the basis of advance knowledge. Conversely, the mechanism
offers nothing on paths that are merely lossy or congested; existing
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
to terrestrial networks through scheduled deep-space contacts. Host
and application behaviors described here are illustrations of what the
signals make possible, not requirements ({{consumers}}).

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
transport sessions on behalf of such hosts, the PEP can consume these
codes for them, so that the mechanism's benefit need not depend on
universal host adoption.

Gateways hold the contact plan for their deep-space links, enforce
admission and precedence policy on contact capacity, and generate the
codes defined in this document: Link Temporarily Unavailable when
admitted trans-boundary traffic arrives while the link is not usable,
and Denied by Link Policy when traffic arrives that policy does not
admit. In this deployment, enclave routing is driven by the same
contact plan the gateways hold, so traffic reaches a gateway that
cannot serve it only when no gateway can; this is what makes the
aggregate-domain invariant of {{applicability}} hold. Because the
signal already accounts for every path the domain could offer,
immediately retrying the same traffic is likely to be futile.

An orchestration function, which may be Earth-based, distributes
contact plans and admission policy to gateways in advance. Since
distribution itself crosses the deep-space link, gateways operate
autonomously between management contacts, and their local plan and
policy are authoritative for their links.

Intra-enclave traffic is unaffected. The codes are generated only for
traffic that requires a trans-boundary link, and hosts that never
originate such traffic need not implement them.

## Use Cases {#use-cases}

### Traffic Sent While the Link Is Unusable {#uc-deferral}

A surface host begins a bulk transfer toward Earth between contacts.
The first packet reaches the gateway, which drops it and returns Link
Temporarily Unavailable carrying its Expected Time Until Link
Usability. The host now knows that the domain regards the path as
transiently unavailable and roughly how long that is expected to last.
It might use this to avoid repeated attempts that would elicit the
same error, and to tell the application that the path is deferred
rather than failed. Whether and how a host does so is an experiment
question; ICMP is unreliable, and the gateway cannot assume that any
particular notification was received.

### Selection of Another Communication Mechanism {#uc-handoff}

A DTN-aware host receiving Link Temporarily Unavailable might, as a
matter of local policy, hand the affected traffic to its local BP
agent, which originates bundles toward the destination; these queue at
the gateway and are forwarded during the next contact. An application
with its own delay tolerance might instead compare the expected wait
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

Both codes are distinct from the existing Destination Unreachable
codes reporting no route or service absence. Those codes retain their
meaning: the domain has no way to reach the destination. Link
Temporarily Unavailable means that service exists and the traffic is
admitted, but the required link is presently unusable under a
transient condition; Denied by Link Policy means that the link may
exist and be usable, but this traffic is not admitted to it.


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
each C-Type per message, taking the first if several are present. A
receiver MUST ignore a Link Condition Object that is not applicable to
the received Destination Unreachable code ({{applicability-objects}});
the presence of such an object does not make an otherwise valid
message malformed.

## Link Temporarily Unavailable {#ltu}

Link Temporarily Unavailable is code TBD1 under ICMPv6 Destination
Unreachable and code TBD2 under ICMPv4 Destination Unreachable. A
gateway generating it asserts all of the following:

- the signaling domain determined that the invoking packet would
  otherwise be served using a constrained link;

- no alternative path in the signaling domain can provide service
  inconsistent with the signal;

- the traffic is otherwise admitted to that link;

- the required constrained link is presently unusable;

- the domain represents that unusability as a transient service
  condition, on the basis of its knowledge of the link, rather than as
  absence of a route or of service; and

- the invoking packet was not forwarded.

The code does not mean merely that a local outgoing interface is
down, and it does not report ordinary routing failure or congestion.
"Temporarily" describes the domain's model of the condition; it does
not guarantee recovery, and the gateway need not know when the
condition will clear.

The message MAY carry any combination of the Expected Time Until Link
Usability, Generation Time, and Expected Link Delay objects
({{objects}}), subject to {{applicability-objects}}. A message
carrying none of them is valid and meaningful: it reports the
condition, and the objects add prediction where the gateway has it.
The message MUST NOT carry an Admission Denial object.

## Denied by Link Policy {#dlp}

Denied by Link Policy is code TBD3 under ICMPv6 Destination
Unreachable and code TBD4 under ICMPv4 Destination Unreachable. It
reports that the invoking packet was not forwarded because the
admission policy governing the constrained link required to reach its
destination does not admit it, independent of the link's current
usability.

The outcome resembles that of the existing administratively prohibited
codes, and those codes are capable of representing the broad outcome.
This experimental code differs in identifying refusal by the admission
policy of a specific constrained link, and in carrying a structured
reason. Whether that distinction warrants a permanent distinct code is
not established by this document; it is left for experimentation and
IETF review ({{experiment}}).

The message SHOULD include exactly one Admission Denial Object
({{ado}}). A message without one reports the denial with no reason,
which a gateway MAY choose where policy forbids disclosure
({{generation}}). The message MUST NOT carry Expected Time Until Link
Usability or Expected Link Delay, and SHOULD NOT carry Generation Time
({{applicability-objects}}).

## Extension Objects {#objects}

The objects below share a single ICMP Extension Object class, the Link
Condition Object class (Class-Num TBD5), distinguished by C-Type. The
object header is as defined in {{Section 8 of RFC4884}}. No object is
mandatory. Which objects a message may carry depends on the
Destination Unreachable code it reports ({{applicability-objects}});
within that set, each object is optional, presence conveys
availability, and absence means the gateway did not supply that
information. No sentinel values are defined for absent data.

The two duration objects, Expected Time Until Link Usability and
Expected Link Delay, are defined and encoded independently of the
Generation Time object. Their interpretation does not depend on the
absolute timestamp representation, so that representation could later
be revised or superseded without changing them.

#### Duration Format {#durations}

Both duration objects carry an unsigned 32-bit integer count of
milliseconds. Millisecond precision is not intrinsically necessary for
every value of Expected Time Until Link Usability, but a single
representation for both duration-valued objects reduces
implementation complexity and the risk of unit-conversion errors.
{{RFC4884}} defines the extension object framework but no duration
datatype; the representation is defined here.

The maximum representable duration, 0xFFFFFFFF milliseconds, is
4,294,967.295 seconds, approximately 49.7 days. It is a valid finite
value, not a sentinel. If an otherwise available estimate exceeds the
maximum representable duration, the corresponding object MUST NOT be
included. No value is reserved to mean infinite, unknown, or greater
than the maximum, and no saturation is defined; absence of an object
means only that no representable estimate was supplied, and the
protocol does not distinguish an unknown estimate from one that is
known but not representable.

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
generated the message, as a 64-bit NTP timestamp per
{{Section 6 of RFC5905}}: 32 bits of seconds since the NTP epoch of 1
January 1900 UTC and 32 bits of fraction, with resolution 2^-32
seconds, wrapping every 2^32 seconds. Following the template of
{{RFC8877}}, no era number is carried; a receiver interpreting the
value absolutely resolves the era as the one placing it nearest its
own current time.

Generation Time identifies when the notification and any estimates in
it were generated. It is principally intended to allow a receiver that
can relate it to its local time reference to account for the time
elapsed since an accompanying ETU estimate was generated ({{etu}}). A
receiver that cannot relate it to its local time reference ignores
it; nothing else in the message depends on it, and it remains
independently parseable. In the absence of ETU this document defines
no use for Generation Time, so a Link Temporarily Unavailable message
SHOULD NOT include Generation Time unless it also includes ETU
({{applicability-objects}}).

### Expected Time Until Link Usability {#etu}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Length = 8           |  Class-Num =  |  C-Type = 2   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Expected Time Until Link Usability               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-etu title="Expected Time Until Link Usability Object"}

Expected Time Until Link Usability (ETU) is the gateway's estimate, at
the moment it generated the notification, of the interval from
generation until the constrained link is expected to become usable for
forwarding traffic. It is an unsigned 32-bit integer count of
milliseconds, in the format of {{durations}}. ETU refers to expected
data-plane usability, not to the start of a scheduled contact or of
preparation for one. Preparatory operations necessary to make the
link usable, such as antenna pointing, modem configuration,
acquisition, synchronization, or analogous link-establishment
operations, are expected to have completed by the end of the reported
interval and are not represented separately.

ETU is measured from notification generation, not from receipt. It is
an estimate, not a guarantee. It is not a retry interval and not an
instruction to transmit at any particular time. It does not include
Expected Link Delay, which describes traversal of the link once
usable.

ETU is meaningful on its own. A receiver that can relate a supplied
Generation Time to its local time reference may compute the expected
absolute usability time as Generation Time + ETU, and may thereby
account for the time the notification spent in delivery; any NTP era
or wraparound considerations apply only to that absolute
interpretation. A receiver without Generation Time, or unable to
relate it to local time, may apply ETU from the moment of receipt.
Doing so is conservative when delivery of the notification was
delayed, since the true remaining interval can only be shorter. This
document does not assume that ICMP delivery is prompt.

The finite range of {{durations}} provides a practical horizon for
this experimental network-layer signal. Unavailability on longer
timescales is not represented by ETU; such timescales may instead
bear on broader application, storage, provisioning, or
communication-mechanism decisions, which this document does not
address.

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
forwarding, as an unsigned 32-bit integer count of milliseconds in
the format of {{durations}}. It is an estimate, not a bound; a
quantity of the link, not an end-to-end path delay; and independent of
ETU and of Generation Time. This document does not require receivers
to combine it with other metadata or otherwise compute an end-to-end
delivery time; how the estimate is used is a matter of endpoint and
application policy.

For perspective only, the maximum representable duration, if it were
pure propagation delay at the speed of light in vacuum, would
correspond to a distance of approximately 1.29 x 10^12 km, roughly
1.29 trillion km. The range is therefore very large even for the
long-delay environments that motivate this document. This observation
is explanatory; Expected Link Delay remains an estimated one-way link
delay comprising whatever components contribute to it.

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

## Object Applicability by Code {#applicability-objects}

Each Link Condition Object is applicable to one of the two codes, as
summarized in {{tab-applicability}}.

| Object | Link Temporarily Unavailable | Denied by Link Policy |
|:-------|:-----------------------------|:----------------------|
| Expected Time Until Link Usability | MAY | MUST NOT |
| Generation Time | MAY; SHOULD NOT without ETU | SHOULD NOT |
| Expected Link Delay | MAY | MUST NOT |
| Admission Denial | MUST NOT | SHOULD |
{: #tab-applicability title="Link Condition Object Applicability by Destination Unreachable Code"}

Expected Time Until Link Usability and Expected Link Delay reveal
characteristics of a constrained link to which the invoking traffic
was not admitted, and admission policy is intentionally evaluated
before any link-state information is disclosed ({{generation}}); a
Denied by Link Policy message therefore MUST NOT carry either. It
SHOULD NOT carry Generation Time, for which this document defines no
use without ETU. Admission Denial describes a policy refusal and MUST
NOT be sent with Link Temporarily Unavailable; its inclusion with
Denied by Link Policy remains subject to the disclosure controls of
{{generation}}.

A Link Temporarily Unavailable message SHOULD NOT include Generation
Time unless it also includes Expected Time Until Link Usability, since
this document defines no other use for it and unnecessary
absolute-time information should not normally be disclosed. This is
not a prohibition; experimentation or later experience may identify
another legitimate use.

These are sender restrictions. A receiver MUST ignore a Link Condition
Object that is not applicable to the received code rather than
rejecting the message ({{common}}); this tolerance exists for
robustness and extensibility and does not relax the sender rules.
Unknown C-Types continue to follow the extension processing rules of
{{RFC4884}}.


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
   send Denied by Link Policy, without Expected Time Until Link
   Usability or Expected Link Delay ({{applicability-objects}}).

4. If the traffic is admitted but the required link is presently
   unusable under a transient condition known to the domain, drop it
   and send Link Temporarily Unavailable, optionally including
   Expected Time Until Link Usability, Generation Time, and Expected
   Link Delay ({{applicability-objects}}).

5. If the traffic is admitted and the link is usable, forward it
   normally. A forwarded packet MUST NOT elicit either code.

A gateway MUST NOT generate either code if another path within the
signaling domain can provide service inconsistent with the signal.
The codes describe the domain's aggregate ability to serve the traffic
toward the destination ({{applicability}}). A gateway that cannot
determine this property for a destination MUST NOT generate these
codes for it.

A gateway MUST NOT generate Link Temporarily Unavailable on the basis
of observed local interface or carrier state alone. The transient
condition it reports MUST derive from the domain's knowledge of the
constrained link's availability ({{applicability}}).

A gateway MUST NOT send an ETU value not derived from its current
knowledge of the link. A gateway that does not know when the link is
expected to become usable omits the ETU object. A gateway MAY send ETU
without Generation Time; it SHOULD NOT send Generation Time without
ETU ({{applicability-objects}}).

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

The absence of these codes promises nothing. ICMP delivery is
unreliable and rate limited, and a gateway MUST NOT assume that a
sender received any particular notification; it generates a fresh
notification, subject to rate limiting, for each invoking packet that
meets the conditions above.


# Receiver Processing {#receiver}

## Validation {#validation}

A receiver MUST validate these messages as ICMP errors per
{{RFC4443}} and the mitigations of {{RFC5927}}: the invoking packet
excerpt is matched against existing connection or flow state, and a
message matching nothing the receiver sent is discarded. A receiver
SHOULD discard these messages when received on an interface facing
outside its administered domain.

## Consumer Behavior {#consumers}

This section is informative. This document does not specify how a
host, transport, application, or management system reacts to the
codes, and no such reaction is required for interoperability. Receipt
of Link Temporarily Unavailable informs the receiver that the domain
models the path as transiently unavailable and, where ETU is present,
roughly when usability is expected; receipt of Denied by Link Policy
informs it that the traffic is not admitted and, where a reason is
present, on what dimension. What follows are possibilities that the
signals enable and that the experiment of {{experiment}} is intended
to explore.

A host might record the information to avoid immediately repeating
attempts known to have failed. Because the signal already reflects
every path the domain could offer, immediate repetition is likely to
elicit the same error. How such state would be keyed, how widely it
would apply, how long it would live, how it would be invalidated when
the domain's knowledge changes, and how it would interact with routing
are all open questions. Because ETU is an estimate and the gateway
cannot revoke a notification, any such state is a hint about the
domain's knowledge at generation time and not an assurance of service
at any later time.

A transport might use the information as an input to its timeout and
failure decisions: for example, comparing the expected wait against a
connection's tolerance, or treating Denied by Link Policy as a
condition that retrying unchanged will not resolve. The remote
endpoint receives no corresponding notification and continues to run
its own timers, which bounds the usefulness of waiting. No TCP or QUIC
behavior is defined here.

An application might be given the condition and its metadata so that
it can defer, fail fast, select another communication mechanism such
as a bundle service, or trigger provisioning. Whether such an
interface should be standardized is not addressed here.

Stronger reactions to unauthenticated ICMP information carry greater
risk ({{security}}); which reactions are worth their risk is itself an
experiment question.


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
abort is a reasonable outcome for unauthorized traffic, soft-error
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
most of the burst. Because ICMP is unreliable and a gateway cannot
know which notifications were received, operators SHOULD size gateway
ICMP rate limits for the expected flow fan-out at gap onset. Hosts
that act on the signal to avoid repeated attempts reduce the offered
load that would otherwise sustain the burst.

ETU is only as meaningful as the gateway's knowledge of the link.
Absolute interpretation of Generation Time additionally depends on the
receiver being able to relate it to its local time reference;
receivers that cannot may apply ETU from receipt ({{etu}}).

Hosts that maintain state derived from these codes SHOULD expose it to
local diagnostics, since such state changes transmission behavior in
ways otherwise invisible to troubleshooting.


# Experimental Status and Goals {#experiment}

This document is published as Experimental. The codes and metadata
are specified normatively so that independent implementations
interoperate at the ICMP layer. The value of the
mechanism, however, rests on two kinds of questions that only
deployment can answer: whether gateways can generate the signals
correctly and usefully, and whether consumers of the signals can do
anything worthwhile with them. Consumer behaviors are deliberately not
standardized here and are not prerequisites for interoperability; the
experiment is intended to produce the experience on which later
specification of such behaviors might be based.

## Scope of the Experiment {#exp-scope}

The experiment runs within administered limited domains meeting the
criteria of {{applicability}}: space networking testbeds and missions,
other networks with scheduled or intermittently available managed
links, and laboratory emulations of the same topologies. The codes are
not exchanged across uncontrolled networks, and no behavior in this
document affects hosts whose traffic never requires a trans-boundary
link.

Prior to IANA assignment, implementations use the experimental ICMP
values reserved by {{RFC4727}} and coordinate their interpretation
bilaterally, per that document's rules. Interoperability reports from
this phase are in scope for the experiment.

## Network Signal Evaluation {#exp-signal}

The first layer of the experiment concerns the signals themselves:

- Generation correctness: whether gateways generate each code only
  under the conditions of {{generation}}, and in particular whether
  the aggregate-domain determination can be made reliably in deployed
  routing architectures.

- Distinction: whether the three-way distinction among no-route,
  transient constrained-link unavailability, and policy denial
  ({{interaction}}) is maintained in practice and is useful to
  operators and senders.

- Prediction accuracy and usefulness: how closely ETU tracks actual
  link usability, how often it is available at all, and whether
  Generation Time and Expected Link Delay are populated and used.

- Generation rate: gateway ICMP generation rates at gap onset under
  realistic fan-out, and the effect of rate limiting on which senders
  receive a notification.

- Legacy and middlebox behavior: how unmodified hosts respond to the
  new codes across the stacks present in target environments,
  including embedded and flight-heritage implementations, and whether
  any hard-error handling of Link Temporarily Unavailable is harmful
  ({{legacy}}); and what filtering changes deployments required.

- Security and disclosure: whether the information disclosed by
  Denied by Link Policy reasons and by predictive metadata is
  operationally acceptable, whether the disclosure controls of
  {{generation}} are sufficient, and whether forged or stale
  notifications caused harm.

- Whether Denied by Link Policy warrants a permanent distinct code, or
  whether an existing administratively prohibited code with the
  Admission Denial Object would serve.

## Consumer Behavior Exploration {#exp-questions}

The second layer concerns what receivers do with the signals. None of
these behaviors is specified here; the experiment is intended to
discover which are worthwhile:

- Whether hosts that record the information to avoid repeated
  attempts reduce futile retransmission and offered load during
  link-unavailable periods, and what keying, scope, lifetime, and
  invalidation rules work.

- What transport reactions are appropriate, including whether
  comparing expected wait against connection tolerance is useful and
  whether connections that wait survive the peer's own timers.

- What application decisions the information enables, including
  selection among communication mechanisms such as IP transports and
  bundle services, and whether an application interface should be
  standardized.

- Whether provisioning and orchestration systems can use the signals,
  and whether divergence telemetry ({{uc-divergence}}) exposed real
  faults ahead of existing monitoring.

## Evidence for Advancement {#exp-criteria}

Later consideration of the mechanism for the standards track could be
informed by evidence of: at least two independent interoperable
implementations of both codes, including extension object processing;
generation accuracy in at least one operational, non-laboratory
domain; usefulness of the metadata objects, with unused objects
removed rather than carried forward; manageable legacy and middlebox
behavior; acceptable security and disclosure properties; consumer
behaviors independently demonstrated to benefit from the signals,
without presupposing any one receiver strategy; and evidence that the
semantics apply across more than one class of constrained network
without special-case changes to the wire format. Reports of failed or
inconclusive experiments are requested to the same degree as
successful ones.


# Security Considerations {#security}

ICMP carries no authentication, and both codes are actionable by
implementations that choose to act on them, so the principal threats
are forgery by off-path attackers and information disclosure to the
senders these messages answer. The containment for both is the
limited-domain scoping of {{applicability}}: validation against sender
state ({{validation}}), boundary filtering ({{generation}}), rate
limiting, and a single administrative authority. None of these
defenses is dependable across the open internet.

A forged Link Temporarily Unavailable message could induce an
implementation that acts on it to withhold trans-boundary traffic for
the advertised interval. This document does not require receivers to
withhold traffic, and stronger reactions to unauthenticated ICMP
information carry correspondingly greater risk; an implementation that
does act on the signal should bound the effect of any single
notification, treat ETU as an estimate that may be stale or wrong, and
consider that the gateway cannot revoke a notification whose basis has
changed. The invoking-packet validation of {{RFC5927}} forces an
attacker to guess connection state, and boundary filtering confines
injection to on-path or in-domain attackers.

A forged Denied by Link Policy message is in the class of attacks
{{RFC5927}} analyzes for existing hard-error codes; it adds no
capability beyond forging the existing administratively prohibited
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
they protect the underlying schedule. Evaluating admission before link
state ({{generation}}) prevents disclosure of schedule information to
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
Generation Time; C-Type 2, Expected Time Until Link Usability; C-Type
3, Expected Link Delay; and C-Type 4, Admission Denial, assigned by
this document. Further C-Types are Specification Required.

A new registry, "Admission Denial Reason Codes," initialized per
{{ado}}, registration policy Specification Required.

Publication as Experimental does not by itself satisfy the
registration procedures for the Destination Unreachable code
registries. Should the requested code assignments not be approved,
the allocation strategy is a matter for further review.


--- back

# Worked Example {#example}

A gateway whose deep-space link is presently unusable receives a
packet for a trans-boundary destination at 18:00:00 UTC on 16
September 2026. No other path in the signaling domain can serve the
traffic. The packet is admissible under the link's policy, but the
domain's contact plan shows the link becoming usable 42 minutes later,
and the gateway's one-way delay estimate for the link is 20 minutes.
The gateway drops the packet and returns Destination Unreachable, code
Link Temporarily Unavailable, with three extension objects. The
Class-Num octet of each is TBD5 and is omitted from the listing.

| Object | C-Type | Wire value | Decoded |
|:-------|-------:|:-----------|:--------|
| Generation Time | 1 | 0xEE5557A0 0x00000000 | 2026-09-16 18:00:00Z |
| Expected Time Until Link Usability | 2 | 0x002673C0 | 2520000 ms (42 min) |
| Expected Link Delay | 3 | 0x00124F80 | 1200000 ms (20 min) |
{: #tab-example title="Worked Example Extension Objects"}

The Generation Time seconds word is the Unix time of the instant plus
2208988800; the fraction word is zero.

A receiver that cannot relate Generation Time to its own clock, or
that did not receive it, needs no timestamp arithmetic: it knows that,
as of generation, the link was expected to become usable in 42
minutes, and it may apply that interval from the moment of receipt. If
the notification spent time in delivery, the receiver waits slightly
longer than necessary, which errs on the safe side.

A receiver that can relate Generation Time to its local clock may
compute Generation Time + ETU = 18:42:00Z and, on receiving the
message at, say, 18:00:03Z, understand that about 41 minutes 57
seconds remain.

Either receiver might, as a matter of local policy, use the 20-minute
link delay estimate in deciding whether the destination suits its
traffic once the link is usable. Whether either receiver does anything
at all with the information is not specified by this document.


# Acknowledgments
{:numbered="false"}

The author used AI-assisted tooling in preparing this document,
including drafting and editing text and converting it to the
kramdown-rfc source format. All technical content, design decisions,
and the final text are the responsibility of the author.
