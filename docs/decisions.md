# Design decisions

The choices that shape this architecture, and why.

## The contract is an OpenAPI spec

The TEP API is one OpenAPI spec in [tep-openapi](https://github.com/pnet-software/tep-openapi).
open5gs-em generates its server from it, and camara-tep generates its client.

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
it as is, and the Open5GS EM acts as that NEF in front of the PCF.

## The core decides roaming

TEP's subscriber status carries `roaming`. The element manager works it out from the home
PLMNs its deployment is configured with.

**Why:** home PLMNs belong to the core's deployment, not to an API on top of it.

## Swap history is a component of its own

Open5GS keeps no history of IMSI or IMEISV changes. open5gs-swap-events polls the
subscribers and records each change, and the EM only reads what it records.

**Why:** polling works with any MongoDB deployment, a standalone one included. The cost is
timestamps up to one poll interval late. A change stream would give exact times, but it
needs a replica set.

## Charts for any cluster

Every chart's defaults fit any Kubernetes cluster: the cluster's default storage class,
NodePorts the cluster assigns, and test PLMNs. A deployment's own settings, like PLMNs,
storage, ports and hosts, go in its values file.

**Why:** the same released chart installs in a lab or in production. Only the values differ.

## Not in scope

Simple Edge Discovery, One-Time Password SMS and Number Verification have no service.
