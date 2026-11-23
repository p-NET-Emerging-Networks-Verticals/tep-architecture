# The TEP contract

The TEP API is the one interface between the CAMARA services and a 5G core. Its spec,
[`tep-api.openapi.yaml`](https://github.com/pnet-software/tep-openapi/blob/main/tep-api.openapi.yaml)
in [tep-openapi](https://github.com/pnet-software/tep-openapi), is the contract. Every
element manager serves it, and every client is generated from it. The current version is
**TEP 1.0.0**.

## What it offers

| Resource | Operations | What for |
|---|---|---|
| **Subscribers** | `GET`, `POST /subscriber`; `GET`, `PUT`, `DELETE /subscriber/{id}` | Provisioning: identity (IMSI, phone number), credentials, and the data networks (DNN in a slice) a subscriber may use |
| | `POST /subscriber/retrieve` | Find a subscriber by phone number |
| | `POST /subscriber/status` | What the core knows now: connection state, serving network, whether it is roaming, PDU sessions and their QoS flows |
| | `POST /subscriber/location` | The cell the UE was last in, when, and that cell's coverage area |
| | `POST /subscriber/swaps` | When the phone number last moved to another SIM (IMSI) or another device (IMEI) |
| **Quality profiles** | `GET /quality-profile`, `GET /quality-profile/{name}` | The QoS levels the core offers, each realized by a 5QI. Read-only. |
| **Cells** | `GET`, `POST /cell`; `GET`, `PUT`, `DELETE /cell/{id}` | The radio cells and their coverage areas (a centre and a radius), which locations are answered with |
| **AS sessions with QoS** | `/3gpp-as-session-with-qos/v1/{scsAsId}/subscriptions[/{subscriptionId}]` | The 3GPP TS 29.122 AsSessionWithQoS API, unchanged: give a UE's traffic a quality profile |

Every path is relative to `{apiRoot}/tep-api/v1`.

## What the deployment provides

Part of what TEP answers isn't created through the API:

1. **Quality profiles** are configured with the deployment. Each has a `name` and a
   `fiveQi`, and the core's policy control must map every profile name to that 5QI. A
   change takes a restart of both.
2. **Cells** are registered through `POST /cell`. A location names the cell the core reports
   the UE in. Once that cell is registered, the location includes its coverage area.
3. **SIM and device swaps** need a component that watches the core's subscribers and
   records each change. Without it, `POST /subscriber/swaps` has nothing to answer.

## Conventions

- **No phone numbers or IMSIs in URLs.** Proxies, gateways and servers log URLs. Every
  lookup by phone number takes it in the body, as a `TepDevice` (`{"phoneNumber": "..."}`),
  the way CAMARA APIs take a `device`.
- **Opaque subscriber ids.** The `id` in `/subscriber/{id}` is assigned by the element
  manager. It stays the same through SIM swaps and number changes.
- **Correlation.** Every operation takes an optional `x-correlator` header, as CAMARA APIs
  do. It is echoed back, errors included, and the request is logged under it. The CAMARA
  services forward theirs, so one value follows a request from the consumer to the core.
- **Errors.** Every error is an RFC 9457 problem (`application/problem+json`). A 400 caused
  by one field can name it in a 3GPP `invalidParams` entry.
- **Paging.** List operations take `limit` (1 to 500, default 50) and `offset`, and answer
  with `items` and `total`.

## 3GPP AsSessionWithQoS inside TEP

Quality on demand needs the core to apply QoS to a flow. 3GPP already defines that API for
an application server (TS 29.122, the T8 reference point), so TEP mounts it as is, under its
own base path, instead of defining its own:

- `qosReference` names a TEP quality profile, which must be `ACTIVE`.
- `ueIpv4Addr` identifies the UE. The core finds its PDU session by it, so the UE must be
  connected.
- When the network ends a session, the application server is notified at
  `{notificationDestination}/notifications`.
- A session is changed with a JSON merge patch of `qosReference`, `flowInfo` or
  `notificationDestination`. PUT isn't supported.

The 3GPP operations are `$ref`s into
[`TS29122_AsSessionWithQoS_Bundle.openapi.yaml`](https://github.com/pnet-software/tep-openapi/blob/main/TS29122_AsSessionWithQoS_Bundle.openapi.yaml).
That file is the 3GPP spec (V17.8.0) with the components it uses from TS 29.122 CommonData,
TS 29.512, TS 29.514 and TS 29.571 copied in. Tools then resolve one file instead of a chain
of five, and the 3GPP content stays unchanged.

The spec is OpenAPI **3.0**, like the 3GPP files. In a 3.1 document, the generators inline
external references and the 3GPP models lose their names.

## Trust model

TEP is a **trusted internal service**. It has **no authentication**, and it returns
subscriber credentials (K, OPc) and identifiers (IMSI, MSISDN) in plaintext. It runs only
behind network-level access control, and is never exposed publicly. Authentication of API
consumers happens a layer up, in the CAMARA services. See [Deployment](deployment.md#trust-boundaries).

## Keeping the spec

- **One spec, identical copies.** tep-openapi holds the contract. Every element manager and
  camara-tep's `tep-client` (`src/main/openapi/`) keep byte-identical copies of both files,
  under the same names. A change to the spec is copied to all of them.
- **Generated on both sides.** Each element manager generates its server interfaces from the
  spec; camara-tep generates its client. A change that breaks either side fails that side's build.
- **Semantic versioning** through `info.version`: a change that can break clients takes a
  new major version.
