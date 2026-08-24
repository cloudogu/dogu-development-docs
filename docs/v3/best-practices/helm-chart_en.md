# Best Practices for Dogu-Helm-Charts

This document describes the best practices to follow when creating Dogu Helm charts.

## Take care when renaming Kubernetes resources

Changing `metadata.name` creates a different Kubernetes object. The impact depends on the resource and its controller: references can break, a controller can issue new credentials, and stateful resources can become disconnected or orphaned. A rename does not by itself mean that Kubernetes deletes the underlying data. If one or more of the above-mentioned issues occur, this can lead to a significant operational disruption.

Before changing a name, check these relationships:

| Relationship | Typical risk |
| --- | --- |
| PVC and claim references | A workload fails to start if the referenced claim does not exist. A new claim can bind different storage; whether the previous volume and its underlying storage remain after the old claim is deleted depends on the PersistentVolume reclaim policy. |
| Service, clients and selectors | Clients and `Exposition` resources can continue to reference the old Service; selector changes can also leave a Service without endpoints. |
| Exposition target | External access can point to a Service or port that no longer exists. |
| ServiceAccountRequest and managed Secret | A new request can regenerate credentials while the application still expects the old Secret. |
| Other integration resources | Authentication or menu integration can be duplicated, replaced or disconnected. |
| Backup and retention | Verify selection labels and restore or retention behavior for the affected resource. |

Treat renames of stateful or externally referenced resources as **migrations**: update every reference, verify backup and retention behavior, and retire the old object only after the new path has been validated.
