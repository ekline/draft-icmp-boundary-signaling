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
  RFC1812:
  RFC4443:
  RFC4727:
  RFC4884:
  RFC5905:

informative:
  RFC1122:
  RFC4890:
  RFC4950:
  RFC5461:
  RFC5837:
  RFC5927:
  RFC6069:
  RFC8799:
  RFC8877:
  RFC8883:
  RFC9171:
  RFC9293:

...

--- abstract

This document defines optional ICMP extension objects that a gateway at
the boundary between a low-delay IP network and a constrained link,
such as a scheduled deep-space link, can attach to existing Destination
Unreachable messages when an otherwise admissible packet is not
forwarded because the required link is presently unusable under a
transient condition known to the domain. The objects convey the
expected time until the link becomes usable, the time the estimate was
generated, and the expected one-way delay of the link; the underlying
Destination Unreachable code retains its existing meaning. This
document also provisionally defines one new Destination Unreachable
code, Denied by Link Policy, reporting that a packet was not forwarded
because the link's admission policy does not admit it. This document
specifies the conditions under which the objects and the code are
generated and the information they convey. It does not specify how
transports, applications, or hosts react to them; making the
boundary's knowledge available to senders is intended to enable
experimentation with such reactions.


--- middle

# Introduction {#intro}

In networks that include constrained links -- deep-space links,
satellite links available only during scheduled passes, periodic
store-and-forward contacts -- an IP sender has no visibility into the
condition of the path beyond its own timers. The router at the boundary
between the sender's low-delay network and the constrained link knows
more: it knows, from a contact plan or equivalent domain knowledge,
when the link is expected to carry traffic, and it enforces an
admission policy describing whose traffic the link will carry. Today
that knowledge shapes forwarding at the boundary but is not
communicated to senders, whose transports retransmit into unavailable
or unauthorized links until timers expire and whose applications
receive a failure indistinguishable from a routing fault or a crashed
peer.

Existing Destination Unreachable messages can report forwarding
failures that are transient. This document defines optional ICMP
extension objects {{RFC4884}} that allow a gateway in a managed
constrained-link domain to supply additional information about such a
failure: the time at which it generated the message, its estimate of
the time until the link becomes usable, and the approximate one-way
delay of the link. The underlying Destination Unreachable code retains
its existing meaning. When the objects are absent, or a receiver does
not process them, the message provides the ordinary Destination
Unreachable indication; it does not separately identify the
constrained-link condition described here.

This document also provisionally defines one new Destination
Unreachable code, for both ICMPv4 and ICMPv6. Denied by Link Policy
reports that a packet was not forwarded because the link's admission
policy does not admit it. Neither the objects nor the code describe
what traffic would be admitted, and neither reports on a packet that
was forwarded.

This document specifies the network-layer signals: when a gateway
attaches the objects or generates the code, what each means, and the
semantics and encoding of the metadata. It does not specify how a
transport, application, host cache, provisioning system, or
orchestrator reacts to them. Such reactions are not required for
interoperability, and they are not merely out of scope: a purpose of
publishing the signals is to enable experimentation with them. A host
might defer further attempts; a transport might use the information as
an input to its timeout and failure decisions; an application might
expose the condition, select another communication mechanism such as a
Bundle Protocol {{RFC9171}} agent for store-and-forward delivery, or
trigger provisioning. Experience with such reactions may motivate
later specifications ({{experiment}}).

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

The objects and the code are specified for use within administered
limited domains {{RFC8799}}. {{experiment}} defines the experiment
this document proposes.


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
  availability and admission policy and generating the metadata and
  code defined in this document.

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

Link Condition Object:
: an ICMP extension object of the class defined in {{objects}},
  carrying constrained-link metadata on a Destination Unreachable
  message.

Carrier code:
: an existing Destination Unreachable code on which Link Condition
  Objects may be carried ({{carriers}}).

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
generating these signals, D the duration of the unavailability being
reported, and T the timescale on which the sender's transport would
otherwise detect failure through retransmission timer growth and user
timeout. The metadata is useful when R is small relative to T, so the
notification arrives while the sender still has options, and R is
small relative to D, so the reported condition is still in effect when
the notification arrives. In the Mars enclave of {{conops}}, R is
milliseconds to seconds, T is seconds to minutes, and D is minutes to
hours.

When R approaches D, a notification describes a link state that may no
longer exist on arrival. Gateways therefore attach the metadata and
generate Denied by Link Policy only into their local low-delay domain
and never across the constrained link: an Earth-side gateway signals
terrestrial senders, a Mars-side gateway signals enclave hosts, and
neither signals across the link between them. "Local" is consequently
relative, not absolute. The entire terrestrial internet, at sub-second
internal round-trip times, is a single low-delay domain relative to a
link whose unavailability is measured in minutes, and a vehicle-area
network is one relative to a SATCOM link whose availability is
measured in scheduled windows of tens of minutes.

The metadata applies only to links whose availability the domain
deliberately models as a constrained resource: the gateway asserts a
transient service condition on the basis of domain or service
knowledge about the link, not on the basis of observed local carrier
or interface state. A router that merely notices its outgoing
interface is down generates whatever Destination Unreachable error
ordinary ICMP rules call for, but has no basis to attach Link
Condition Objects to it; ordinary transient link failures are not the
condition this document addresses. How the domain acquires its
knowledge -- a contact plan, orchestration, link-management
signaling, or other domain-specific means -- is not specified; the
objects encode what the gateway is entitled to assert, not how it
learned it.

The metadata and Denied by Link Policy are further applicable only
where the signaling domain can determine that no path within it can
serve the invoking traffic toward its destination. They describe the
domain's aggregate ability to serve the traffic, not the local state
of the gateway that happened to receive the packet; the signal is
meaningful only when receiving it tells the sender something true
about every path the domain could offer. This restriction applies to
the metadata and to Denied by Link Policy; it does not alter when a
router generates ordinary Destination Unreachable errors. The
mechanism is not applicable to arbitrary routing architectures in
which a gateway cannot make this determination ({{generation}}).

The same criteria identify deployments beyond the interplanetary case,
wherever a domain manages a scheduled or intermittently available
link on the basis of advance knowledge. Conversely, the mechanism
offers nothing on paths that are merely lossy or congested; existing
congestion signaling and routing convergence address those conditions.

The metadata and Denied by Link Policy are specified for use within
limited domains {{RFC8799}} under a single administrative authority.
ICMP carries no authentication, so forged notifications are contained
only by validation against sender state and filtering at domain
boundaries, and the Denied by Link Policy code discloses admission
policy that is diagnostic within a cooperative domain but
reconnaissance outside it ({{security}}). Boundary routers discard
Denied by Link Policy on ingress from and egress to uncontrolled
networks. The carrier codes are ordinary Destination Unreachable
codes whose filtering remains a matter of existing operational
policy; see {{validation}} regarding the metadata they may carry.


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
same signals toward terrestrial senders. This section describes the
Mars side for concreteness.

## Roles {#roles}

Enclave hosts originate and receive application traffic. Some are
DTN-aware, running a BP agent or a stack extension that can hand
traffic to one; others are unmodified IP hosts, including legacy
instruments and commercial payloads whose stacks predate this
specification. Where a performance-enhancing proxy (PEP) terminates
transport sessions on behalf of such hosts, the PEP can consume these
signals for them, so that the mechanism's benefit need not depend on
universal host adoption.

Gateways hold the contact plan for their deep-space links, enforce
admission and precedence policy on contact capacity, and generate the
signals defined in this document: a Destination Unreachable error
carrying Link Condition Objects when admitted trans-boundary traffic
arrives while the link is not usable, and Denied by Link Policy when
traffic arrives that policy does not admit. In this deployment,
enclave routing is driven by the same contact plan the gateways hold,
so traffic reaches a gateway that cannot serve it only when no gateway
can; this is what makes the aggregate-domain invariant of
{{applicability}} hold. Because the metadata already accounts for
every path the domain could offer, immediately retrying the same
traffic is likely to be futile.

An orchestration function, which may be Earth-based, distributes
contact plans and admission policy to gateways in advance. Since
distribution itself crosses the deep-space link, gateways operate
autonomously between management contacts, and their local plan and
policy are authoritative for their links.

Intra-enclave traffic is unaffected. The signals are generated only
for traffic that requires a trans-boundary link, and hosts that never
originate such traffic need not implement them.

## Use Cases {#use-cases}

### Traffic Sent While the Link Is Unusable {#uc-deferral}

A surface host begins a bulk transfer toward Earth between contacts.
The first packet reaches the gateway, which drops it and returns a
Destination Unreachable error carrying an Expected Time Until Link
Usability object. From the object, the host knows that the domain
regards the path as transiently unavailable and roughly how long that
is expected to last; from the bare error alone it would know only that
the packet was not delivered. It might use the metadata to avoid
repeated attempts that would elicit the same error, and to tell the
application that the path is deferred rather than failed. Whether and
how a host does so is an experiment question; ICMP is unreliable, and
the gateway cannot assume that any particular notification was
received.

### Selection of Another Communication Mechanism {#uc-handoff}

A DTN-aware host receiving an unreachable error with Link Condition
Objects might, as a matter of local policy, hand the affected traffic
to its local BP agent, which originates bundles toward the
destination; these queue at the gateway and are forwarded during the
next contact. An application with its own delay tolerance might
instead compare the expected wait against that tolerance and fail
fast. The metadata does not prescribe either response; it supplies the
information on which the endpoint decides.

### Traffic Denied by Link Policy {#uc-denied}

A misconfigured or newly integrated payload host sends best-effort
traffic toward Earth during a contact whose capacity is allocated to
command and telemetry classes. The gateway classifies the traffic
against its local policy, which may consider DSCP, addresses, protocol
and port, or any other criteria, and returns Denied by Link Policy.
The code is diagnostic, not prescriptive: it reports only that the
link's admission policy refused the traffic, not why or what traffic
would be admitted, since admission policy is multidimensional and
local, and coaching senders on acceptable markings would invert an
admission model in which access is granted through the control plane.
The host learns that the traffic was refused by link policy rather
than lost to a routing fault; the operator debugging it consults the
gateway's own logs and counters ({{operational}}). Obtaining admission
is a provisioning action.

A flow that policy admitted at contact start and later displaces in
favor of higher-precedence traffic falls under the same code. Whether
an implementation regards this internally as preemption, resource
allocation, or precedence enforcement is a matter of local policy;
the sender learns only that the traffic is not presently admitted.

### Plan/Reality Divergence {#uc-divergence}

In nominal operation, provisioned traffic should never elicit these
signals. Metadata-bearing unreachable errors generated during a
planned contact, or Denied by Link Policy generated for a flow the
orchestrator believes it admitted, indicate that the control plane's
model and the data plane's state have diverged: an unplanned outage, a
policy distribution failure, or traffic outside the provisioning
workflow. Gateways count and report generation of these signals so
that operators can use them as a diagnostic feed ({{operational}}).

## Interaction of the Two Signals {#interaction}

The two signals answer different questions and may both apply to one
flow: a link can be simultaneously restricted and unavailable.
Admission is evaluated first. Traffic that would be refused regardless
of link state receives Denied by Link Policy, which avoids disclosing
link schedule information to traffic that could not use the link
anyway; traffic that is admitted but whose link is not presently
usable receives the ordinary Destination Unreachable error, to which
the gateway may attach Link Condition Objects. A sender whose denial
is resolved through provisioning may then receive a metadata-bearing
unreachable error for the same flow: first become admissible, then
wait for the link.

Ordinary Destination Unreachable errors do not by themselves establish
that service is permanently absent; {{Section 3.2.2.1 of RFC1122}}
notes that they can result from routing transients, and {{RFC6069}}
has previously used them experimentally as indications of connectivity
disruption. This document does not change their meaning. Link
Condition Objects add to such an error the domain's assertion that the
failure reflects a modeled transient condition of a constrained link
to which the traffic was admitted, and, where supplied, an estimate of
its duration. Denied by Link Policy means that the link may exist and
be usable, but this traffic is not admitted to it.


# Message Formats and Extension Objects {#formats}

This section is normative. The Denied by Link Policy code and the Link
Condition Object class are shown as TBD pending IANA assignment
({{iana}}); prior to assignment, implementations use the experimental
values of {{RFC4727}} for those as described in {{exp-scope}}. The
carrier codes of {{carriers}} are already assigned.

## Common Message Format {#common}

Both signals are carried in Destination Unreachable messages: ICMPv6
Type 1 {{RFC4443}} and ICMPv4 Type 3 {{RFC0792}}. {{RFC4884}} already
permits an ICMP Extension Structure on the existing Destination
Unreachable message types of both IP versions; no new type or code is
needed to carry the objects defined here. The message layout is the
extended Destination Unreachable message of {{RFC4884}}; this document
adds no fields. The Length field, original-datagram truncation and
padding, extension header, and checksum rules follow
{{Section 4 of RFC4884}} and {{Section 7 of RFC4884}}. Attaching
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

A message that carries no objects defined by this document need not
carry an extension structure at all; it is then an ordinary
Destination Unreachable message. An extension structure, when
present, may also carry other independently applicable {{RFC4884}}
objects.

Recognition of a carrier code implies nothing about extension support.
Receivers that do not process {{RFC4884}} extensions process the
message as the ordinary error its code denotes; the extension
structure lies beyond the portion of the original datagram such
receivers examine.

A receiver that processes extensions MUST ignore Link Condition
Objects with an unrecognized C-Type, and MUST process at most one
object of each C-Type per message, taking the first if several are
present. A receiver MUST ignore a Link Condition Object that is not
applicable to the received Destination Unreachable code
({{applicability-objects}}). An unrecognized or inapplicable Link
Condition Object does not by itself invalidate the underlying ICMP
error; this tolerance does not extend to malformed extension framing
or an invalid extension checksum, which are handled per {{RFC4884}}.

## Constrained-Link Metadata on Destination Unreachable Errors {#ltu}

When a gateway does not forward an otherwise admissible packet because
the constrained link required to reach its destination is presently
unusable, it generates whatever Destination Unreachable error the
ordinary rules of {{RFC1812}} or {{RFC4443}} call for, according to
its own forwarding state. This document does not change which code is
generated or when. It permits the gateway to attach Link Condition
Objects to that error, subject to the conditions in this section, so
that the sender can learn what the bare error does not convey.

A gateway MAY attach Link Condition Objects to a Destination
Unreachable message only when all of the following hold:

- the signaling domain determined that the invoking packet would
  otherwise be served using a constrained link;

- no alternative path in the signaling domain can provide service
  inconsistent with the metadata;

- the traffic is otherwise admitted to that link;

- the required constrained link is presently unusable;

- the domain represents that unusability as a transient service
  condition, on the basis of its knowledge of the link, rather than as
  absence of a route or of service;

- the invoking packet was not forwarded; and

- the message's code is an eligible carrier code ({{carriers}}).

A gateway that meets these conditions but has no estimate to supply,
or withholds estimates by policy, sends the ordinary error without
Link Condition Objects. The gateway need not know when the condition
will clear.

None of these assertions attaches to a bare Destination Unreachable
error. A message without Link Condition Objects means only what its
code has always meant; it does not imply that admission succeeded,
that a managed constrained link caused the failure, or that the domain
predicts recovery. The presence of an Expected Time Until Link
Usability or Expected Link Delay object supplied under this
specification carries the constrained-link meaning defined for that
object in {{objects}}. A Generation Time object supplies a timestamp
and is not by itself evidence of any particular cause of failure.

### Eligible Carrier Codes {#carriers}

Link Condition Objects are applicable only to the Destination
Unreachable codes listed in this section. They MUST NOT be attached to
other Destination Unreachable codes or to other ICMP error types. The
code is selected by existing ICMP rules according to the gateway's
forwarding state; this document neither alters that selection nor
redefines any code.

For ICMPv6, per {{Section 3.1 of RFC4443}}:

| Type 1 Code | Name | Eligibility |
|------------:|:-----|:------------|
| 0 | No route to destination | Eligible when the forwarding table lacks a matching entry for the destination and the gateway independently holds the constrained-service knowledge required by {{ltu}}. |
| 3 | Address unreachable | Eligible when the gateway retains a route whose required link is unusable and no more specific code applies. {{RFC4443}} names a link-specific problem as an example of this code. |
{: #tab-carriers-v6 title="Eligible ICMPv6 Carrier Codes"}

For ICMPv4, per {{Section 4.3.3.1 of RFC1812}} and
{{Section 5.2.7.1 of RFC1812}}:

| Type 3 Code | Name | Eligibility |
|------------:|:-----|:------------|
| 0 | Network unreachable | Eligible when ordinary forwarding failure warrants Network Unreachable, such as absence of any route to the destination network. |
| 1 | Host unreachable | Eligible when ordinary forwarding failure warrants Host Unreachable; see the note below. |
{: #tab-carriers-v4 title="Eligible ICMPv4 Carrier Codes"}

The existence of a managed service and the presence of a currently
usable forwarding-table entry are distinct. A domain may withdraw the
route to a destination between contacts while retaining knowledge of
the future service, yielding No Route or Network Unreachable; or it
may retain a route toward a presently unusable link, yielding Address
Unreachable. Both are legitimate carriers. This document does not
require a particular routing implementation, and a gateway that would
have sent a given code before implementing this specification does not
change that code because of it.

Editor's note (IPv4 applicability, to be resolved before publication):
{{RFC1812}}'s code-specific text ties Host Unreachable to a
destination on a directly connected network, while its general
Destination Unreachable text covers an unreachable next hop.
Implementations are known to send Host Unreachable on transit-link
failure, but that is precedent rather than mandate, and ICMPv4 has no
residual code equivalent to ICMPv6 Address Unreachable. The
retained-route, unusable-transit-link case therefore requires an
explicit IPv4 applicability clarification that this document does not
yet supply. The worked example in {{example}} uses ICMPv6 so that its
interpretation does not depend on resolving this point.

## Denied by Link Policy {#dlp}

Denied by Link Policy is code TBD3 under ICMPv6 Destination
Unreachable and code TBD4 under ICMPv4 Destination Unreachable. It
reports that the invoking packet was not forwarded because the
admission policy governing the constrained link required to reach its
destination does not admit it, independent of the link's current
usability.

The code conveys only this coarse condition. It does not identify the
policy dimension, rule, precedence threshold, allocation state, or
other detail on which admission failed, and it does not describe what
traffic would be admitted.

The outcome resembles that of the existing administratively prohibited
codes, and those codes are capable of representing the broad outcome.
This experimental code differs in identifying refusal by the admission
policy of a specific constrained link. Whether that distinction
warrants a permanent distinct code is not established by this
document; it is left for experimentation and IETF review
({{experiment}}).

The message uses the extension-capable format of {{common}}, but this
document defines no extension objects applicable to it
({{applicability-objects}}). Future specifications may define
extension objects for Denied by Link Policy if operational experience
identifies information that is both useful to receivers and
appropriate to disclose.

## Extension Objects {#objects}

The objects below share a single ICMP Extension Object class, the Link
Condition Object class (Class-Num TBD5), distinguished by C-Type. The
object header is as defined in {{Section 8 of RFC4884}}. No object is
mandatory. Which objects a message may carry depends on the
Destination Unreachable code it reports ({{applicability-objects}});
within that set, each object is optional, presence conveys
availability, and absence means the gateway did not supply that
information. No sentinel values are defined for absent data. A
message carrying none of these objects has only the ordinary meaning
of its code ({{ltu}}).

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
no use for Generation Time, so a message SHOULD NOT include Generation
Time unless it also includes ETU ({{applicability-objects}}).

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

## Object Applicability by Code {#applicability-objects}

The Link Condition Objects defined in this document are applicable
only to the eligible carrier codes of {{carriers}}, as summarized in
{{tab-applicability}}. This document defines no Link Condition Object
applicable to Denied by Link Policy.

| Object | Eligible carrier codes | Denied by Link Policy |
|:-------|:-----------------------|:----------------------|
| Expected Time Until Link Usability | MAY | MUST NOT |
| Generation Time | MAY; SHOULD NOT without ETU | SHOULD NOT |
| Expected Link Delay | MAY | MUST NOT |
{: #tab-applicability title="Link Condition Object Applicability by Destination Unreachable Code"}

Expected Time Until Link Usability and Expected Link Delay reveal
characteristics of a constrained link to which the invoking traffic
was not admitted, and admission policy is intentionally evaluated
before any link-state information is disclosed ({{generation}}); a
Denied by Link Policy message therefore MUST NOT carry either. It
SHOULD NOT carry Generation Time, for which this document defines no
use without ETU.

A metadata-bearing message SHOULD NOT include Generation Time unless
it also includes Expected Time Until Link Usability, since this
document defines no other use for it and unnecessary absolute-time
information should not normally be disclosed. This is not a
prohibition; experimentation or later experience may identify another
legitimate use.

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
in the other cases those documents prohibit. Nothing in this document
suppresses a Destination Unreachable error that those documents
otherwise require.

On receiving trans-boundary traffic, a gateway proceeds as follows:

1. Assess forwarding and service state, including whether another
   path in the signaling domain can serve the traffic. If so, use that
   path and generate neither signal. The aggregate-domain restriction
   below applies to attaching Link Condition Objects and to generating
   Denied by Link Policy; it does not alter when ordinary Destination
   Unreachable errors are generated.

2. Apply the constrained link's admission policy before disclosing
   any link state or prediction.

3. If the traffic is not admitted, drop it and, where configured,
   send Denied by Link Policy, without Expected Time Until Link
   Usability or Expected Link Delay ({{applicability-objects}}).

4. If the traffic is admitted but cannot be forwarded because the
   required link is presently unusable under a transient condition
   known to the domain, drop it and generate the Destination
   Unreachable error that ordinary ICMP rules call for. If the
   conditions of {{ltu}} are satisfied and the code is an eligible
   carrier ({{carriers}}), the gateway MAY attach Expected Time Until
   Link Usability, Generation Time, and Expected Link Delay objects
   ({{applicability-objects}}).

5. If the traffic is admitted and the link is usable, forward it
   normally. A forwarded packet MUST NOT elicit a Destination
   Unreachable error.

A gateway MUST NOT attach Link Condition Objects or generate Denied by
Link Policy if another path within the signaling domain can provide
service inconsistent with the signal. These signals describe the
domain's aggregate ability to serve the traffic toward the destination
({{applicability}}). A gateway that cannot determine this property for
a destination MUST NOT attach Link Condition Objects or generate Denied
by Link Policy for it; it generates ordinary errors under ordinary
rules.

A gateway MUST NOT attach Link Condition Objects on the basis of
observed local interface or carrier state alone. The transient
condition they describe MUST derive from the domain's knowledge of the
constrained link's availability ({{applicability}}).

A gateway MUST NOT send an ETU value not derived from its current
knowledge of the link. A gateway that does not know when the link is
expected to become usable omits the ETU object. A gateway MAY send ETU
without Generation Time; it SHOULD NOT send Generation Time without
ETU ({{applicability-objects}}). If metadata cannot be generated, is
withheld by policy, or is out of range ({{durations}}), the gateway
sends the ordinary error without the affected objects.

Attachment of Link Condition Objects and generation of Denied by Link
Policy MUST be configurable per policy. An operator MUST be able to
configure, per source, prefix, or policy class, whether Denied by Link
Policy is generated at all, the alternative being silent discard. An
operator SHOULD be able to configure omission of the Link Condition
Objects ({{security}}).

A gateway MUST NOT attach Link Condition Objects to, or generate Denied
by Link Policy in, messages sent toward the constrained link, and MUST
NOT do so in response to traffic arriving from it ({{applicability}}).

The absence of these signals promises nothing. ICMP delivery is
unreliable and rate limited, and a gateway MUST NOT assume that a
sender received any particular notification; it generates a fresh
notification, subject to rate limiting, for each invoking packet that
meets the conditions above.


# Receiver Processing {#receiver}

## Validation {#validation}

Ordinary validation and handling of a Destination Unreachable error
are governed by the receiver's existing ICMP and transport rules; this
document does not change them and does not impose the checks below on
bare errors.

Before acting on Link Condition Objects or on Denied by Link Policy, a
receiver MUST validate the message as an ICMP error per {{RFC4443}}
and the mitigations of {{RFC5927}}: the invoking packet excerpt is
matched against existing connection or flow state, and a message
matching nothing the receiver sent is discarded. A receiver SHOULD
disregard Link Condition Objects, and SHOULD discard Denied by Link
Policy, when the message arrives on an interface facing outside its
administered domain. Disregarding the objects for experimental use
does not require rejecting the otherwise valid ordinary error.

The carrier code does not prove that a message originated inside the
domain, and a filter that acts on code alone cannot selectively
contain Link Condition Objects without also affecting ordinary errors
of the same code. A deployment that requires selective containment of
the metadata at a boundary needs extension-aware enforcement; this
document does not specify one.

## Consumer Behavior {#consumers}

This section is informative. This document does not specify how a
host, transport, application, or management system reacts to the
signals, and no such reaction is required for interoperability.
Receipt of a Destination Unreachable error carrying Link Condition
Objects informs the receiver that the domain models the path as
transiently unavailable and, where ETU is present, roughly when
usability is expected; receipt of the same error without the objects
carries only the ordinary meaning of its code. Receipt of Denied by
Link Policy informs the receiver that the link's admission policy did
not admit the traffic. What follows are possibilities that the signals
enable and that the experiment of {{experiment}} is intended to
explore.

A host might record the metadata to avoid immediately repeating
attempts known to have failed. Because the metadata already reflects
every path the domain could offer, immediate repetition is likely to
elicit the same error. How such state would be keyed, how widely it
would apply, how long it would live, how it would be invalidated when
the domain's knowledge changes, and how it would interact with routing
are all open questions. Because ETU is an estimate and the gateway
cannot revoke a notification, any such state is a hint about the
domain's knowledge at generation time and not an assurance of service
at any later time.

A transport might use the metadata as an input to its timeout and
failure decisions: for example, comparing the expected wait against a
connection's tolerance, or treating Denied by Link Policy as a
condition that retrying unchanged will not resolve. The remote
endpoint receives no corresponding notification and continues to run
its own timers, which bounds the usefulness of waiting. No TCP or QUIC
behavior is defined here, and the presence of metadata does not change
how a transport is required to treat the underlying error.

An application might be given the condition and its metadata so that
it can defer, fail fast, select another communication mechanism such
as a bundle service, or trigger provisioning. Whether such an
interface should be standardized is not addressed here.

Stronger reactions to unauthenticated ICMP information carry greater
risk ({{security}}); which reactions are worth their risk is itself an
experiment question.


# Legacy Host and Middlebox Behavior {#legacy}

The two signals present different legacy questions.

The carrier codes are long-established. {{Section 3.9.2.2 of RFC9293}}
classifies ICMPv4 Destination Unreachable codes 0 and 1 and ICMPv6
Destination Unreachable codes 0 and 3 as soft errors, which TCP MUST
NOT abort a connection on, while also noting widespread implementation
behavior that treats soft errors as hard errors during connection
establishment; {{RFC5461}} documents further divergence. A stack that
does not process {{RFC4884}} extensions handles a metadata-bearing
message exactly as it handles the same error today; the objects are
invisible to it. This document does not promise identical behavior
across stacks or that any connection survives, only that the metadata
does not alter the ordinary error a legacy stack sees.

Denied by Link Policy is a new code under an existing type, so legacy
behavior is whatever a stack does with an unrecognized Destination
Unreachable code. {{Section 4.2.3.9 of RFC1122}} and {{RFC9293}}
partition the codes they enumerate into hard and soft errors but
prescribe nothing for other codes; {{RFC4443}} is likewise silent.
Deployed stacks variously treat an unrecognized code as a generic
unreachable, as a soft error, as a hard error, or discard it on a
range check. This document does not assume that all stacks behave
alike; recording actual behavior across the stacks present in target
environments is an experiment question ({{exp-signal}}). Each of these
behaviors is acceptable for this code: an abort is a reasonable outcome
for unauthorized traffic, soft-error handling reaches the same outcome
through timer expiry, and discard leaves the sender where a silently
discarding gateway leaves it today.

Firewalls and stateful middleboxes inside the deploying domain may
discard an unfamiliar Destination Unreachable code such as Denied by
Link Policy under default-deny policy, and may strip or discard
messages carrying unfamiliar extension structures. This fails safe but
silently defeats the mechanism, so deployment includes reviewing ICMP
filtering policy on the paths between gateways and the hosts they
serve; {{RFC4890}} provides the ICMPv6 filtering framework. Legacy
hosts generate nothing new and are unaffected as senders.


# Operational Considerations {#operational}

Gateways SHOULD count generation of Denied by Link Policy and of
metadata-bearing Destination Unreachable messages, including which
Link Condition Objects were included, and expose the counters through
network management. Counts of ordinary Destination Unreachable errors
do not by themselves identify this experiment's activity, since the
carrier codes are generated for other reasons as well. A gateway
generating Denied by Link Policy knows which policy dimension refused
the traffic even though the code does not convey it; gateways MAY log
or report that detail through local operational telemetry, which is
where operators debugging a refused flow should look. This document
does not prescribe a management or telemetry mechanism. In a correctly
provisioned domain these signals are rare for authorized traffic, so
the counters serve as the divergence telemetry of {{uc-divergence}}.

ICMP rate limiters deserve attention at contact boundaries. When a
link becomes unusable, every active trans-boundary flow can elicit an
unreachable error within a short interval, and a token-bucket limiter
tuned for steady-state error rates may suppress most of the burst.
Because ICMP is unreliable and a gateway cannot know which
notifications were received, operators SHOULD size gateway ICMP rate
limits for the expected flow fan-out at gap onset. Hosts that act on
the metadata to avoid repeated attempts reduce the offered load that
would otherwise sustain the burst.

ETU is only as meaningful as the gateway's knowledge of the link.
Absolute interpretation of Generation Time additionally depends on the
receiver being able to relate it to its local time reference;
receivers that cannot may apply ETU from receipt ({{etu}}).

Hosts that maintain state derived from these signals SHOULD expose it
to local diagnostics, since such state changes transmission behavior
in ways otherwise invisible to troubleshooting.


# Experimental Status and Goals {#experiment}

This document is published as Experimental. The objects and the Denied
by Link Policy code are specified normatively so that independent
implementations interoperate at the ICMP layer. The value of the
mechanism, however, rests on two kinds of questions that only
deployment can answer: whether gateways can generate the signals
correctly and usefully, and whether the metadata lets consumers make
better decisions than the ordinary unreachable error alone would allow.
Consumer behaviors are deliberately not standardized here and are not
prerequisites for interoperability; the experiment is intended to
produce the experience on which later specification of such behaviors
might be based.

## Scope of the Experiment {#exp-scope}

The experiment runs within administered limited domains meeting the
criteria of {{applicability}}: space networking testbeds and missions,
other networks with scheduled or intermittently available managed
links, and laboratory emulations of the same topologies. Link
Condition Objects and Denied by Link Policy are not exchanged across
uncontrolled networks, and no behavior in this document affects hosts
whose traffic never requires a trans-boundary link.

Prior to IANA assignment of the Denied by Link Policy codes and the
Link Condition Object class, implementations use the experimental
values reserved by {{RFC4727}} for those and coordinate their
interpretation bilaterally, per that document's rules. The carrier
codes are already assigned and are used as is. Interoperability
reports from this phase are in scope for the experiment.

## Network Signal Evaluation {#exp-signal}

The first layer of the experiment concerns the signals themselves:

- Generation correctness: whether gateways attach Link Condition
  Objects and generate Denied by Link Policy only under the conditions
  of {{generation}}, whether the aggregate-domain determination can be
  made reliably in deployed routing architectures, and which carrier
  codes gateways actually produce for the constrained-link condition
  in each IP version ({{carriers}}).

- Distinction: whether the metadata usefully distinguishes transient
  constrained-link unavailability from ordinary unreachable errors of
  the same code, and whether policy denial remains distinguishable
  from both ({{interaction}}).

- Prediction accuracy and usefulness: how closely ETU tracks actual
  link usability, how often it is available at all, and whether
  Generation Time and Expected Link Delay are populated and used.

- Generation rate: gateway ICMP generation rates at gap onset under
  realistic fan-out, and the effect of rate limiting on which senders
  receive a notification.

- Legacy and middlebox behavior: how unmodified hosts respond to
  metadata-bearing carrier errors and to the new Denied by Link Policy
  code across the stacks present in target environments, including
  embedded and flight-heritage implementations ({{legacy}}); and what
  filtering changes deployments required, including whether
  middleboxes strip or discard extension structures.

- Security and disclosure: whether the information disclosed by the
  Denied by Link Policy code and by the metadata is operationally
  acceptable, whether the disclosure controls of {{generation}} are
  sufficient, and whether forged or stale notifications caused harm.

- Whether the coarse Denied by Link Policy signal is operationally
  useful, whether it warrants a permanent distinct code or an existing
  administratively prohibited code would serve, and whether experience
  establishes a need for richer policy-denial metadata. This document
  does not speculate about the format of any such metadata.

## Consumer Behavior Exploration {#exp-questions}

The second layer concerns what receivers do with the signals. None of
these behaviors is specified here; the experiment is intended to
discover which are worthwhile:

- Whether hosts that record the metadata to avoid repeated attempts
  reduce futile retransmission and offered load during
  link-unavailable periods, relative to hosts receiving the same
  ordinary errors without metadata, and what keying, scope, lifetime,
  and invalidation rules work.

- What transport reactions are appropriate, including whether
  comparing expected wait against connection tolerance is useful and
  whether connections that wait survive the peer's own timers.

- What application decisions the metadata enables, including
  selection among communication mechanisms such as IP transports and
  bundle services, and whether an application interface should be
  standardized.

- Whether provisioning and orchestration systems can use the signals,
  and whether divergence telemetry ({{uc-divergence}}) exposed real
  faults ahead of existing monitoring.

## Evidence for Advancement {#exp-criteria}

Later consideration of the mechanism for the standards track could be
informed by evidence of: at least two independent interoperable
implementations of the Link Condition Objects and of Denied by Link
Policy, including extension object processing; generation accuracy in
at least one operational, non-laboratory domain; usefulness of the
metadata objects beyond the ordinary unreachable error, with unused
objects removed rather than carried forward; manageable legacy and
middlebox behavior; acceptable security and disclosure properties;
consumer behaviors independently demonstrated to benefit from the
metadata, without presupposing any one receiver strategy; and evidence
that the semantics apply across more than one class of constrained
network without special-case changes to the wire format. Reports of
failed or inconclusive experiments are requested to the same degree as
successful ones.


# Security Considerations {#security}

ICMP carries no authentication, and both the metadata and Denied by
Link Policy are actionable by implementations that choose to act on
them, so the principal threats are forgery by off-path attackers and
information disclosure to the senders these messages answer. The
containment for both is the limited-domain scoping of
{{applicability}}: validation against sender state ({{validation}}),
boundary filtering of Denied by Link Policy, rate limiting, and a
single administrative authority. None of these defenses is dependable
across the open internet, and none authenticates the carrier code or
the metadata.

Three forgery cases arise. A forged ordinary Destination Unreachable
error is an existing threat with existing mitigations ({{RFC5927}});
this document does not change it. Forged Link Condition Objects on an
otherwise plausible error could induce an implementation that acts on
them to withhold trans-boundary traffic for the advertised interval.
This document does not require receivers to withhold traffic, and
stronger reactions to unauthenticated ICMP information carry
correspondingly greater risk; an implementation that does act on the
metadata should bound the effect of any single notification, treat ETU
as an estimate that may be stale or wrong, and consider that the
gateway cannot revoke a notification whose basis has changed. The
invoking-packet validation of {{RFC5927}} forces an attacker to guess
connection state, and in-domain filtering confines injection to
on-path or in-domain attackers. A forged Denied by Link Policy message
is in the class of attacks {{RFC5927}} analyzes for existing hard-error
codes; it adds no capability beyond forging the existing
administratively prohibited codes, and the same mitigations apply.

Because the metadata rides on ordinary codes, a boundary filter on
code values alone cannot contain it without also blocking ordinary
errors ({{validation}}). Deployments that require such containment
need extension-aware enforcement, which this document does not
specify. The absence of Link Condition Objects from a message is not a
trustworthy indication that no constrained-link condition exists,
since objects may be omitted by policy, by rate limiting, by
middleboxes, or by an attacker.

Denied by Link Policy reveals to the sender that constrained-link
admission policy refused its traffic, and a responsive gateway is a
probing oracle: an attacker varying markings, sources, and
destinations can map which traffic the domain admits, even without
being told why any particular packet was refused. {{generation}}
therefore requires per-policy control over whether the code is
generated at all; toward any sender the operator does not trust,
silent discard is the expected posture, consistent with existing
firewall practice.

The Link Condition Objects disclose link schedule information, which
in some deployments is sensitive operational information.
{{generation}} permits omitting them per policy; operators of such
deployments should protect the metadata as they protect the underlying
schedule. Evaluating admission before link state ({{generation}})
prevents disclosure of schedule information to traffic that would not
be admitted regardless.

Ordinary ICMP generation restrictions and rate limiting
({{Section 2.4 of RFC4443}}, {{RFC1812}}) apply to all of these
messages. Quoting and padding of the invoking packet and the optional
objects affect response size; a message carrying all three objects is
larger than a minimal Destination Unreachable error. This document
makes no claim about the absence of amplification beyond what those
restrictions provide.


# IANA Considerations {#iana}

This document requests the following assignments, shown as TBD values
throughout. No numeric values are proposed.

From the "Type 1 - Destination Unreachable" code registry of the
"ICMPv6 Parameters" registry: code TBD3, Denied by Link Policy. The
registration procedure for this registry is Standards Action or IESG
Approval; this document requests IESG Approval.

From the "Type 3 - Destination Unreachable" code registry of the
"ICMP Parameters" registry: code TBD4, Denied by Link Policy. The
registration procedure for this registry is IESG Approval or Standards
Action; this document requests IESG Approval.

From the "ICMP Extension Object Classes and Class Sub-types" registry
{{RFC4884}}: a new class TBD5, Link Condition Object, with C-Type 1,
Generation Time; C-Type 2, Expected Time Until Link Usability; and
C-Type 3, Expected Link Delay, assigned by this document. Further
C-Types are Specification Required.

This document requests no change to the already-assigned Destination
Unreachable codes on which Link Condition Objects may be carried:
ICMPv6 Type 1 codes 0 and 3, and ICMPv4 Type 3 codes 0 and 1
({{carriers}}). Their existing meanings are unchanged.

Publication as Experimental does not by itself satisfy the
registration procedures for the Destination Unreachable code
registries. Should the requested code assignments not be approved,
the allocation strategy is a matter for further review.


--- back

# Worked Example {#example}

A gateway whose deep-space link is presently unusable receives an
IPv6 packet for a trans-boundary destination at 18:00:00 UTC on 16
September 2026. No other path in the signaling domain can serve the
traffic. The packet is admissible under the link's policy. The gateway
retains a route toward the destination, but the next hop over the
deep-space link is unusable; under {{Section 3.1 of RFC4443}} this is
a link-specific problem not covered by another code, so the gateway
generates ICMPv6 Destination Unreachable, Code 3 (Address
unreachable). The domain's contact plan shows the link becoming usable
42 minutes later, and the gateway's one-way delay estimate for the
link is 20 minutes, so it attaches three Link Condition Objects. The
Class-Num octet of each is TBD5 and is omitted from the listing.

| Object | C-Type | Wire value | Decoded |
|:-------|-------:|:-----------|:--------|
| Generation Time | 1 | 0xEE5557A0 0x00000000 | 2026-09-16 18:00:00Z |
| Expected Time Until Link Usability | 2 | 0x002673C0 | 2520000 ms (42 min) |
| Expected Link Delay | 3 | 0x00124F80 | 1200000 ms (20 min) |
{: #tab-example title="Worked Example Extension Objects"}

The Generation Time seconds word is the Unix time of the instant plus
2208988800; the fraction word is zero.

Had the gateway attached no objects -- because it had no estimate,
withheld it by policy, or lacked the domain knowledge required by
{{ltu}} -- the receiver would have an ordinary Address Unreachable
error, meaning only that the packet was not delivered. A receiver that
does not process {{RFC4884}} extensions sees exactly that in either
case.

A receiver that processes the objects but cannot relate Generation
Time to its own clock, or that did not receive it, needs no timestamp
arithmetic: it knows that, as of generation, the link was expected to
become usable in 42 minutes, and it may apply that interval from the
moment of receipt. If the notification spent time in delivery, the
receiver waits slightly longer than necessary, which errs on the safe
side.

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
