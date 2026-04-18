# unifi

![Version: 1.0.0](https://img.shields.io/badge/Version-1.0.0-informational?style=flat-square)
![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)
![AppVersion: 10.1.89-ls124](https://img.shields.io/badge/AppVersion-10.1.89-ls124-informational?style=flat-square)

The unifi software is a powerful, enterprise wireless software engine ideal for high-density client deployments requiring low latency and high uptime performance.

Runs the [linuxserver/unifi-network-application](https://hub.docker.com/r/linuxserver/unifi-network-application) container as a StatefulSet with persistent storage and MongoDB integration.

## Prerequisites

- Kubernetes >= 1.21
- Helm >= 3.0
- A running MongoDB instance accessible from the cluster
- MongoDB admin credentials stored in a Kubernetes Secret

## Installation

### 1. Create MongoDB credentials secret

The chart requires a secret containing MongoDB admin credentials for the init container to create the UniFi database user:

```bash
kubectl create secret generic unifi-db-credentials \
  --from-literal=admin-username=<your-mongo-admin-user> \
  --from-literal=admin-password=<your-mongo-admin-password>
```

### 2. Install the chart

```bash
helm install unifi oci://ghcr.io/qaoru/unifi --version 1.0.0 \
  --set database.host=<your-mongodb-host> \
  --set database.credentials.generate=true
```

Or with a local `values.yaml`:

```bash
helm install unifi oci://ghcr.io/qaoru/unifi --version 1.0.0 -f values.yaml
```

## Configuration

### MongoDB

The chart uses an init container (`mongo:7.0`) to automatically create the `unifi` database user and grant `dbOwner` roles on the required databases (`unifi`, `unifi_stat`, `unifi_audit`).

| Parameter | Description | Default |
|-----------|-------------|---------|
| `database.host` | MongoDB host | `"mongodb"` |
| `database.port` | MongoDB port | `27017` |
| `database.dbName` | Database name | `unifi` |
| `database.authSource` | Authentication source | `admin` |
| `database.credentials.generate` | Generate a secret with a random password | `false` |
| `database.credentials.secretName` | Secret name for DB credentials | `"unifi-db-credentials"` |
| `database.initCredentials.secretName` | Secret name for admin credentials | `"unifi-db-credentials"` |

### Services

Two service configurations are available:

- **internal** (ClusterIP, enabled by default) — exposes only the HTTPS UI (port 8443)
- **public** (LoadBalancer, disabled by default) — exposes all UniFi ports (HTTP inform, guest HTTP/HTTPS, speedtest, STUN, discovery, syslog)

### Ingress

Ingress is disabled by default. When enabled, it routes to the internal service on port 8443. TLS can be configured via the `ingress.tls` list.

### Network Policies

Supports both standard Kubernetes `NetworkPolicy` and `CiliumNetworkPolicy` (mutually exclusive). Select the flavor via `networkPolicy.flavor`: `kubernetes` or `cilium`.

### Persistence

Data is persisted at `/config` via a StatefulSet `volumeClaimTemplate` (default 5Gi). Set `persistence.enabled: false` to use an `emptyDir` instead.

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Affinity rules for pod scheduling |
| containerSecurityContext | string | `nil` | Security context for the main container (defaults to none) |
| database | object | `{"authSource":"admin","credentials":{"generate":false,"passwordKey":"password","secretName":"unifi-db-credentials","usernameKey":"username"},"dbName":"unifi","host":"mongodb","initCredentials":{"passwordKey":"admin-password","secretName":"unifi-db-credentials","usernameKey":"admin-username"},"port":27017}` | MongoDB connection settings |
| database.authSource | string | `"admin"` | MongoDB authentication source |
| database.credentials | object | `{"generate":false,"passwordKey":"password","secretName":"unifi-db-credentials","usernameKey":"username"}` | Credentials for the UniFi database user |
| database.dbName | string | `"unifi"` | MongoDB database name |
| database.host | string | `"mongodb"` | MongoDB host |
| database.initCredentials | object | `{"passwordKey":"admin-password","secretName":"unifi-db-credentials","usernameKey":"admin-username"}` | Admin credentials used by the init container to create the UniFi DB user |
| database.port | int | `27017` | MongoDB port |
| env | object | `{"PGID":"1000","PUID":"1000","TZ":"Europe/Paris"}` | Environment variables for the UniFi container |
| env.PGID | string | `"1000"` | Process group ID |
| env.PUID | string | `"1000"` | Process user ID |
| env.TZ | string | `"Europe/Paris"` | Timezone |
| extraEnv | list | `[]` | Additional environment variables as a list of {name, value} objects |
| extraVolumeMounts | list | `[]` | Extra volume mounts for the main container |
| extraVolumes | list | `[]` | Extra volumes for the pod |
| image | object | `{"pullPolicy":"IfNotPresent","repository":"linuxserver/unifi-network-application","tag":null}` | Image configuration |
| image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| image.repository | string | `"linuxserver/unifi-network-application"` | Container image repository |
| image.tag | string | `nil` | Container image tag (defaults to Chart appVersion) |
| ingress | object | `{"annotations":{"server-ssl":"true"},"className":"","enabled":false,"hosts":[{"host":"unifi.example.com","paths":[{"path":"/","pathType":"Prefix"}]}],"tls":[]}` | Ingress configuration |
| ingress.annotations | object | `{"server-ssl":"true"}` | Ingress annotations |
| ingress.className | string | `""` | Ingress class name |
| ingress.enabled | bool | `false` | Enable ingress |
| ingress.hosts | list | `[{"host":"unifi.example.com","paths":[{"path":"/","pathType":"Prefix"}]}]` | Ingress host rules |
| ingress.tls | list | `[]` | TLS configuration (e.g. [{ hosts: ["unifi.example.com"], secretName: unifi-tls }]) |
| initContainerSecurityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}` | Security context for the init container |
| metrics | object | `{"enabled":false,"unifiCredentials":{"generate":true,"passwordKey":"password","secretName":"unifi-metrics-credentials","usernameKey":"username"}}` | Metrics configuration |
| metrics.unifiCredentials | object | `{"generate":true,"passwordKey":"password","secretName":"unifi-metrics-credentials","usernameKey":"username"}` | generate monitoring credentials, you must configure it after pod bootstrap |
| networkPolicy | object | `{"cilium":{"egress":[{"toFQDNs":[{"matchName":"fw-update.ubnt.com"},{"matchName":"fw-download.ubnt.com"}]},{"toEndpoints":[{"matchLabels":{"k8s:io.kubernetes.pod.namespace":"kube-system","k8s:k8s-app":"kube-dns"}}],"toPorts":[{"ports":[{"port":"53","protocol":"UDP"},{"port":"53","protocol":"TCP"}],"rules":{"dns":[{"matchName":"fw-update.ubnt.com"},{"matchName":"fw-download.ubnt.com"}]}}]}],"ingress":[{"toPorts":[{"ports":[{"port":"http-inform"}]}]}]},"egress":{},"enabled":false,"flavor":"kubernetes","ingress":[{"ports":[{"port":"http-inform","protocol":"TCP"}]}]}` | Network policy configuration |
| networkPolicy.cilium | object | `{"egress":[{"toFQDNs":[{"matchName":"fw-update.ubnt.com"},{"matchName":"fw-download.ubnt.com"}]},{"toEndpoints":[{"matchLabels":{"k8s:io.kubernetes.pod.namespace":"kube-system","k8s:k8s-app":"kube-dns"}}],"toPorts":[{"ports":[{"port":"53","protocol":"UDP"},{"port":"53","protocol":"TCP"}],"rules":{"dns":[{"matchName":"fw-update.ubnt.com"},{"matchName":"fw-download.ubnt.com"}]}}]}],"ingress":[{"toPorts":[{"ports":[{"port":"http-inform"}]}]}]}` | Cilium-specific egress and ingress rules (used when flavor is "cilium") |
| networkPolicy.egress | object | `{}` | Standard Kubernetes NetworkPolicy egress rules |
| networkPolicy.enabled | bool | `false` | Enable network policy creation |
| networkPolicy.flavor | string | `"kubernetes"` | Policy flavor: "kubernetes" or "cilium" |
| networkPolicy.ingress | list | `[{"ports":[{"port":"http-inform","protocol":"TCP"}]}]` | Standard Kubernetes NetworkPolicy ingress rules |
| nodeSelector | object | `{}` | Node selector for pod scheduling |
| persistence | object | `{"accessModes":["ReadWriteOnce"],"annotations":{},"enabled":true,"size":"5Gi","storageClass":""}` | Persistence configuration for /config |
| persistence.accessModes | list | `["ReadWriteOnce"]` | Access modes for the PVC |
| persistence.annotations | object | `{}` | Annotations for the PVC |
| persistence.enabled | bool | `true` | Enable persistent storage |
| persistence.size | string | `"5Gi"` | PVC size |
| persistence.storageClass | string | `""` | Storage class (empty string uses default) |
| podAnnotations | object | `{}` | Additional pod annotations |
| podLabels | object | `{}` | Additional pod labels |
| podPorts | object | `{"discovery":{"containerPort":10001,"protocol":"UDP"},"guest-http":{"containerPort":8880,"protocol":"TCP"},"guest-https":{"containerPort":8843,"protocol":"TCP"},"http-inform":{"containerPort":8080,"protocol":"TCP"},"https-ui":{"containerPort":8443,"protocol":"TCP"},"speedtest":{"containerPort":6789,"protocol":"TCP"},"stun":{"containerPort":3478,"protocol":"UDP"},"syslog":{"containerPort":5514,"protocol":"UDP"}}` | Container port definitions |
| podSecurityContext | object | `{"fsGroup":1000,"fsGroupChangePolicy":"OnRootMismatch","runAsGroup":1000,"runAsUser":0}` | Pod-level security context |
| probes | object | `{"liveness":{"enabled":true,"failureThreshold":6,"initialDelaySeconds":20,"periodSeconds":15,"timeoutSeconds":5},"readiness":{"enabled":true,"failureThreshold":12,"initialDelaySeconds":20,"periodSeconds":10,"timeoutSeconds":5}}` | Liveness and readiness probe configuration |
| replicaCount | int | `1` | Number of replicas |
| resources | object | `{"limits":{"memory":"2Gi"},"requests":{"cpu":"200m","memory":"256Mi"}}` | Resource requests and limits |
| revisionHistoryLimit | int | `3` | Number of revisions to keep in history |
| serviceAccount | object | `{"annotations":{},"automount":false,"create":true,"name":""}` | Service account configuration |
| serviceAccount.annotations | object | `{}` | Annotations for the service account |
| serviceAccount.automount | bool | `false` | Whether to automount the service account token |
| serviceAccount.create | bool | `true` | Whether to create a service account |
| serviceAccount.name | string | `""` | Service account name (auto-generated if empty and create is true) |
| services | object | `{"internal":{"annotations":{},"enablePorts":{"discovery":false,"guest-http":false,"guest-https":false,"http-inform":false,"https-ui":true,"speedtest":false,"stun":false,"syslog":false},"enabled":true,"loadBalancerIP":"","type":"ClusterIP"},"public":{"annotations":{},"enablePorts":{"discovery":true,"guest-http":true,"guest-https":true,"http-inform":true,"https-ui":false,"speedtest":true,"stun":true,"syslog":true},"enabled":false,"loadBalancerIP":"","type":"LoadBalancer"}}` | Service definitions |
| services.internal | object | `{"annotations":{},"enablePorts":{"discovery":false,"guest-http":false,"guest-https":false,"http-inform":false,"https-ui":true,"speedtest":false,"stun":false,"syslog":false},"enabled":true,"loadBalancerIP":"","type":"ClusterIP"}` | Internal service (ClusterIP, exposes HTTPS UI only) |
| services.public | object | `{"annotations":{},"enablePorts":{"discovery":true,"guest-http":true,"guest-https":true,"http-inform":true,"https-ui":false,"speedtest":true,"stun":true,"syslog":true},"enabled":false,"loadBalancerIP":"","type":"LoadBalancer"}` | Public service (LoadBalancer, exposes all UniFi ports) |
| tolerations | list | `[]` | Tolerations for pod scheduling |
| unpoller | object | `{"extraEnv":[{"name":"UP_UNIFI_CONTROLLER_0_URL","value":"https://unifiunpoller:8443"},{"name":"UP_UNIFI_CONTROLLER_0_USER","valueFrom":{"secretKeyRef":{"key":"username","name":"unifi-metrics-credentials"}}},{"name":"UP_UNIFI_CONTROLLER_0_PASS","valueFrom":{"secretKeyRef":{"key":"password","name":"unifi-metrics-credentials"}}}]}` | Unpoller subchart configuration |
