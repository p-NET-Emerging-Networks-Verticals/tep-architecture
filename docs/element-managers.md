# Element managers

An element manager (EM) serves the TEP API for one 5G core. It answers every TEP operation
from that core's own data and network functions. Everything core-specific lives in the EM,
and nothing core-specific lives anywhere else.

## What a core must provide

| TEP needs | Why |
|---|---|
| Read and write subscribers (IMSI, MSISDN, K, OPc, AMF, slices and DNNs) | Subscriber provisioning |
| The UE's registration and connection state, serving PLMN, and PDU sessions with their addresses and QoS flows | `POST /subscriber/status`, and so reachability, roaming and QoD |
| The UE's current cell (NCGI) and when it was reported | `POST /subscriber/location` |
| A record of IMSI and IMEISV changes per MSISDN | `POST /subscriber/swaps` |
| Policy control that applies a QoS level to an application flow, keyed by a QoS reference that maps to a 5QI | AS sessions with QoS |
| Notifications when the network ends such a flow | QoD's `NETWORK_TERMINATED` |

A core that lacks one of these can still serve TEP. The operations that depend on it answer
an error, and the CAMARA APIs built on them aren't available on that core.

## Implemented cores

| Core | Element manager | Details |
|---|---|---|
| Open5GS | [open5gs-em](https://github.com/pnet-software/open5gs-em) | [Open5GS implementation](open5gs.md) |

## Adding another core

1. **Create the EM** in its own repository, e.g. `<core>-em`. Copy the two spec files from
   [tep-openapi](https://github.com/pnet-software/tep-openapi) and generate the server
   from them.
2. **Map each operation** to the core's data and network functions, using
   [the Open5GS mapping](open5gs.md#where-each-answer-comes-from) as the reference. Operations the core can't support answer an error (unimplemented ones
   answer 501).
3. **Keep the conventions:** opaque subscriber ids, phone numbers only in bodies, the
   `x-correlator` echo, and RFC 9457 errors.
4. **Provide what the deployment needs:** quality profiles mapped to 5QIs in the core's
   policy control, and a swap recorder if the core keeps no history of IMSI and IMEI changes.
5. **Point the CAMARA services at it** through `TEP_BASE_URL` (the camara chart's
   `global.camara.tepBaseUrl`). Nothing in camara-tep changes.
