# app-chart

Helm chart with Deployment, Service, ServiceAccount, HPA, PDB, ServiceMonitor, External Secrets (SecretStore + ExternalSecret), Certificate (cert-manager) and Ingress (Traefik).

## Resources

| Resource | Description | Enabled by |
|---------|-------------|-----------|
| **Workload** | **Deployment** (default) or **Argo Rollouts** (Blue Green) | `workload.type` |
| **Deployment** | Optional envs; optional resources | When `workload.type: deployment` |
| **Rollout** | Argo Rollouts Blue Green (active + preview services) | When `workload.type: argorollout` |
| **Service** | One service (Deployment) or two: `-active` and `-preview` (Rollout) | Always |
| **ServiceAccount** | With optional annotations | `serviceAccount.create: true` |
| **HPA** | Horizontal Pod Autoscaler | `hpa.enabled: true` |
| **PDB** | Pod Disruption Budget | `pdb.enabled: true` |
| **ServiceMonitor** | Prometheus Operator | `serviceMonitor.enabled: true` |
| **SecretStore + ExternalSecret** | External Secrets Operator; secret used in the Deployment via secretRef | `externalSecrets.enabled: true` |
| **Certificate** | cert-manager (used by Ingress in tls.secretName) | Always created |
| **Ingress** | Main Ingress (points to active service or single service) | Always created |
| **Ingress preview** | Ingress for preview environment (Blue Green); backend = `-preview` service | When `workload.type: argorollout` and `workload.previewHosts` is set |

**Resource names:** Ingress name, Certificate name, ServiceAccount name, Service name, Deployment name and other resources inherit from the chart name: use **nameOverride** (and optionally **fullnameOverride**) in `values`; when the `name` fields are empty, the template uses the chart full name (fullname).

## Envs in the Deployment

- **env.enabled**: enables the use of environment variables.
- **env.fromSecretRef** / **env.fromConfigMapRef**: Secret/ConfigMap names to load **all** their contents as env (`envFrom`).
- **env.secretKeyRefs** / **env.configMapKeyRefs**: list of specific keys (env name, Secret/ConfigMap name, key).
- **env.literals**: literal name/value pairs.

When **externalSecrets.enabled** and **externalSecrets.useInDeployment** are `true`, the Secret created by the ExternalSecret is added to the workload through `envFrom.secretRef`.

## Workload type: Deployment or Argo Rollouts (Blue Green)

Use **workload.type**: `deployment` (default) or `argorollout`.

- **deployment**: one Deployment and one Service; Ingress points to this service.
- **argorollout**: one Rollout (Argo Rollouts) with Blue Green strategy, two Services (`-active` and `-preview`), main Ingress pointing to `-active` and, optionally, **workload.previewHosts** for a second Ingress that points to `-preview` (to test the new version before promoting). The HPA and PDB apply to the Rollout. For TLS on the preview Ingress, include the hosts in **certificate.dnsNames**.

## External Secrets

1. Set **externalSecrets.secretStore** with the `provider` (e.g.: `aws`, `vault`).
2. Set **externalSecrets.externalSecret.data** with `secretKey` and `remoteRef`.
3. The generated Secret is named **externalSecrets.secretName** (default: `{fullname}-externalsecret`) and is referenced in the Deployment when **useInDeployment** is `true`.

## Certificate and Ingress

Certificate and Ingress are always created by the chart. The **hostnames (DNS)** are provided in a single place: **certificate.dnsNames** — used in both the Certificate and the Ingress (TLS + rules). The Ingress inherits the Certificate secret in **tls.secretName**. Path `/`, pathType Prefix and Ingress backend are fixed in the template. Traefik annotations (ingress.class, router.entrypoints, router.tls) are fixed; use **ingress.extraAnnotations** to add more.

## Installation

```bash
helm install my-app . -f values.yaml -n my-namespace
```

## Minimal example with Certificate + Ingress

```yaml
nameOverride: my-app
replicaCount: 1
image:
  repository: myregistry/my-app
  tag: "1.0.0"

service:
  port: 8080
  targetPort: 8080

certificate:
  secretName: my-site-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:  # single place for hostnames; used in both Certificate and Ingress
    - "api.mysite.com"

ingress:
  extraAnnotations: {}
```
