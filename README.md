# OCI Helm Charts Mirror

This repository mirrors legacy HTTP Helm charts to OCI for the `shamubernetes` environment. Its layout, build pipeline, keyless Cosign signing, Renovate tracking, release behavior, and deprecation workflow are based on [`home-operations/charts-mirror`](https://github.com/home-operations/charts-mirror), audited at upstream revision `a3bc2bd56d454df1c4e839793953df6dd920aef1`.

> [!CAUTION]
> These are mirrors, not independent chart releases. Move consumers to an official upstream OCI artifact when one becomes available. A deprecated mirror remains published for six months before registry removal.

## Automation

Each `apps/*/metadata.yaml` file declares an upstream Helm repository and exact chart version.

- Pull requests fetch and package every changed chart without publishing it.
- Merges to `main` fetch, package, publish to GHCR, and sign the OCI artifact with GitHub Actions OIDC through Cosign.
- The self-hosted `shamubernetes` Renovate service discovers this repository automatically, tracks the upstream Helm indexes, opens update pull requests, and auto-merges chart-only updates after CI succeeds.
- The `Deprecate Chart` workflow removes metadata, merges the deprecation change, and publishes a six-month removal notice.
- The `Stale` workflow applies the same issue and pull-request lifecycle used by the reference repository.

The reference repository uses an organization GitHub App for release, pull-request, and issue mutations. This repository uses its scoped `GITHUB_TOKEN` for those repository-local operations and the existing `shamubernetes` Renovate GitHub App for dependency updates. No long-lived package or signing credential is required.

## Usage

### CLI

```sh
helm install ${RELEASE_NAME} --namespace ${NAMESPACE} oci://ghcr.io/shamubernetes/charts-mirror/${CHART_NAME} --version ${CHART_VERSION}
```

### Flux

> [!WARNING]
> Cosign verifies that this repository's workflow published an artifact. It does not establish that upstream chart contents are safe.

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: ${CHART_NAME}
  namespace: ${NAMESPACE}
spec:
  interval: 1h
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: copy
  ref:
    tag: ${CHART_VERSION}
  url: oci://ghcr.io/shamubernetes/charts-mirror/${CHART_NAME}
  verify:
    provider: cosign
    matchOIDCIdentity:
      - issuer: ^https://token.actions.githubusercontent.com$
        subject: ^https://github.com/shamubernetes/charts-mirror.*$
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ${RELEASE_NAME}
  namespace: ${NAMESPACE}
spec:
  interval: 1h
  chartRef:
    kind: OCIRepository
    name: ${CHART_NAME}
    namespace: ${NAMESPACE}
```

## Mirrored charts

| Artifact           | Upstream repository                                      | Initial version | Consumers                                 |
| ------------------ | -------------------------------------------------------- | --------------: | ----------------------------------------- |
| `ingress-nginx`    | `https://kubernetes.github.io/ingress-nginx`             |        `4.15.1` | Internal and external ingress controllers |
| `kuberhealthy`     | `https://kuberhealthy.github.io/kuberhealthy/helm-repos` |           `104` | Kuberhealthy                              |
| `plane-enterprise` | `https://helm.plane.so/`                                 |         `2.6.1` | Plane Enterprise                          |

Plane publishes `plane-enterprise` publicly from [`makeplane/helm-charts`](https://github.com/makeplane/helm-charts), including a provenance file for version `2.6.1`, but that repository does not declare a separate repository license. This mirror preserves the upstream chart unchanged except for deterministic repackaging into OCI.

## Adding a chart

1. Confirm the application does not publish an acceptable official OCI chart.
2. Add `apps/<artifact>/metadata.yaml` using the metadata schema in `.vscode/metadata-schema.json`.
3. Pin the exact upstream version and open a pull request.
4. Let pull-request CI fetch and package the chart.
5. After merge, verify the GHCR artifact and its keyless Cosign signature before changing any consumer.
