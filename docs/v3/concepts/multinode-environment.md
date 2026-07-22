# Understand the Multinode Runtime Environment

A Dogu V3 is delivered as a Helm chart and runs as a namespaced application in a Multinode Cloudogu EcoSystem (CES). The chart describes the application workloads and may opt into CES capabilities through Kubernetes custom resources.

This page explains the resulting responsibilities and interactions. It is not a general Kubernetes or Helm tutorial.

## The mental model

![System context of a Dogu V3](../images/multinode-system-context.svg)

The responsibilities are deliberately split:

| Area | Dogu chart | CES platform |
| --- | --- | --- |
| Application | Defines Deployments, StatefulSets, Jobs, containers, probes and internal Services | Schedules and runs the resulting Kubernetes resources |
| Lifecycle | Supplies chart metadata, defaults, validation and optional migration logic | Resolves the chart, validates the requested transition and reconciles the Helm release |
| Configuration | Defines supported values, safe defaults and a schema | Supplies instance-specific values through the Dogu resource |
| Persistent data | Defines PVCs and volume mounts needed by the application | Provides the configured StorageClasses and binds volumes |
| External access | Defines a Service and, when needed, an `Exposition` | Reconciles gateway routes and external ports |
| Authentication | Defines an `AuthRegistration` and consumes its credential Secret | Registers the client with the identity provider and publishes credentials |
| Technical access to another CES service | Defines a `ServiceAccountRequest` to request access or a `ServiceAccountProducer` to offer access | Mediates credential creation and stores credentials in a Secret |
| Warp Menu | Defines zero or more `WarpMenuEntry` resources for internal application pages | Validates the entries and renders the shared menu |

The chart remains responsible for making the application work. CES controllers provide integrations; they do not infer missing application configuration from a container image.

## Namespace and Kubernetes context

A Dogu and its namespaced resources are installed together in the namespace selected by the CES. A chart must therefore be namespace-independent:

- use `.Release.Namespace` rather than hard-coding `ecosystem` or another namespace;
- derive names from `.Release.Name` and chart helper templates instead of assuming a fixed Helm release name;
- reference Services, ConfigMaps, Secrets and custom resources within the release namespace unless a documented API explicitly states otherwise; and
- never derive application behavior from a developer's local `kubectl` context.

When diagnosing a cluster, first verify the context and always make the namespace explicit:

```shell
kubectl config current-context
kubectl get dogu -A
kubectl -n <dogu-namespace> get pods,services,pvc
```

The Dogu Registry namespace in a name such as `official/my-dogu`, the OCI repository path and the Kubernetes runtime namespace are different concepts. Do not use one as a substitute for another.

## Helm release and Dogu lifecycle

The Dogu custom resource represents the platform's desired state. In a Blueprint-managed CES, change the Blueprint; the Blueprint Operator owns the derived Dogu resource. Modify a Dogu resource directly only when the deployment procedure explicitly identifies it as the source of truth. Do not modify the generated Helm release directly.

![Dogu V3 reconciliation and lifecycle](../images/multinode-lifecycle.svg)

The intended flow is:

1. A Dogu resource requests a specific Dogu/chart version.
2. The Dogu Operator resolves and validates the chart and its metadata.
3. Helm validates the merged values and installs or upgrades the release.
4. Kubernetes starts the chart workloads.
5. CES controllers reconcile integration resources independently.
6. The status fields on the Dogu and integration resources report progress or failure.

Reconciliation is continuous. A successful `helm template` or resource creation does not mean that the Dogu is ready. Workloads, PVCs and every required integration must also become ready.

### Upgrades

The Dogu version is the chart `version`; `appVersion` records the packaged application's version. The two versions may differ. In the accepted target architecture, `dogu-upgrade.yaml` describes valid transitions between Dogu/chart versions and the parameters that the Dogu Operator needs to coordinate an upgrade. A chart can supply migration logic through Kubernetes-native mechanisms such as init containers or Helm hook Jobs. Migration logic must be:

- safe to retry after a Pod or controller restart or when a Job runs again;
- explicit about the source and target Dogu/chart versions and relevant application versions;
- compatible with the chart's volume access modes;
- observable through useful Job logs and workload status.

The Dogu Operator must coordinate values and workload scaling when migration containers or hook Jobs need existing workloads or `ReadWriteOnce` volumes. A chart may implement migrations with init containers or Jobs started by Helm upgrade hooks. `dogu-upgrade.yaml` is the platform-level contract for valid version transitions; hook annotations alone do not replace that contract. The platform does not automatically execute V2 `pre-upgrade.sh` or `post-upgrade.sh` scripts.

### Reconciliation and status

Inspect status before inspecting controller internals:

```shell
kubectl -n <dogu-namespace> get dogu <dogu-name> -o yaml
kubectl -n <dogu-namespace> get exposition,authregistration,serviceaccountrequest,warpmenuentry
kubectl -n <dogu-namespace> describe <kind> <name>
```

The APIs do not share a common readiness condition. Check their API-specific success conditions:

- Exposition: `Valid`, `IngressesReady`, `NetworkPolicyReady`, `IngressTCPRoutesCreated`, `IngressUDPRoutesCreated` and `LoadBalancerPortsAllocated` as applicable;
- AuthRegistration: `Completed` and `CredentialsPublished`;
- ServiceAccountRequest: `ServiceAccountReady`; and
- WarpMenuEntry: `Ready` and `Visible`.

Use `metadata.generation` and the condition fields `status`, `reason`, `message` and `observedGeneration` where supported by the API. A resource that exists but does not report its API-specific success conditions is not successfully integrated yet.

## Configuration

Treat `values.yaml` as the chart's public configuration interface:

1. provide safe defaults so a normal installation needs few overrides;
2. document values that partners or instance operators are expected to change;
3. validate the final value structure with `values.schema.json`; and
4. use `dogu-values-metadata.yaml` for CES-wide mapped settings such as the root log level when required by the V3 contract.

Instance-specific overrides are supplied through the Dogu resource and merged by the platform. In a Blueprint-managed CES, define these overrides in the Blueprint instead of editing the derived Dogu resource. Do not require operators to patch Deployments, StatefulSets or Helm release Secrets after installation; reconciliation can overwrite those changes.

Passwords, tokens and private keys must not be committed to `values.yaml` or rendered into ConfigMaps. Consume a namespaced Kubernetes Secret or a credential Secret produced by a CES integration controller. The V3 architecture does not currently define one generic sensitive-values mechanism, so document each Secret contract in the chart.

## Storage and PVCs

V3 charts define their own Kubernetes PVCs and volume mounts. Unlike V2, a Dogu can use multiple volumes, for example, separate volumes for the application and database.

Document for each PVC:

- what data it contains,
- which workload uses it,
- the required access mode,
- the default size and
- whether `storageClassName` is configurable.

Do not hard-code a cluster-specific StorageClass. Define, document and test the retention behavior for persistent data when the Dogu is uninstalled.

## Service exposition and request routing

An application stays cluster-internal until the chart declares external access. Create a normal Kubernetes Service for the target workload and one or more [`Exposition`](https://github.com/cloudogu/k8s-exposition-lib) resources containing the required HTTP, TCP or UDP routes. Do not create a platform Ingress or Traefik resource directly.

![Request routing from a client to a Dogu](../images/multinode-request-routing.svg)

For HTTP routes, the Exposition identifies the target Service and port, the CES-relative path and an optional path rewrite. Design applications to work below a path such as `/my-dogu`; do not assume they run at `/`. Use TCP or UDP routes only when the application protocol cannot run through the HTTP gateway. Requested external ports can conflict; inspect `LoadBalancerPortsAllocated` and its reason to determine whether allocation succeeded.

The service-discovery controller owns the generated gateway resources. Helm tracks the Service and Exposition rendered by the Dogu chart; the chart must not patch the generated routes.

## Identity provider and authentication

To participate in CES single sign-on, declare an [`AuthRegistration`](https://github.com/cloudogu/k8s-auth-registration-lib) (`k8s.cloudogu.com/v1`). It specifies the protocol (`CAS`, `OIDC` or `OAUTH`), the consumer, and any optional logout URL and parameters.

The authentication controller registers the client and publishes credentials to a Secret. Mount or read that Secret in the application; never duplicate the credentials in chart values. Read `status.resolvedSecretRef` and the `Completed` and `CredentialsPublished` conditions when troubleshooting.

An AuthRegistration only provisions the client-side integration. The application must still implement the selected authentication protocol, construct CES-path-aware callback and logout URLs, and handle unavailable or rotated credentials.

## Two types of service account

### Kubernetes workload ServiceAccount

A Kubernetes `ServiceAccount` provides a Pod with an identity for Kubernetes API access. If the application requires this access, configure a namespaced ServiceAccount, set `serviceAccountName` on the workload and grant only the required Role permissions. Use cluster-wide RBAC only when explicitly required.

### CES service-account credentials

A [`ServiceAccountRequest`](https://github.com/cloudogu/k8s-serviceaccount-lib) (`k8s.cloudogu.com/v2`) requests technical credentials from another Dogu or component. The Service Account Operator provides them in a managed Secret.

The consuming application and its chart must:

- follow the producer's documented parameter and Secret-key contract;
- mount the credential Secret with `optional: true` when `ServiceAccountRequest.spec.optional` is `true`;
- ensure that the application reloads rotated credentials or restart the workload; and
- leave the managed Secret unchanged.

A Dogu that offers accounts must declare a `ServiceAccountProducer` and implement its configured producer strategy. The HTTP strategy is typically exposed through an adapter such as the [service-account-producer sidecar](https://github.com/cloudogu/service-account-producer-sidecar). Consult the [`ServiceAccountProducer` API](https://github.com/cloudogu/k8s-serviceaccount-lib/tree/develop/api/v2) for the complete contract. Containers in the same Dogu normally do not need this CES-wide mechanism. Inspect `ServiceAccountReady` when credentials are unavailable.

## Warp Menu

Declare a [`WarpMenuEntry`](https://github.com/cloudogu/k8s-warp-menu-entry-lib) (`k8s.cloudogu.com/v1`) for each internal user-facing entry point. A Dogu may define zero or more entries.

Each entry supplies German and English display names, a category key and a CES-relative path. External URLs are not allowed in `WarpMenuEntry` resources; the platform manages them centrally. Set `disabled: true` to retain a declaration without rendering it. Use the `Ready` and `Visible` status conditions to diagnose an entry that does not appear.

The Warp Menu does not expose the application. Its path must correspond to an independently working HTTP Exposition.

## Labels and selectors

Use the recommended Kubernetes application labels consistently on the chart resources:

```yaml
app.kubernetes.io/name: <dogu-name>
app.kubernetes.io/instance: <release-name>
app.kubernetes.io/version: <application-version>
app.kubernetes.io/managed-by: <release-service>
helm.sh/chart: <chart-name>-<chart-version>
```

Use only stable identity labels such as `app.kubernetes.io/name` and `app.kubernetes.io/instance` in workload selectors. Version and chart labels must not be part of a selector because their values change during upgrades.

Do not use general labels such as `app: ces` to select resources belonging to a specific Dogu. Add Dogu-specific labels only when required by a documented CES contract.

## Resource ownership

Every resource should have one clear manager:

- Define application resources and CES integration custom resources in the chart. Helm manages these resources.
- Integration controllers manage the resources they generate, such as gateway routes or credential Secrets. Do not include these generated resources in the chart or modify them manually.
- Adopt existing Secrets or PVCs only through a documented procedure.

Document and test the deletion behavior of persistent or externally managed resources.

## Pre-release checklist

Use this checklist before handing over a chart and when testing it in a partner CES:

- [ ] The chart renders with any release name and namespace, and `values.schema.json` validates its values.
- [ ] Workloads define probes, resource requests and limits, and stable selectors.
- [ ] Storage and uninstall behavior is documented and tested.
- [ ] External paths use Exposition resources, and Warp Menu paths work.
- [ ] Authentication and technical-account credentials are consumed from Secrets without being exposed.
- [ ] Every CES integration used by the Dogu reports a successful status.
- [ ] Installation, upgrades, reconciliation after configuration changes and uninstallation are tested against the targeted CES release.
