# Deployment

Everything runs on Kubernetes, installed with Helm charts from Docker Hub
(`oci://registry-1.docker.io/pnetrepo`). A full deployment is two Helm releases: the CAMARA
APIs and a core with its element manager.

```mermaid
flowchart TB
    internet(["API consumers"])

    subgraph rel_camara["camara chart"]
        ingress["Ingress<br/>one host: API paths, Swagger UI, Keycloak"]
        apis["CAMARA services<br/>(those enabled)"]
        kc["Keycloak"]
        redis[("Redis")]
    end

    subgraph rel_stack["open5gs-em-stack chart"]
        em["open5gs-em"]
        swap["open5gs-swap-events"]
        core["Open5GS NFs + MongoDB"]
    end

    gnb(["gNBs"]) -- "NGAP (NodePort)" --> core
    upf["UPF<br/>(own chart)"] -- PFCP --> core

    internet --> ingress
    ingress --> apis
    ingress --> kc
    apis --> redis
    apis -- "TEP, cluster-internal" --> em
    em --> core
    swap --> core
```

## The two releases

| Release | Chart | What it deploys |
|---|---|---|
| CAMARA APIs | `oci://registry-1.docker.io/pnetrepo/camara-chart` | Any set of the seven services, each from its own released chart. Also one Ingress host, one Swagger UI with every enabled API's spec, and Keycloak and Redis when needed, in the release or external. |
| Core + EM | `oci://registry-1.docker.io/pnetrepo/open5gs-em-stack` | Open5GS (AMF, AUSF, BSF, NRF, NSSF, PCF, SCP, SMF, UDM, UDR, WebUI, MongoDB), open5gs-em and open5gs-swap-events, wired by release name. The UPF has its own chart. |

The camara chart reaches TEP through `global.camara.tepBaseUrl`. That URL is the only link
between the two releases, and the only thing that changes when the core changes.

The stack keeps one list of quality profiles (`global.qualityProfiles`). open5gs-em serves
it, and the PCF's `qos_profiles` is rendered from it.

Both charts install on any cluster with their defaults. A deployment sets its own PLMNs,
storage class, NodePorts and Ingress host in its values file.

## Images, charts and releases

Every component is versioned and released on its own, from a tag. GitHub Actions builds and
publishes it to Docker Hub (`pnetrepo`), and creates a GitHub Release with its notes:

| Repository | Tag | Publishes |
|---|---|---|
| camara-tep | `<service>-vX.Y.Z` | Image `pnetrepo/<service>` and chart `pnetrepo/<service>-chart` |
| camara-tep | `camara-vX.Y.Z` | Chart `pnetrepo/camara-chart`, which pins one released version of each service chart |
| open5gs-em | `vX.Y.Z` | Image `pnetrepo/open5gs-em` and chart `pnetrepo/open5gs-em-chart` |
| open5gs-swap-events | `vX.Y.Z` | Image `pnetrepo/open5gs-swap-events` and chart `pnetrepo/open5gs-swap-events-chart` |
| open5gs-em-stack | `vX.Y.Z` | Chart `pnetrepo/open5gs-em-stack`, which pins one version each of the em and swap-events charts |

Upgrading a component in a deployment therefore takes two steps. Release the component,
then bump its pinned version in the umbrella chart (camara or stack) and release that.

## Trust boundaries

| Boundary | Protection |
|---|---|
| API consumers → CAMARA services | OAuth2 access tokens with the API's scope. Only the Ingress is exposed: API paths, spec files, Swagger UI and Keycloak. |
| Actuator (health, metrics) | Its own management port, which the Ingress never exposes |
| CAMARA services → TEP | **None.** TEP is cluster-internal and unauthenticated. It returns subscriber credentials, so it must never be exposed. |
| TEP → CAMARA QoD notifications | **None.** `/tep-notifications` is outside the Ingress's paths. |
| open5gs-em → Open5GS | Cluster-internal: MongoDB, PCF SBI and AMF/SMF metrics are ClusterIP |
| Outside → core | Only the AMF's NGAP (for gNBs) and the WebUI are NodePorts |

Phone numbers never appear in URLs on any hop, and the services mask them in logs.
