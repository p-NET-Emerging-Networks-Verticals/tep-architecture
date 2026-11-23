# Design decisions

The choices that shape this architecture, and why.

## The contract is an OpenAPI spec

The TEP API is one OpenAPI spec in [tep-openapi](https://github.com/pnet-software/tep-openapi).
Each element manager generates its server from it, and camara-tep generates its client.

**Why:** a spec is language-neutral, so an element manager for another core needn't be Java.
It can also be read, reviewed and versioned on its own. Generating both sides means a
breaking change fails a build instead of a request at runtime.

## The CAMARA services share one repository

[camara-tep](https://github.com/pnet-software/camara-tep) is one Maven reactor with shared
libraries (`tep-client`, `camara-common`) and one module per service. Each service has its
own version, tag, image and chart.

**Why:** what every service does the same way, like errors, auth, correlation, logging and
the TEP client, is written once. Each image contains exactly the library code of the commit
it was tagged on, and a change to shared code tests every service.

## OAuth2, as the CAMARA specs declare

Each service is an OAuth2 resource server enforcing the `openId` security of its CAMARA
spec, with scopes, and two- and three-legged tokens.

**Why:** it is what the CAMARA specs declare. A three-legged token also identifies the
subscriber, so the device can come from the token instead of the request.

## Phone numbers never in URLs

Every TEP lookup by phone number takes it in the request body (`TepDevice`), and subscribers
have an opaque id for `/subscriber/{id}`.

**Why:** proxies, ingresses and servers log URLs. A body stays out of access logs, and the
opaque id survives SIM swaps and number changes.

## QoS through 3GPP AsSessionWithQoS

TEP doesn't define its own QoS API. It mounts the 3GPP TS 29.122 AsSessionWithQoS API
unchanged, and quality profiles are referenced by name (`qosReference`).

**Why:** it is the standard northbound API of a NEF. A core with a NEF of its own can serve
it as is. For a core without one, the element manager acts as that NEF in front of the PCF
(as [open5gs-em](open5gs.md#qos-on-open5gs) does).

## The core decides roaming

TEP's subscriber status carries `roaming`. The element manager works it out from the home
PLMNs its deployment is configured with.

**Why:** home PLMNs belong to the core's deployment, not to an API on top of it.

## Swap history is a component of its own

Not every core keeps a history of IMSI or IMEISV changes. When it doesn't, a swap recorder
runs beside the element manager and records each change, and the EM only reads what it
records.

**Why:** the EM stays a reader of the core, and recording history is a separate job with its
own release cycle. A core that keeps the history itself needs no recorder. How the Open5GS
recorder works, and its trade-offs, is in [the Open5GS implementation](open5gs.md#repositories).

## Charts for any cluster

Every chart's defaults fit any Kubernetes cluster: the cluster's default storage class,
NodePorts the cluster assigns, and test PLMNs. A deployment's own settings, like PLMNs,
storage, ports and hosts, go in its values file.

**Why:** the same released chart installs in a lab or in production. Only the values differ.

## Not in scope

Simple Edge Discovery, One-Time Password SMS and Number Verification have no service.
