# Open5GS implementation

[Open5GS](https://open5gs.org/) is the first core with a TEP element manager. All seven CAMARA
APIs run on it. This page covers what's specific to that core. What any core must provide is
in [Element managers](element-managers.md).

## The picture

```mermaid
flowchart LR
    camara["camara-tep<br/>CAMARA services"]

    subgraph em5gc["Open5GS element manager"]
        direction TB
        em["open5gs-em<br/>serves TEP"]
        swap["open5gs-swap-events"]
    end

    subgraph core["Open5GS 5G core"]
        direction TB
        mongo[("MongoDB<br/>subscribers")]
        amf["AMF<br/>/ue-info"]
        smf["SMF<br/>/pdu-info"]
        pcf["PCF<br/>Npcf_PolicyAuthorization"]
    end

    camara -- "TEP API 1.0.0<br/>/tep-api/v1" --> em
    swap -- "polls subscribers,<br/>writes swap_events" --> mongo
    em -- "subscribers, cells,<br/>swap_events, qos_sessions" --> mongo
    em -- status, location --> amf
    em -- "PDU sessions, QoS flows" --> smf
    em -- "app sessions" --> pcf
    pcf -- "termination notifications" --> em
```

## Repositories

| Repository | What it holds |
|---|---|
| [open5gs-em](https://github.com/pnet-software/open5gs-em) | The TEP API on Open5GS |
| [open5gs-swap-events](https://github.com/pnet-software/open5gs-swap-events) | Records SIM and device swaps from Open5GS's subscribers |
| [open5gs-upf](https://github.com/pnet-software/open5gs-upf) | The Open5GS UPF as a chart of its own, associating to an SMF in another release |
| [open5gs-core-camara](https://github.com/pnet-software/open5gs-core-camara) | The demo: the Open5GS core, its UPF, open5gs-em, open5gs-swap-events and the CAMARA APIs in one release |

- **open5gs-em** is one Spring Boot app. Its server interfaces are generated from the TEP
  spec, whose copy it keeps in `src/main/resources/static/openapi/`. Its only northbound
  entrypoint is `/tep-api/v1`.
- **open5gs-swap-events** is the swap recorder Open5GS lacks. Every poll interval it compares
  each subscriber with its `swap_events` row, and records when the IMSI or IMEISV of an MSISDN
  last changed. Its timestamps are when a poll saw the change: late by at most one interval,
  never early. Polling works with any MongoDB deployment, a standalone one included. A change
  stream would give exact times, but it needs a replica set.
- **open5gs-upf** is Gradiant's UPF chart, adapted so the UPF starts the PFCP association
  with an SMF it's given, and so any Open5GS release can use it.
- **open5gs-core-camara** deploys everything above, plus the CAMARA APIs, as one Helm release,
  wired to each other.

## Where each answer comes from

| TEP operation | Open5GS source |
|---|---|
| Subscribers | `subscribers`, the collection the Open5GS WebUI writes, written in the same shape. The subscriber id is the document's `_id`. |
| Status | AMF `ue-info` (connection state, serving PLMN) and SMF `pdu-info` (PDU sessions, QoS flows), matched by SUPI. `roaming` compares the serving PLMN with the configured home PLMNs. |
| Location | The NR cell and timestamp in AMF `ue-info`, plus the coverage area from TEP's own `cells` collection |
| Swaps | `swap_events`, written by open5gs-swap-events |
| Quality profiles | The EM's configuration |
| Cells | TEP's own `cells` collection |
| AS sessions with QoS | PCF app sessions (Npcf_PolicyAuthorization), tracked in `qos_sessions` |

## QoS on Open5GS

open5gs-em acts as a 3GPP NEF for the AsSessionWithQoS API:

1. **qosReference to 5QI.** The session's `qosReference`, a quality profile name, is passed to
   the PCF, which maps it to a 5QI through its `qos_profiles` configuration. The demo chart
   renders that configuration from the same list of quality profiles the EM serves, so the
   two can't disagree. On an existing core, the two lists are kept in step by hand.
2. **PCC rule.** The PCF only authorizes a 5QI for which the UE's PDU session has a PCC
   rule. Before asking the PCF, the EM gives the session one if it has none. Open5GS limits
   a session to 8 PCC rules and 4 QoS flows (so at most 3 QoS sessions per UE), and the EM
   enforces both.
3. **Rates and ARP** come from the quality profile's fields and are sent in the media
   component. Rates go for GBR 5QIs only: a non-GBR QoS flow has no flow bit rates, and the
   PCC rule TEP gives a non-GBR 5QI has none either. A profile's ARP (`priority` through
   `reservation_priorities`, and its pre-emption fields) needs a PCF with AF-requested ARP,
   [Open5GS PR #4793](https://github.com/open5gs/open5gs/pull/4793) (`main`, or the release after
   2.8.0). Open5GS 2.8.0 ignores it and copies the PCC rule's ARP, which TEP sets to a safe
   fallback: priority level 15, no pre-emption of other flows, pre-emptable.
4. **Termination.** When the PCF ends an app session, it notifies the EM. The EM notifies
   the application server at `{notificationDestination}/notifications`.

## Deployment

There are two ways to run TEP on Open5GS:

- **The demo**, [open5gs-core-camara](deploy-demo.md): one release with the Open5GS core
  (AMF, AUSF, BSF, NRF, NSSF, PCF, SCP, SMF, UDM, UDR, WebUI, MongoDB), its UPF, open5gs-em,
  open5gs-swap-events and the CAMARA APIs, wired by release name. It keeps one list of quality
  profiles (`global.qualityProfiles`): open5gs-em serves it, and the PCF's `qos_profiles` is
  rendered from it.
- **An existing Open5GS** (2.8.0 or later): open5gs-em, open5gs-swap-events and the CAMARA APIs
  are installed next to it, from their own charts. See
  [Adding TEP to an existing Open5GS](deploy-existing-open5gs.md).

```mermaid
flowchart TB
    subgraph rel["open5gs-core-camara chart"]
        camara["CAMARA APIs"]
        em["open5gs-em (tep)"]
        swap["open5gs-swap-events"]
        core["Open5GS NFs + MongoDB"]
        upf["UPF"]
    end

    gnb(["gNBs"]) -- "NGAP (NodePort)" --> core
    gnb -- "GTP-U" --> upf
    camara -- "TEP, cluster-internal" --> em
    upf -- PFCP --> core
    em --> core
    swap --> core
```

Inside the cluster, TEP, MongoDB, the PCF SBI and the AMF/SMF metrics are ClusterIP. In the demo,
only the AMF's NGAP (for gNBs), the WebUI and the CAMARA Swagger UI are NodePorts.

### Releases

| Repository | Tag | Publishes |
|---|---|---|
| open5gs-em | `vX.Y.Z` | Image `pnetrepo/open5gs-em` and chart `pnetrepo/open5gs-em-chart` |
| open5gs-swap-events | `vX.Y.Z` | Image `pnetrepo/open5gs-swap-events` and chart `pnetrepo/open5gs-swap-events-chart` |
| open5gs-upf | `vX.Y.Z` | Chart `pnetrepo/open5gs-upf` (the image is Gradiant's `gradiant/open5gs`) |
| open5gs-core-camara | `vX.Y.Z` | Chart `pnetrepo/open5gs-core-camara`, which pins one version each of the em, swap-events, upf and camara charts |

Charts, releases and trust boundaries for the whole system are in [Deployment](deployment.md).
