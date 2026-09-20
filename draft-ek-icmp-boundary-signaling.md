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
  RFC4884:
  RFC8877:

informative:
  RFC1122:
  RFC4890:
  RFC4950:
  RFC5461:
  RFC5837:
  RFC5905:
  RFC5927:
  RFC6069:
  RFC8799:
  RFC8883:
  RFC9171:
  RFC9293:

...

--- abstract

This document defines optional ICMP extension objects that a gateway at
the boundary between an IP enclave and a managed constrained long-haul
link, such as a scheduled deep-space link, can attach to existing
Destination Unreachable messages. On the ordinary unreachable errors
generated when an admitted packet cannot be forwarded because the
required link is presently unusable, the objects convey the expected
time until the link becomes usable, the time the estimate was generated,
and the expected one-way delay of the link. On the existing
administrative prohibition errors generated when the link's admission
policy refuses a packet, a presence-only object identifies boundary-link
admission policy as the source of the refusal. The underlying
Destination Unreachable codes retain their existing meanings, and this
document requests no new ICMP type or code. It specifies when the
objects may be attached and what they convey; it does not specify how
transports, applications, or hosts react to them. Making the boundary's
knowledge available to senders is intended to enable experimentation
with such reactions.


--- middle

# Introduction {#intro}

Some IP enclaves are connected to the rest of a network only by
constrained long-haul links that are high-delay, periodically disrupted,
or both: deep-space links, satellite links available only during
scheduled passes, periodic store-and-forward contacts. An administrative
domain, or a set of cooperating domains, manages those links: it knows
from a contact plan or equivalent scheduling knowledge when each link is
expected to carry traffic, it enforces an admission policy describing
whose traffic each link will carry, and it can distribute routing
updates to migrate long-haul traffic onto the best current or next link
according to its policy. An IP sender inside such an enclave has none of
this knowledge. Today it learns of a link's unavailability or of a
policy refusal only through its own timers, retransmitting until they
expire and then receiving a failure indistinguishable from a routing
fault or a crashed peer.

Existing Destination Unreachable messages already report the
forwarding failures that result. This document defines optional ICMP
extension objects {{RFC4884}} that allow a gateway in a managed
long-haul domain to supply additional information on those messages.
On the ordinary unreachable errors generated when an admitted packet
cannot be forwarded because the required link is presently unusable,
the gateway may attach the time at which it generated the message,
its estimate of the time until the link becomes usable, and the
approximate one-way delay of the link. On the existing administrative
prohibition errors generated when the link's admission policy refuses
a packet, the gateway may attach a presence-only object indicating
that boundary-link admission policy, rather than some other
administrative control, was the source of the refusal. In every case
the underlying Destination Unreachable code retains its existing
meaning. When the objects are absent, or a receiver does not process
them, the message provides the ordinary Destination Unreachable
indication and nothing more.

This document specifies the network-layer signals: when a gateway may
attach each object, what each asserts, and its encoding. It does not
specify how a transport, application, host cache, provisioning system,
or orchestrator reacts to them. Such reactions are not required for
interoperability, and they are not merely out of scope: a purpose of
publishing the objects is to enable experimentation with them. A host
might defer further attempts; a transport might use the information as
an input to its timeout and failure decisions; an application that
learns its present IP traffic was refused by boundary-link policy, or
will be deferred for a long interval, might reconsider its end-to-end
communication options, for example by using a separately provisioned
application-layer gateway or Bundle Protocol {{RFC9171}} service.
Experience with such reactions may motivate later specifications
({{experiment}}).

The availability metadata is useful because the boundary is close and
the condition is long-lived: gateway feedback reaches senders in local
round-trip times, while the unavailability it reports lasts minutes to
hours ({{applicability}}). The policy indication is useful for a
different reason: it tells an application that changing what it sends,
rather than waiting, is what would help.

Where an application can learn in advance which communication
mechanisms suit a destination -- through configuration, provisioning,
DNS, or orchestration -- selecting an appropriate mechanism before
initiating unsuitable IP communication is preferable to discovering
the condition reactively. Such discovery and selection mechanisms are
out of scope for this document, which addresses only what a gateway
reports once a packet has been sent.

The objects are specified for use within administered limited domains
{{RFC8799}}, or cooperating domains that have agreed on a common
coordination and disclosure scope. {{experiment}} defines the
experiment this document proposes.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Enclave:
: an IP network whose connectivity to other networks passes over one
  or more managed constrained long-haul links.

Constrained long-haul link:
: a link connecting enclaves whose usability is high-delay,
  scheduled, intermittent, or otherwise restricted according to
  knowledge held by the managing domain, and whose use is governed by
  admission policy. Ordinary transient interface or carrier failure
  does not by itself make a link a managed constrained long-haul link
  ({{applicability}}).

Managed service:
: the domain's knowledge that a particular constrained long-haul link,
  or a sequence of link opportunities, is intended to carry particular
  traffic, together with the associated schedule, admission policy, and
  forwarding state. A managed service may exist while no route to its
  destination is currently installed.

Orchestration function:
: the logical function, possibly distributed across cooperating
  administrative domains and not necessarily continuously reachable,
  that manages the constrained long-haul links, distributes routing and
  policy updates for them, and supplies gateways with the knowledge on
  which the objects in this document are based.

Contact:
: an interval during which a constrained long-haul link is planned to
  carry traffic between a gateway and its remote counterpart.

Contact plan:
: the schedule of contacts for a link or set of links, distributed to
  gateways in advance of execution. A contact plan is one form of
  managed-service knowledge, not the only one.

Gateway:
: the router at the boundary between an enclave and a constrained
  long-haul link, holding the managed-service knowledge for that link
  and attaching the objects defined in this document.

Coordination scope:
: the set of links, routes, and admission policies under the same
  orchestration function, whose current state a gateway's assertions
  describe.

Disclosure scope:
: the administrative boundary within which the objects are generated
  and within which their contents are considered appropriate to
  reveal. It is at most the coordination scope and may be narrower.

Trans-boundary traffic:
: traffic whose path crosses a gateway onto a constrained long-haul
  link.

Admission:
: a gateway's determination, against local policy, that particular
  traffic may use a constrained long-haul link.

Link Condition Object:
: an ICMP extension object of the class defined in {{objects}},
  carrying constrained long-haul link metadata on a Destination
  Unreachable message.

Carrier code:
: an existing Destination Unreachable code on which one or more Link
  Condition Objects may be carried ({{carriers}}).

Expected Time Until Link Usability (ETU):
: the gateway's estimate, at generation of a notification, of the
  interval from generation until the relevant constrained long-haul
  link is expected to become usable for forwarding ({{etu}}).

Enclave time reference:
: the shared time reference, established by the specification or
  provisioning of the managed deployment, against which Generation
  Time is expressed ({{gen-time}}). Its time scale, epoch, and
  discontinuity treatment are deployment choices, not protocol
  prerequisites; UTC is one permitted choice.


# Applicability {#applicability}
The mechanisms in this document apply to deployments in which one or
more constrained long-haul links connect IP enclaves, and in which the
links, the routes over them, and the admission policies governing them
are managed by a common orchestration function. When several long-haul
options exist for the same traffic, this document assumes they
participate in the same logical orchestration. This does not require a
single physical controller, a particular routing protocol, or continuous
connectivity between gateways and the controller ({{orchestration}}).
({{orchestration}}).

Three relationships need to be kept distinct.
The first is the timing relationship between a notifying gateway and the
senders it notifies. Let R be the round-trip time between a sender and
the gateway, D the duration of a link unavailability being reported, and
T the timescale on which the sender's transport would otherwise detect
failure. Availability metadata is useful when R is small relative to T,
so the notification arrives while the sender still has options, and
small relative to D, so the reported condition is still in effect on
arrival. In the Mars enclave of {{conops}}, R is milliseconds to
seconds, T is seconds to minutes, and D is minutes to hours. When R
approaches D, a notification describes a link state that may no longer
exist on arrival; gateways therefore attach objects only into their
local enclave and never across the constrained long-haul link. This
timing argument is specific to availability metadata. A policy refusal
applies regardless of current link usability and needs no predicted
outage to be worth reporting.
predicted outage to be worth reporting.

The second is the coordination scope: the set of links, routes, and
policies whose current state the gateway's assertions describe. A
gateway's assertions are about the applicable, current managed
service and forwarding state for the affected traffic within this
scope. They are not claims about every path that could ever exist,
about unprovisioned routes, about unrelated domains, or about
application-layer alternatives. Equally, the failure of one local link
is not by itself a basis for an assertion when the current managed
forwarding state provides a usable, policy-permitted alternative for
that traffic.

The third is the disclosure scope: the administrative boundary within
which the objects are generated and considered appropriate to reveal.
Cooperating domains must agree on both the coordination scope and the
disclosure scope; their agreement does not authenticate ICMP, and a
short local round-trip time does not by itself make a network part of
either scope. For comparison of timescales only, the public Internet
has sub-second internal round-trip times relative to a link whose
unavailability is measured in minutes, but it is not thereby a
cooperating or trusted signaling domain.

The objects apply only to links whose availability and admission the
domain deliberately manages as a long-haul service. A gateway attaches
availability metadata on the basis of managed-service knowledge about
the link, not on the basis of observed local carrier or interface
state. A router that merely notices its outgoing interface is down
generates whatever Destination Unreachable error ordinary ICMP rules
call for, but has no basis to attach Link Condition Objects to it;
ordinary transient link failures are not the condition this document
addresses. How the domain acquires and distributes its knowledge is
not specified; the objects encode what the gateway is entitled to
assert, not how it learned it.

The same criteria identify deployments beyond the interplanetary case,
wherever a domain manages scheduled or intermittently available links
between enclaves on the basis of advance knowledge. Conversely, the
mechanism offers nothing on paths that are merely lossy or congested;
existing congestion signaling and routing convergence address those
conditions. High delay alone is not a reason to generate a Destination
Unreachable error when forwarding is possible.

The objects are specified for use within limited domains {{RFC8799}}
under a single administrative authority or cooperating authorities
sharing a coordination and disclosure scope. ICMP carries no
authentication; validation against sender state and filtering are
mitigations, not authentication ({{security}}). The carrier codes are
ordinary Destination Unreachable codes whose filtering remains a
matter of existing operational policy; see {{validation}} regarding
the objects they may carry.


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

Gateways hold the managed-service knowledge for their deep-space
links, enforce admission and precedence policy on contact capacity,
and attach the objects defined in this document: availability
metadata on the unreachable error generated when admitted
trans-boundary traffic arrives while the link is not usable, and the
Denied by Link Policy object on the administrative prohibition error
generated when traffic arrives that policy does not admit.

An orchestration function, which may be Earth-based, distributes
contact plans, routing updates, and admission policy to gateways in
advance ({{orchestration}}). It is intended to steer trans-boundary
traffic toward whichever gateway and link can serve it, so that in
coordinated operation traffic reaches a gateway that cannot serve it
only when no managed alternative currently exists. Routing, link, and
policy updates are not installed atomically, however, and distribution
itself crosses the deep-space link; during transitions, or when plan
and reality diverge, a gateway may receive traffic that another
gateway could have served. Gateways operate autonomously between
management contacts on their most recent applicable knowledge, and
omit assertions they cannot currently support ({{generation}}).

Intra-enclave traffic is unaffected. The objects are attached only to
errors for traffic that requires a trans-boundary link, and hosts that
never originate such traffic need not implement them.

## Orchestration Example {#orchestration}

This subsection illustrates one feasible path by which a gateway
obtains the knowledge on which it bases the objects. It describes
conceptual information, not a management protocol, data model, or
scheduling algorithm; any existing management channel, controller
interface, or local provisioning may carry it.

Times in this example are expressed in the enclave time reference
({{gen-time}}) as seconds from its epoch; the deployment has chosen
the reference, and nothing here depends on which one. The
orchestration function determines that traffic for a terrestrial
destination prefix should, for an upcoming period, leave the enclave
via a particular relay orbiter's deep-space link, whose next contact
begins at reference time 12,400 s. Allowing for antenna pointing,
acquisition, and modem configuration, it expects the link to be ready
to forward traffic at reference time 12,520 s, and it estimates the
one-way delay over the link at that time as 20 minutes. It installs or schedules the routing
and admission policy needed for that link, and it supplies the
gateway with an associated record containing at least:

- the traffic-to-service binding: the destination prefix and the
  routing or policy context to which the record applies, associated
  with the relevant link or next managed opportunity;

- a revision identifier or equivalent coherence mechanism, an
  applicability period, and validity or expiry information;

- optionally, the expected data-plane-ready time, or a duration with
  an explicit reference time;

- optionally, a directional one-way link-delay estimate applicable to
  that opportunity; and

- the admission and disclosure policy that applies, together with
  whatever local evidence the gateway has about the link's actual
  usability.

Delivering this record is only part of the implementation. The
gateway's error-generation path must be able to consult it when
forwarding fails, including when the route to the destination has
been withdrawn between contacts and the failure is reported as a
no-route error.

At reference time 10,000 s, a packet for the prefix arrives at the
gateway. No usable route exists, so ordinary rules produce a
Destination Unreachable error. The gateway consults its record, finds
it current and applicable, confirms the traffic is admitted, and
attaches Generation Time (10,000 s), ETU (the interval from 10,000 s
to 12,520 s, 2,520,000 ms), and Expected Link Delay (1,200,000 ms).
Generation Time is the time the ICMP message was generated, not the
time the orchestrator created or distributed the plan.

Several operational cases follow from this model, and
{{operational}} draws the corresponding lessons:

- A later plan revision selects a different link or changes the
  readiness estimate. Subsequent notifications use the current
  applicable information; earlier notifications cannot be recalled.

- The planned readiness time passes but acquisition fails. The gateway
  does not continue reporting a countdown clamped to zero; it either
  has a newly supported estimate, or it omits ETU. An Expected Link
  Delay that remains valid for the selected link need not be omitted.

- Controller connectivity is lost. A still-valid local record remains
  usable unless contradicted by local evidence; when it expires or
  becomes incoherent, the basis for the affected objects is gone and
  they are omitted. Ordinary errors continue to be generated.

- Routing state and metadata revisions temporarily disagree, for
  instance when a route has migrated to a second gateway but the
  first gateway's record has not yet been updated. The gateway omits
  assertions it cannot support rather than describing a service
  opportunity other than the one applicable to the traffic.

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

### Reconsidering the Communication Mechanism {#uc-handoff}

An application that learns, from a Link Condition Object, that its
present IP traffic will be deferred for a long interval or was refused
by boundary-link admission policy may reconsider its end-to-end
communication options. A DTN-aware host might, as a matter of local
policy, hand the affected traffic to its local BP agent, which
originates bundles toward the destination for forwarding during the
next contact. An application with separately provisioned knowledge of
an authorized application-layer gateway might direct its traffic
there. An application with its own delay tolerance might instead
compare the expected wait against that tolerance and fail fast.

The object is a trigger to reconsider, not a description of the
alternatives. Configuration, discovery, or provisioning supplies the
alternative mechanism and its authorization; the object does not prove
that another mechanism exists, is admitted, or will succeed, and it
does not require the application to change anything.

### Traffic Denied by Link Policy {#uc-denied}

A misconfigured or newly integrated payload host sends best-effort
traffic toward Earth during a contact whose capacity is allocated to
command and telemetry classes. The gateway classifies the traffic
against its local policy, which may consider DSCP, addresses, protocol
and port, or any other criteria, and drops it. Ordinary rules and
local response policy call for a Destination Unreachable, Communication
Administratively Prohibited error; the gateway attaches the Denied by
Link Policy object to it. From the object, the host learns that the
refusal came from the admission policy of the long-haul service
rather than from some other administrative control such as a
firewall; from the bare error it would know only that communication
was administratively prohibited. The object does not say why the
traffic was refused or what traffic would be admitted, since admission
policy is multidimensional and local, and coaching senders on
acceptable markings would invert an admission model in which access
is granted through the control plane. The operator debugging the flow
consults the gateway's own logs and counters ({{operational}});
obtaining admission is a provisioning action. An application receiving
the object may reconsider its communication mechanism as described in
{{uc-handoff}}.

A flow that policy admitted at contact start and later displaces in
favor of higher-precedence traffic falls under the same object.
Whether an implementation regards this internally as preemption,
resource allocation, or precedence enforcement is a matter of local
policy; the sender learns only that the traffic is not presently
admitted.

### Plan/Reality Divergence {#uc-divergence}

In nominal operation, provisioned traffic should never elicit these
objects. Availability metadata generated during a planned contact, or
a Denied by Link Policy object generated for a flow the orchestrator
believes it admitted, indicate that the control plane's model and the
data plane's state have diverged: an unplanned outage, a policy or
routing distribution failure, or traffic outside the provisioning
workflow. Gateways count and report generation of these objects so
that operators can use them as a diagnostic feed ({{operational}}).

## Interaction of the Two Signals {#interaction}

The availability metadata and the Denied by Link Policy object answer
different questions and may both be relevant to one flow over time: a
link can be simultaneously restricted and unavailable. Admission is
evaluated first. Traffic that would be refused regardless of link
state receives the administrative prohibition error, optionally with
the Denied by Link Policy object, and never receives availability
metadata; this avoids disclosing link schedule information to traffic
that could not use the link anyway. Traffic that is admitted but whose
link is not presently usable receives the ordinary unreachable error,
optionally with availability metadata. A sender whose denial is
resolved through provisioning may then receive a metadata-bearing
unreachable error for the same flow: first become admissible, then
wait for the link.
Ordinary Destination Unreachable errors do not by themselves establish
that service is permanently absent; {{Section 3.2.2.1 of RFC1122}} notes
that they can result from routing transients, and {{RFC6069}} has
previously used them experimentally as indications of connectivity
disruption. Nor does an ordinary administrative prohibition error
identify which administrative control refused the packet. This document
does not change either meaning. Availability metadata adds to an
unreachable error the domain's assertion that the failure reflects a
managed transient condition of a constrained long-haul link to which the
traffic was admitted, and, where supplied, an estimate of its duration.
The Denied by Link Policy object adds to an administrative prohibition
error the assertion that boundary-link admission policy was the source
of the refusal.
was the source of the refusal.


# Message Formats and Extension Objects {#formats}

This section is normative. The Link Condition Object class is shown as
TBD5 pending IANA assignment ({{iana}}); prior to assignment,
experiments use a locally coordinated Private Use class as described
in {{exp-scope}}. The carrier codes of {{carriers}} are already
assigned, and this document requests no new ICMP type or code.

## Common Message Format {#common}

The objects are carried in Destination Unreachable messages: ICMPv6
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
({{applicability-objects}}), and MUST ignore a Link Condition Object
whose Length is not valid for its C-Type ({{objects}}). An
unrecognized, inapplicable, or invalid-length Link Condition Object
does not by itself invalidate the underlying ICMP error; this
tolerance does not extend to malformed extension framing or an invalid
extension checksum, which are handled per {{RFC4884}}.

## Constrained Long-Haul Link Metadata on Destination Unreachable Errors {#ltu}
When a gateway does not forward an otherwise admitted packet because the
constrained long-haul link required to reach its destination is
presently unusable, it generates whatever Destination Unreachable error
the ordinary rules of {{RFC1812}} or {{RFC4443}} call for, according to
its own forwarding state. This document does not change which code is
generated or when. It permits the gateway to attach availability
metadata -- the Generation Time, Expected Time Until Link Usability, and
Expected Link Delay objects -- to that error, subject to the conditions
in this section, so that the sender can learn what the bare error does
not convey.
error does not convey.

A gateway MAY attach availability metadata to a Destination
Unreachable message only when all of the following hold:

- the gateway holds a sufficiently current and applicable
  managed-service association between the invoking traffic and a
  constrained long-haul link, or next managed opportunity, within its
  coordination scope;

- according to the current coordinated forwarding state, no other
  policy-permitted path within the coordination scope is presently
  available to serve that traffic;

- the traffic is admitted to that service;

- the link is presently unusable, and the managed-service knowledge
  represents that unusability as a transient interruption of a
  service that is intended to resume;

- the invoking packet was not forwarded; and

- the message's code is an eligible availability carrier code
  ({{carriers}}).

The second condition is an assertion about the current managed
forwarding state for the affected traffic, not about every path that
could exist. A gateway that cannot establish the basis for a
particular object omits that object; ordinary forwarding and ICMP
rules are unaffected. The transient interruption may coincide with
absence of an installed route: a domain may withdraw the route to a
destination between contacts while retaining the managed-service
knowledge that explains the failure, and the resulting no-route error
is an eligible carrier.

A gateway that meets these conditions but has no estimate to supply,
or withholds estimates by policy, sends the ordinary error without
the objects. The gateway need not know when the condition will clear.
None of these assertions attaches to a bare Destination Unreachable
error. A message without Link Condition Objects means only what its code
has always meant; it does not imply that admission succeeded, that a
managed constrained long-haul link caused the failure, or that the
domain predicts recovery. The presence of an Expected Time Until Link
Usability or Expected Link Delay object supplied under this
specification carries the meaning defined for that object in
{{objects}}. A Generation Time object supplies a timestamp and is not by
itself evidence of any particular cause of failure.
by itself evidence of any particular cause of failure.

### Eligible Carrier Codes {#carriers}

Link Condition Objects are applicable only to the Destination
Unreachable codes listed in this section, and each object is
applicable only to the carrier group indicated ({{applicability-objects}}).
Link Condition Objects MUST NOT be attached to other Destination
Unreachable codes or to other ICMP error types. In every case the code
is selected by existing ICMP rules according to the gateway's
forwarding state and response policy; this document neither alters
that selection nor redefines any code.

#### Availability Carriers {#carriers-avail}

The availability metadata objects (Generation Time, Expected Time
Until Link Usability, Expected Link Delay) may be carried on the
following codes.

For ICMPv6, per {{Section 3.1 of RFC4443}}:

| Type 1 Code | Name | Eligibility |
|------------:|:-----|:------------|
| 0 | No route to destination | Eligible when the forwarding table lacks a matching entry for the destination and the gateway holds the managed-service knowledge required by {{ltu}}. |
| 3 | Address unreachable | Eligible when the gateway retains a route whose required link is unusable and no more specific code applies. {{RFC4443}} names a link-specific problem as an example of this code. |
{: #tab-carriers-v6 title="Eligible ICMPv6 Availability Carrier Codes"}

For ICMPv4, per {{Section 4.3.3.1 of RFC1812}} and
{{Section 5.2.7.1 of RFC1812}}:

| Type 3 Code | Name | Eligibility |
|------------:|:-----|:------------|
| 0 | Network unreachable | Eligible when ordinary forwarding failure warrants Network Unreachable, such as absence of any route to the destination network. |
| 1 | Host unreachable | Eligible when ordinary forwarding failure warrants Host Unreachable; see the note below. |
{: #tab-carriers-v4 title="Eligible ICMPv4 Availability Carrier Codes"}

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
yet supply. The worked examples in {{example}} use ICMPv6 so that
their interpretation does not depend on resolving this point.

#### Policy Carriers {#carriers-policy}

The Denied by Link Policy object may be carried on the following
codes, whose existing definitions already cover administrative refusal
of a packet:

| IP version | Type | Code | Existing meaning |
|:-----------|-----:|-----:|:-----------------|
| IPv4 | 3 | 13 | Communication Administratively Prohibited ({{Section 5.2.7.1 of RFC1812}}) |
| IPv6 | 1 | 1 | Communication with destination administratively prohibited ({{Section 3.1 of RFC4443}}) |
{: #tab-carriers-policy title="Eligible Policy Carrier Codes"}

{{RFC4443}}'s reference to a firewall filter is an example of
administrative prohibition, not an exclusive definition. The object
identifies boundary-link admission policy as the particular
administrative control that refused the packet; it does not introduce
a different meaning for the base code.

## Extension Objects {#objects}

The objects below share a single ICMP Extension Object class, the Link
Condition Object class (Class-Num TBD5), distinguished by C-Type. The
object header is as defined in {{Section 8 of RFC4884}}. No object is
mandatory. Which objects a message may carry depends on the
Destination Unreachable code it reports ({{applicability-objects}});
within that set, each object is optional, presence conveys that the
gateway supplied that information, and absence means only that it did
not. No sentinel values are defined for absent data. A message
carrying none of these objects has only the ordinary meaning of its
code.

Because the class now includes both availability metadata and a
policy indication, the presence of a Link Condition Object does not by
itself mean transient unavailability, nor does it necessarily disclose
timing information. Each assertion in this document is attributed to
the specific object that carries it.

Each object has exactly one valid Length: 12 octets for Generation
Time, 8 for Expected Time Until Link Usability and Expected Link
Delay, and 4 for Denied by Link Policy. A receiver MUST ignore a Link
Condition Object whose Length does not match the value for its C-Type
({{common}}).

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

Generation Time is the gateway's reading of the applicable enclave
time reference at the moment it generated the ICMP message. It is the
time of the notification, not the time at which any underlying plan
was created or distributed, and not an arbitrary device-local clock.

Following the timestamp specification template of
{{Section 3 of RFC8877}}, this document fixes the timestamp syntax and
units and leaves the remaining semantics to the deployment:

Size:
: 64 bits in network byte order: a 32-bit unsigned count of whole
  seconds followed by a 32-bit unsigned binary fraction of a second.

Units and resolution:
: seconds and 2^-32 seconds respectively; 1,000 milliseconds per
  second, consistent with the duration objects. No planetary or
  calendar units are used on the wire.

Wraparound:
: the seconds field wraps every 2^32 seconds. No era number is
  carried; era resolution is performed against the applicable enclave
  time reference, not against an implicit wall clock.

Epoch, time scale, and leap seconds:
: supplied by the enclave time reference. The specification or
  provisioning of the managed deployment MUST establish the time
  scale, epoch, treatment of discontinuities such as leap seconds or
  clock resets, and era or wraparound interpretation, and MUST define
  the relationship needed to interpret elapsed seconds so that
  Generation Time can be combined with ETU. Choosing an epoch alone
  does not settle time-scale or discontinuity behavior. UTC with a
  conventional epoch is one permitted choice; it is not a protocol
  prerequisite, and this document does not presuppose the time scale
  a future deployment will adopt.

Synchronization aspects:
: a receiver MAY use Generation Time for a calculation only when it
  knows the applicable enclave time reference and can relate it to
  its own clock with confidence sufficient for that calculation. A
  shared time-scale name alone does not establish clock
  synchronization, and a common orchestration function is not by
  itself evidence of a shared reference or a known conversion;
  cooperating administrative domains need one or the other before
  their receivers compare timestamps. Where the reference, the
  conversion, the era, or the effect of a clock discontinuity is
  unknown or ambiguous, the receiver ignores Generation Time for that
  calculation. No synchronization accuracy is required by this
  document and no synchronization protocol is specified.

The seconds/fraction layout is the same as the 64-bit timestamp
format of {{Section 6 of RFC5905}}, and implementations may reuse
code that handles it. Reusing the layout is separate from adopting
NTP's timestamp semantics: the NTP epoch, the UTC time scale, and
NTP's era conventions are not imported by this document, and apply
only where a deployment selects them as its enclave time reference.
No time-scale identifier or negotiation is carried in this version;
the applicable reference is established through the managed
deployment context. Reference changes and clock resets require
operational handling so that receivers do not unknowingly interpret
timestamps generated under one reference using another
({{operational}}).

Generation Time identifies when the notification and any estimates in
it were generated. It is principally intended to allow a receiver that
can interpret it to account for the time elapsed since an accompanying
ETU estimate was generated ({{etu}}). A receiver that cannot interpret
it ignores it; nothing else in the message depends on it, it remains
independently parseable, and ignoring it does not invalidate ETU, any
other applicable object, or the ordinary ICMP error. In the absence of
ETU this document defines no use for Generation Time, so a message
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
generation until the constrained long-haul link selected for the
invoking traffic is expected to become usable for forwarding it. It is
an unsigned 32-bit integer count of milliseconds, in the format of
{{durations}}. ETU refers to expected data-plane usability, not to the
start of a scheduled contact or of preparation for one. Preparatory
operations necessary to make the link usable, such as antenna
pointing, modem configuration, acquisition, synchronization, or
analogous link-establishment operations, are expected to have
completed by the end of the reported interval and are not represented
separately.

ETU is measured from notification generation, not from receipt. It is
an estimate, not a guarantee. It is not a retry interval and not an
instruction to transmit at any particular time. It does not include
Expected Link Delay, which describes traversal of the link once
usable. When ETU and Expected Link Delay appear in the same message
they MUST describe the same link or managed opportunity; a gateway
MUST NOT combine the earliest usability of one exit with the delay of
another.

ETU is meaningful on its own. A receiver that can interpret a supplied
Generation Time against the enclave time reference ({{gen-time}}) may
compute the expected usability time in that reference as Generation
Time + ETU, and may thereby account for the time the notification
spent in delivery; any era or wraparound considerations apply only to
that interpretation and are resolved against the enclave reference. A
receiver without Generation Time, or unable to interpret it, may apply
ETU from the moment of receipt.
Doing so is a conservative interpretation of the same forecast: it
expires no earlier than the forecast itself, and later by however long
the notification spent in delivery. It is not a guarantee about the
link; actual recovery may occur later than either interpretation, or
earlier. This document does not assume that ICMP delivery is prompt.

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
traversal of the constrained long-haul link selected for the invoking
traffic -- the link whose unavailability prevented forwarding or, where
routing has migrated the traffic, the next managed opportunity that ETU
describes -- as an unsigned 32-bit integer count of milliseconds in the
format of {{durations}}. It is an estimate, not a bound; a quantity of
the link, not an end-to-end path delay; and independent of ETU and of
Generation Time. This document does not require receivers to combine it
with other metadata or otherwise compute an end-to-end delivery time;
how the estimate is used is a matter of endpoint and application policy.
application policy.

For perspective only, the maximum representable duration, if it were
pure propagation delay at the speed of light in vacuum, would
correspond to a distance of approximately 1.29 x 10^12 km, roughly
1.29 trillion km. The range is therefore very large even for the
long-delay environments that motivate this document. This observation
is explanatory; Expected Link Delay remains an estimated one-way link
delay comprising whatever components contribute to it.

### Denied by Link Policy {#dlp}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Length = 4           |  Class-Num =  |  C-Type = 4   |
|                               |     TBD5      |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-dlp title="Denied by Link Policy Object"}

The Denied by Link Policy object consists of the object header alone;
it has no payload, and its Length is exactly 4. It is carried only on
the policy carrier codes of {{carriers-policy}}.

Its presence asserts that the invoking packet was not forwarded
because the applicable admission policy for the managed long-haul
service refused it, independently of the current usability of any
link. Presence alone supplies this classification. There is no
Boolean, denial-reason field, allocation identifier, alternate
gateway address, or retry time, and this document defines no way to
convey why the traffic was refused or what would be admitted
({{uc-denied}}).

Absence of the object from an administrative prohibition error means
only that this classification was not supplied. It does not establish
that the refusal came from a firewall or other control, that no
boundary-link policy exists, or that the application has no
alternatives. A receiver that ignores extensions receives an ordinary
administrative prohibition error. An error conveys this classification
only when it actually contains a valid, applicable Denied by Link
Policy object; an object of any other length does not establish it
({{common}}).

Generating the object is optional and subject to disclosure policy
({{generation}}). Not every administrative prohibition error need
carry it, and its presence does not promise permanent denial, prove
that waiting could never help, require the receiver to change
mechanisms, or authorize evading policy.

## Object Applicability by Code {#applicability-objects}

Each Link Condition Object is applicable to one group of carrier
codes, as summarized in {{tab-applicability}}. Availability carriers
are ICMPv4 Type 3 codes 0 and 1 and ICMPv6 Type 1 codes 0 and 3
({{carriers-avail}}); policy carriers are ICMPv4 Type 3 code 13 and
ICMPv6 Type 1 code 1 ({{carriers-policy}}).

| Object | Availability carriers | Policy carriers |
|:-------|:----------------------|:----------------|
| Generation Time (C-Type 1) | MAY; SHOULD NOT without ETU | SHOULD NOT; no use defined here without ETU |
| Expected Time Until Link Usability (C-Type 2) | MAY, subject to {{ltu}} | MUST NOT |
| Expected Link Delay (C-Type 3) | MAY, subject to {{ltu}} | MUST NOT |
| Denied by Link Policy (C-Type 4) | MUST NOT | MAY, subject to {{generation}} |
{: #tab-applicability title="Link Condition Object Applicability by Carrier Group"}

Expected Time Until Link Usability and Expected Link Delay reveal
characteristics of a constrained long-haul link to which the invoking
traffic was not admitted, and admission policy is intentionally
evaluated before any link-state information is disclosed
({{generation}}); a policy carrier MUST NOT carry either. A policy
carrier SHOULD NOT carry Generation Time, for which this document
defines no use without ETU; this is discouraged rather than
inapplicable, and a receiver that finds Generation Time on a policy
carrier simply has no defined use for it.
for it.

An availability carrier MUST NOT carry Denied by Link Policy, since an
admitted packet was by definition not refused by admission policy.
Consequently, under these rules Denied by Link Policy and the
availability metadata are never emitted in the same message.

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
otherwise require, and uncertainty about whether an optional object
can be supported never suppresses the underlying error.

On receiving trans-boundary traffic, a gateway proceeds as follows:

1. Assess the current coordinated forwarding and service state for
   the traffic, including whether a policy-permitted path within the
   coordination scope is presently available to serve it. If so, use
   that path and attach nothing. The coordination-scope restriction
   below governs attaching Link Condition Objects; it does not alter
   when ordinary Destination Unreachable errors are generated.

2. Apply the long-haul service's admission policy before disclosing
   any link state or prediction.

3. If the traffic is not admitted, drop it. Where ordinary rules and
   local response policy call for one, generate the appropriate
   administrative prohibition error ({{carriers-policy}}); where
   disclosure policy permits, attach the Denied by Link Policy object.
   Do not attach Expected Time Until Link Usability or Expected Link
   Delay ({{applicability-objects}}). Denial of admission to one
   long-haul service does not by itself mean no policy-permitted
   managed alternative exists; step 1 addresses that.

4. If the traffic is admitted but cannot be forwarded because the
   selected link is presently unusable, drop it and generate the
   Destination Unreachable error that ordinary ICMP rules call for. If
   the conditions of {{ltu}} are satisfied and the code is an eligible
   availability carrier ({{carriers-avail}}), the gateway MAY attach
   Expected Time Until Link Usability, Generation Time, and Expected
   Link Delay objects.

5. If the traffic is admitted and the link is usable, forward it
   normally. A forwarded packet MUST NOT elicit a Destination
   Unreachable error.

A gateway MUST NOT attach availability metadata when, according to
its current coordinated forwarding state, another policy-permitted
path within the coordination scope is presently available for the
traffic. A gateway MUST NOT attach any Link Condition Object whose
assertion it cannot establish from a sufficiently current and
applicable association between the traffic, the managed service, the
forwarding state, the admission policy, and any advertised prediction
({{orchestration}}); it omits that object and generates the ordinary
error under ordinary rules.

A gateway MUST NOT attach availability metadata on the basis of
observed local interface or carrier state alone. The transient
interruption it describes MUST derive from managed-service knowledge
of the link ({{applicability}}).

A gateway MUST NOT send an ETU value not derived from its current
managed-service knowledge. A gateway that does not know when the link
is expected to become usable omits the ETU object; it MUST NOT report
an expired estimate clamped to zero. A gateway MAY send ETU without
Generation Time; it SHOULD NOT send Generation Time without ETU
({{applicability-objects}}). If metadata cannot be generated, is
withheld by policy, or is out of range ({{durations}}), the gateway
sends the ordinary error without the affected objects.

Attachment of every Link Condition Object MUST be configurable per
policy. An operator MUST be able to configure, per source, prefix, or
policy class, whether an administrative prohibition error is generated
at all for refused traffic, the alternative being silent discard, and
independently whether the Denied by Link Policy object is attached to
it; withholding the object does not by itself mean suppressing the
base error. An operator SHOULD be able to configure omission of the
availability metadata objects ({{security}}).

A gateway MUST NOT attach Link Condition Objects to messages sent
toward the constrained long-haul link, and MUST NOT attach them in
response to traffic arriving from it ({{applicability}}).

The absence of these objects promises nothing. ICMP delivery is
unreliable and rate limited, and a gateway MUST NOT assume that a
sender received any particular notification; it generates a fresh
notification, subject to rate limiting, for each invoking packet that
meets the conditions above, using its current applicable knowledge.


# Receiver Processing {#receiver}

## Validation {#validation}

Ordinary validation and handling of a Destination Unreachable error
are governed by the receiver's existing ICMP and transport rules; this
document does not change them and does not impose the checks below on
bare errors.

Before acting on Link Condition Objects, a receiver MUST validate the
message as an ICMP error per {{RFC4443}} and the mitigations of
{{RFC5927}}: the invoking packet excerpt is matched against existing
connection or flow state, and objects on a message matching nothing
the receiver sent are disregarded. A receiver SHOULD disregard Link
Condition Objects on a message that arrives on an interface facing
outside its disclosure scope. Disregarding the objects for
experimental use does not require rejecting, or changing the ordinary
handling of, the otherwise valid base error.

The carrier code does not prove that a message originated inside the
domain, and a filter that acts on Type and Code alone cannot
selectively remove Link Condition Objects -- availability metadata or
Denied by Link Policy -- without also affecting ordinary errors of the
same code. A deployment that requires selective containment of the
objects at a boundary needs extension-aware enforcement; this document
does not specify one and does not mandate blanket filtering of the
carrier codes.

## Consumer Behavior {#consumers}

This section is informative. This document does not specify how a
host, transport, application, or management system reacts to the
objects, and no such reaction is required for interoperability.
Receipt of an unreachable error carrying availability metadata informs
the receiver that the domain models the path as transiently
unavailable and, where ETU is present, roughly when usability is
expected; receipt of an administrative prohibition error carrying
Denied by Link Policy informs it that boundary-link admission policy
refused the traffic. Receipt of either error without objects carries
only the ordinary meaning of its code. What follows are possibilities
that the objects enable and that the experiment of {{experiment}} is
intended to explore.

A host might record availability metadata to avoid immediately
repeating attempts likely to elicit the same error. How such state
would be keyed, how widely it would apply, how long it would live, how
it would be invalidated when the domain's knowledge changes, and how
it would interact with routing are all open questions. Because ETU is
an estimate reflecting the domain's coordinated state at generation
time, and the gateway cannot revoke a notification, any such state is
a hint and not an assurance of service at any later time.

A transport might use the metadata as an input to its timeout and
failure decisions, for example by comparing the expected wait against
a connection's tolerance. The remote endpoint receives no
corresponding notification and continues to run its own timers, which
bounds the usefulness of waiting. No TCP or QUIC behavior is defined
here, and the presence of objects does not change how a transport is
required to treat the underlying error.

An application might be given the condition and its objects so that it
can defer, fail fast, or reconsider its communication mechanism
({{uc-handoff}}). On learning that its present traffic was refused by
boundary-link policy, an application with separately provisioned
knowledge of an authorized application-layer gateway or bundle
service might choose that mechanism instead; the object is the trigger
to reconsider, and provisioning supplies the alternative and its
authorization. Whether an application can receive and correlate this
context at all, and whether the extra classification improves its
decisions relative to an ordinary administrative prohibition error,
are experiment questions. Whether an application interface should be
standardized is not addressed here.

Stronger reactions to unauthenticated ICMP information carry greater
risk ({{security}}); which reactions are worth their risk is itself an
experiment question.


# Legacy Host and Middlebox Behavior {#legacy}

Every message defined by this document is an existing Destination
Unreachable error, optionally carrying additional information. A
stack that does not process {{RFC4884}} extensions handles such a
message exactly as it handles the same error today; the objects are
invisible to it. This document does not promise identical behavior
across stacks or that any connection survives, only that the objects
do not alter the ordinary error a legacy stack sees.

For context, {{Section 3.9.2.2 of RFC9293}} classifies the
availability carriers (ICMPv4 Type 3 codes 0 and 1, ICMPv6 Type 1
codes 0 and 3) as soft errors for TCP and the administrative
prohibition codes among the hard errors, while noting widespread
implementation behavior that treats soft errors as hard errors during
connection establishment; {{RFC5461}} documents further divergence.
These classifications describe existing TCP processing. They are not
the basis on which this document selects carriers, they do not define
permanent versus temporary network failure, and this document does not
rely on any particular transport reaction, promise a uniform one, or
instruct a transport to abort so that an application will reconsider.

Firewalls and stateful middleboxes inside the deploying domain may
strip or discard messages carrying unfamiliar extension structures, or
may already filter some Destination Unreachable codes. This fails
safe but silently defeats the mechanism, so deployment includes
reviewing ICMP filtering policy on the paths between gateways and the
hosts they serve; {{RFC4890}} provides the ICMPv6 filtering framework.
Legacy hosts generate nothing new and are unaffected as senders.


# Operational Considerations {#operational}

Gateways SHOULD count generation of object-bearing Destination
Unreachable messages, including which Link Condition Objects were
included, and expose the counters through network management. Counts
of ordinary Destination Unreachable or administrative prohibition
errors do not by themselves identify this experiment's activity, since
the carrier codes are generated for other reasons as well. A gateway
attaching Denied by Link Policy knows which policy dimension refused
the traffic even though the object does not convey it; gateways MAY
log or report that detail through local operational telemetry, which
is where operators debugging a refused flow should look. This document
does not prescribe a management or telemetry mechanism. In a correctly
provisioned domain these objects are rare for authorized traffic, so
the counters serve as the divergence telemetry of {{uc-divergence}}.

The orchestration model of {{orchestration}} yields several
operational lessons. Gateways need their managed-service records to
carry revision and validity information so that stale records are
recognized rather than acted on. When a readiness estimate passes
without the link becoming usable, gateways should have a supported new
estimate or omit ETU; a countdown clamped to zero is not an estimate.
Loss of controller connectivity is expected, and a gateway should
continue to use a still-valid record until it expires or local
evidence contradicts it, while ordinary error generation continues
regardless. During routing migrations, the record consulted for a
notification must be the one applicable to the traffic's current
selected exit; describing a different opportunity is worse than
attaching nothing. Operators should expect notifications to be less
informative during transitions and after plan revisions, and should
treat that as normal rather than as a fault.

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
Using Generation Time additionally depends on the receiver knowing the
enclave time reference and being able to relate it to its own clock
({{gen-time}}); receivers that cannot may apply ETU from receipt
({{etu}}). The deployment is responsible for establishing and
distributing the reference. When the reference changes or a gateway's
clock is reset, operators need to ensure that receivers do not
interpret timestamps generated under the old reference using the new
one; until that is assured, receivers fall back to ignoring Generation
Time. Cooperating administrative domains need a shared reference or a
known conversion before their receivers compare timestamps; sharing an
orchestration function does not by itself provide either.

Hosts that maintain state derived from these objects SHOULD expose it
to local diagnostics, since such state changes transmission behavior
in ways otherwise invisible to troubleshooting.


# Experimental Status and Goals {#experiment}

This document is published as Experimental. The objects are specified
normatively so that independent implementations interoperate at the
ICMP layer. The value of the mechanism, however, rests on two kinds of
questions that only deployment can answer: whether gateways can
generate the objects correctly and usefully, and whether the objects
let consumers make better decisions than the ordinary unreachable or
administrative prohibition error alone would allow. Consumer behaviors
are deliberately not standardized here and are not prerequisites for
interoperability; the experiment is intended to produce the experience
on which later specification of such behaviors might be based. An
extension object, like a new code, requires implementation support to
expose a new meaning to applications; neither inherently solves
application delivery.

## Scope of the Experiment {#exp-scope}

The experiment runs within administered limited domains, or
cooperating domains sharing a coordination and disclosure scope,
meeting the criteria of {{applicability}}: space networking testbeds
and missions, other networks with scheduled or intermittently
available managed links between enclaves, and laboratory emulations of
the same topologies. Link Condition Objects are not intended to be
exchanged across uncontrolled networks, though because they ride on
ordinary codes, containing them requires extension-aware enforcement
({{validation}}). No behavior in this document affects hosts whose
traffic never requires a trans-boundary link.

Prior to IANA assignment of the Link Condition Object class,
experiments use a class value from the Private Use range 247-255 of
the "ICMP Extension Object Classes and Class Sub-types" registry
{{RFC4884}}, with its interpretation and scope explicitly configured
among the participants. Such a value is not globally unique, is not a
claim on any permanent assignment, and is used only within the
experiment's coordination scope. The carrier codes are already
assigned and are used as is; no experimental ICMP type or code is
involved. Interoperability reports from this phase are in scope for
the experiment.

## Network Signal Evaluation {#exp-signal}

The first layer of the experiment concerns the objects themselves:

- Generation correctness: whether gateways attach each object only
  under the conditions of {{generation}}, whether the
  coordination-scope determination can be made reliably in deployed
  routing architectures and during transitions, and which carrier
  codes gateways actually produce for the long-haul conditions in each
  IP version ({{carriers}}).

- Distinction: whether availability metadata usefully distinguishes
  managed transient unavailability from ordinary unreachable errors of
  the same code, and whether the Denied by Link Policy object usefully
  distinguishes boundary-link admission refusal from other
  administrative prohibition ({{interaction}}).

- Prediction accuracy and usefulness: how closely ETU tracks actual
  link usability, how often it is available at all, how often plan
  revisions or acquisition failures invalidate it, and whether
  Generation Time and Expected Link Delay are populated and used.

- Orchestration feasibility: whether the information path of
  {{orchestration}} can be realized with existing management channels,
  and whether gateways can consult applicable records on the
  error-generation path, including after route withdrawal.

- Generation rate: gateway ICMP generation rates at gap onset under
  realistic fan-out, and the effect of rate limiting on which senders
  receive a notification.

- Legacy and middlebox behavior: how unmodified hosts respond to
  object-bearing carrier errors across the stacks present in target
  environments, including embedded and flight-heritage
  implementations ({{legacy}}); and what filtering changes deployments
  required, including whether middleboxes strip or discard extension
  structures.

- Security and disclosure: whether the information disclosed by the
  objects is operationally acceptable, whether the disclosure controls
  of {{generation}} are sufficient, and whether forged or stale
  objects caused harm.

- Whether a coarse boundary-policy object is useful beyond the
  ordinary administrative refusal, and whether experience establishes
  a need for richer policy-denial metadata. This document does not
  speculate about the format of any such metadata.

## Consumer Behavior Exploration {#exp-questions}

The second layer concerns what receivers do with the objects. None of
these behaviors is specified here; the experiment is intended to
discover which are worthwhile:

- Whether hosts that record availability metadata to avoid repeated
  attempts reduce futile retransmission and offered load during
  link-unavailable periods, relative to hosts receiving the same
  ordinary errors without objects, and what keying, scope, lifetime,
  and invalidation rules work.

- What transport reactions are appropriate, including whether
  comparing expected wait against connection tolerance is useful and
  whether connections that wait survive the peer's own timers.

- Whether applications can receive and correlate the objects with
  their traffic at all, and whether the Denied by Link Policy
  classification improves their decisions relative to an ordinary
  administrative prohibition error, including selection among
  separately provisioned mechanisms such as IP transports,
  application-layer gateways, and bundle services.

- Whether provisioning and orchestration systems can use the objects,
  and whether divergence telemetry ({{uc-divergence}}) exposed real
  faults ahead of existing monitoring.

## Evidence for Advancement {#exp-criteria}

Later consideration of the mechanism for the standards track could be
informed by evidence of: at least two independent interoperable
implementations of the Link Condition Objects, including extension
object processing on both carrier groups; generation accuracy in at
least one operational, non-laboratory domain; usefulness of the
objects beyond the ordinary errors they ride on, with unused objects
removed rather than carried forward; manageable legacy and middlebox
behavior; acceptable security and disclosure properties; consumer
behaviors independently demonstrated to benefit from the objects,
without presupposing any one receiver strategy; and evidence that the
semantics apply across more than one class of long-haul network
without special-case changes to the wire format. Reports of failed or
inconclusive experiments are requested to the same degree as
successful ones.


# Security Considerations {#security}

ICMP carries no authentication, and the objects are actionable by
implementations that choose to act on them, so the principal threats
are forgery by off-path attackers and information disclosure to the
senders these messages answer. The mitigations are the scoping of
{{applicability}}, validation against sender state ({{validation}}),
rate limiting, and agreed disclosure policy among the cooperating
authorities. These are mitigations, not authentication: none of them
authenticates the carrier code or the objects, filtering and
invoking-packet validation raise the cost of injection without
confining it to trusted parties, and none is dependable across the
open internet.

Three forgery cases arise. A forged ordinary Destination Unreachable
or administrative prohibition error is an existing threat with
existing mitigations ({{RFC5927}}); this document does not change it.
Forged availability metadata on an otherwise plausible error could
induce an implementation that acts on it to withhold trans-boundary
traffic for the advertised interval. A forged Denied by Link Policy
object could induce an application to abandon a working IP path for
another mechanism, or to conclude that provisioning is needed; this
influence on application choices is a capability beyond what forging
the bare administrative prohibition error provides. This document does
not require receivers to act on any object, and stronger reactions to
unauthenticated ICMP information carry correspondingly greater risk;
an implementation that does act should bound the effect of any single
notification, treat ETU as an estimate that may be stale or wrong,
treat Denied by Link Policy as a classification that may be forged,
and consider that the gateway cannot revoke a notification whose basis
has changed.

Because the objects ride on ordinary codes, a boundary filter on Type
and Code alone cannot contain either availability metadata or Denied
by Link Policy without also blocking ordinary errors of the same code
({{validation}}). Deployments that require such containment need
extension-aware enforcement, which this document does not specify.
The absence of Link Condition Objects from a message is not a
trustworthy indication that no long-haul condition exists, since
objects may be omitted by policy, by rate limiting, by middleboxes, or
by an attacker.

The Denied by Link Policy object, although it has no payload,
discloses that boundary-link admission policy refused particular
traffic, and a gateway that attaches it is a probing oracle: an
attacker varying markings, sources, and destinations can map which
traffic the long-haul service admits, even without being told why any
particular packet was refused. {{generation}} therefore requires
per-policy control over whether the object is attached and,
independently, whether the base error is generated at all; toward any
sender the operator does not trust, silent discard is the expected
posture, consistent with existing firewall practice.

The availability metadata objects disclose link schedule information,
which in some deployments is sensitive operational information.
{{generation}} permits omitting them per policy; operators of such
deployments should protect the metadata as they protect the underlying
schedule. Evaluating admission before link state ({{generation}})
prevents disclosure of schedule information to traffic that would not
be admitted regardless.

Ordinary ICMP generation restrictions and rate limiting
({{Section 2.4 of RFC4443}}, {{RFC1812}}) apply to all of these
messages. Quoting and padding of the invoking packet and the optional
objects affect response size; a message carrying all three
availability objects is larger than a minimal Destination Unreachable
error. This document makes no claim about the absence of amplification
beyond what those restrictions provide.


# IANA Considerations {#iana}

This document requests no new ICMP type and no new Destination
Unreachable code. It requests one assignment, shown as a TBD value
throughout; no numeric value is proposed.

From the "ICMP Extension Object Classes and Class Sub-types" registry
{{RFC4884}}: a new class TBD5, Link Condition Object, with C-Type 1,
Generation Time; C-Type 2, Expected Time Until Link Usability; C-Type
3, Expected Link Delay; and C-Type 4, Denied by Link Policy, assigned
by this document. Further C-Types are Specification Required. The
class registry's procedure is First Come First Served for values
0-246.

This document requests no change to the already-assigned Destination
Unreachable codes on which Link Condition Objects may be carried:
ICMPv6 Type 1 codes 0, 1, and 3, and ICMPv4 Type 3 codes 0, 1, and 13
({{carriers}}). Their existing meanings are unchanged.


--- back

# Worked Examples {#example}

## Availability Metadata {#example-avail}

Times in this example are expressed in the enclave time reference
({{gen-time}}) as seconds from its epoch. A gateway whose deep-space
link is presently unusable receives an IPv6 packet for a
trans-boundary destination at reference time 10,000 s. According to
its current coordinated forwarding state, no other policy-permitted
path within the coordination scope is available for the traffic. The
packet is admitted under the service's policy. The gateway retains a
route toward the destination, but the next hop over the deep-space
link is unusable; under {{Section 3.1 of RFC4443}} this is a
link-specific problem not covered by another code, so the gateway
generates ICMPv6 Destination Unreachable, Code 3 (Address
unreachable). Its managed-service record ({{orchestration}}) shows the
link expected to be ready to forward at reference time 12,520 s, and
its one-way delay estimate for the link is 20 minutes, so it attaches
three Link Condition Objects. The Class-Num octet of each is TBD5 and
is omitted from the listing.

| Object | C-Type | Length | Wire value | Decoded |
|:-------|-------:|-------:|:-----------|:--------|
| Generation Time | 1 | 12 | 0x00002710 0x00000000 | 10,000 s in the enclave reference |
| Expected Time Until Link Usability | 2 | 8 | 0x00267360 | 2,520,000 ms (42 min) |
| Expected Link Delay | 3 | 8 | 0x00124F80 | 1,200,000 ms (20 min) |
{: #tab-example title="Availability Metadata Example"}

The Generation Time seconds word is 10,000 and the fraction word is
zero. ETU is the interval from reference time 10,000 s to 12,520 s.
If this deployment had chosen UTC as its enclave time reference, the
seconds word would instead be the corresponding count of seconds from
the epoch the deployment specified; that is the deployment's choice
and is not required by this document.

Had the gateway attached no objects -- because it had no estimate,
withheld it by policy, or could not establish an applicable
managed-service association -- the receiver would have an ordinary
Address Unreachable error, meaning only that the packet was not
delivered. A receiver that does not process {{RFC4884}} extensions
sees exactly that in either case.

A receiver that processes the objects but does not know the enclave
time reference, cannot relate it to its own clock, or did not receive
Generation Time needs no timestamp arithmetic: it knows that, as of
generation, the link was expected to become usable in 42 minutes, and
it may apply that interval from the moment of receipt. If the
notification spent time in delivery, this interpretation expires later
than the original forecast; it remains a conservative reading of the
same forecast, not a guarantee, and the link may in fact become usable
later than either.

A receiver that knows the enclave time reference and can relate it to
its own clock may compute Generation Time + ETU = 12,520 s and, on
receiving the message at reference time 10,003 s, understand that
about 2,517 s remain, subject to its own clock uncertainty and to the
accuracy of the forecast.

Either receiver might, as a matter of local policy, use the 20-minute
link delay estimate in deciding whether the destination suits its
traffic once the link is usable. Whether either receiver does anything
at all with the information is not specified by this document.

## Denied by Link Policy {#example-dlp}

The same gateway, during an active contact, receives an IPv6 packet
whose traffic class is not admitted by the service's policy. It drops
the packet. Local response policy calls for an error, so it generates
ICMPv6 Destination Unreachable, Code 1 (Communication with destination
administratively prohibited), and, disclosure policy permitting,
attaches a single Link Condition Object:

| Object | C-Type | Length | Payload |
|:-------|-------:|-------:|:--------|
| Denied by Link Policy | 4 | 4 | none |
{: #tab-example-dlp title="Denied by Link Policy Example"}

The object occupies one 32-bit word: Length 0x0004, Class-Num TBD5,
C-Type 0x04. It is preceded by the {{RFC4884}} extension header and
by the invoking packet, padded to at least 128 octets. No Expected
Time Until Link Usability or Expected Link Delay is attached, and
Generation Time is omitted because no ETU accompanies it.

A receiver that processes the object learns that boundary-link
admission policy refused the traffic and may, for instance,
reconsider its communication mechanism ({{uc-handoff}}). A receiver
that ignores extensions, or that strips or disregards the object, has
an ordinary administrative prohibition error, meaning only that
communication with the destination was administratively prohibited by
some control; the object's removal restores exactly that meaning and
nothing else.


# Acknowledgments
{:numbered="false"}

The author used AI-assisted tooling in preparing this document,
including drafting and editing text and converting it to the
kramdown-rfc source format. All technical content, design decisions,
and the final text are the responsibility of the author.
