# CRD-based ztunnel Enrollment via ClusterSPIFFEID

**SIG: SIG-ServiceMesh** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2026-10-08

**Cilium Release:** TBD

**Authors:** Ethan Ermovick <eermovick@microsoft.com>, Tamilmani Manoharan <tamanoha@microsoft.com>

**Status:** Draft

**Related CFP:** [Mutual Authentication for Service Mesh][cfp-mutual-auth]

## Summary

Currently, the Cilium Operator registers SPIRE entries imperatively through the SPIRE Entry API, and Cilium's bundled SPIRE server stores them in a single SQLite database. With this setup, we cannot have multiple redundent databases for SPIRE registration entries.

In this CFP, we propose storing registration intent declaratively. Instead of storing registration entries directly via the Cilium Operator, we adopt the `spire-controller-manager` setup from [todo]. Upon installation, helm creates a singular `ClusterSPIFFEID` custom resource that defines registration intent. In each spire pod, a `spire-controller-manager` sidecar container reconciles the registration intent with its database. Multiple redundent spire server pods can then reconcile from the single `ClusterSPIFFEID` for high availbitiy.

## Motivation

Currently, the Cilium Operator directly calls the Spire entry API to upsert registrations into the Spire database. The Spire entry API only connects to a single database, so entries are not replicated. This is a single point of failure.

The `spire-controller-manager` from [link] provides a solution for this issue. Using this, multiple Spire databases can independently recreate the registration entries, using the intent from the `ClusterSPIFFEID` custom resource, and the current cluster state. Using this system, Spire agents can failover from one Spire pod to another.

## Goals

* Create a high availibity Spire mtls system for Cilium
* Gate the declarative registration behind `encryption.ztunnel.spire.useCRD=true` (default false)
* Leave the current SPIRE path unchanged when `encryption.ztunnel.spire.useCRD=false`
* Make no SPIRE Entry API calls from the Cilium Operator when the CRD mode is enabled
* Match the current behavior of creating SPIRE entries

## Non-Goals

* [todo]

## Proposal

### Overview

[todo diagram]
```mermaid

```

Currently, the Cilium Operator handles registration entries using the Spire Entry API to add entries to the Spire database. The operator updates the entries whenever a change in cluster state (new pods, new namespace enrolled in mtls, etc.) requires it. In this CFP, we propose [todo "removing Cilium Operator registration entry handling" or "adding an option to disable Cilium Operator registration entry handling"]. Instead, we propose making use of the `spire-controller-manager` from [todo link]. The SPIRE Controller Manager project adds multiple components we propose using, including the `ClusterSPIFFEID` and `ClusterStaticEntry` CRDs, and the `spire-controller-manager` sidecar container. Upon installation, helm creates a single `ClusterSPIFFEID` custom resource. This defines registration intent for user managed service accounts. The `ClusterSPIFFEID` provides a template for how registration entries should be created. Additionally, helm creates `ClusterStaticEntries` to store registration intent for static workloads. The `spire-controller-manager` sidecar container runs in the SPIRE server pods. It repeatedly reconciles the registration intent from the `ClusterSPIFFEID` and `ClusterStaticEntries` into a database using the SPIRE Entry API, asynchrounsly. This replaces all Cilium Operator interactions with the SPIRE database. Under this setup, multiple spire servers running `spire-controller-manager` in separate pods can independently create the same SPIRE registration entries, allowing for redundency.

### UseCRD [todo name] Flag

This feature is gated behind the helm flag [todo]. When set to false, there are no changes.

### One ClusterSPIFFEID defining registration intent and registration template

`ClusterSPIFFEID` [todo link] is a custom resource from the SPIRE Controller Manager Project that serves as both a selector for what pods should have SPIRE entries, and a template for those entries. Registrations created from the `ClusterSPIFFEID` are dynamic based on the cluster state, depending on what pods are in service accounts enrolled in mtls. For example:

```todo```

### ClusterStaticEntries for static serviceaccounts

`ClusterStaticEntries` are used for static SPIRE entries, and are set for the below workloads. This replaces bootstrapping by `cilium-init`. These represent SPIRE entries that do not change.

[todo list]

### SPIRE Controller Manager Sidecar

The `spire-controller-manager` sidecar container runs in each spire server pod. It repeatedly reads the `ClusterSPIFFEID` and `ClusterStaticEntry` resources and reconciles the registrations into a database using the SPIRE Entry API. For the registrations coming from the `ClusterSPIFFEID`, it selects pods based on the selectors in the `ClusterSPIFFEID`, and creates the registration dynamically using the template. For `ClusterStaticEntry` registration entries, it does not depend on the current pods or service accounts etc., and creates them solely based on the static entry resources.

## Impacts / Key Questions

### Impact: Dependency on SPIRE Controller Manager

This new option requires new custom resource defintions and container images. When this feature is enabled, the `ClusterStaticEntry` and `ClusterSPIFFEID` custom resources must be installed from the SPIRE Controller Manager project, as well as the `spire-controller-manager` sidecar. These are all new dependencies for Cilium, and would only be installed when this feature is enabled in helm.

### Key Question: High availability via SPIRE CRDs with multiple databases vs a single replicated database

This proposal suggests using declaratively stored custom resources for SPIRE registration intent, with the option for multiple independent databases reconciling the entries for redundancy. This is in contrast with a single database with built-in replication.

### Option 1:

This current proposal suggests storing SPIRE registration intent as custom resources, from which the `spire-controller-manager` can then rebuild the SPIRE entries. Doing this on multiple databases with multiple `spire-controller-manager` instances creates redundancy.

#### Pros

* Ability to have multiple redundant databases
* Multiple simple databases have less overhead than a single high-availability database

#### Cons

* Only SPIRE registration data is replicated, so agents would have to reattest upon failover

### Option 2:

Use a SPIRE database with built-in redundancy, such as PostgreSQL operating across multiple nodes.

#### Pros

* Redundancy of all data, rather than just registration data, so agents wouldn't have to reattest during failover

#### Cons

* More database configuration overhead

<!-- Reference links -->

[cfp-mutual-auth]: https://github.com/cilium/design-cfps/blob/main/cilium/CFP-22215-mutual-auth-for-service-mesh.md
[flag-register]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/client.go#L71-L73
[config-field]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/client.go#L46-L47
[new-client]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/client.go#L130-L147
[spire-client]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/client.go#L108-L120
[sa-pruner]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/client.go#L122-L127
[build-cse]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/crd_client.go#L107-L136
[upsert]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/crd_client.go#L151-L190
[entry-name]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/crd_client.go#L92-L105
[crd-prune]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/spire/crd_client.go#L242-L301
[reconciler-delete]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/reconciler/reconciler.go#L81-L119
[reconciler-prune]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/reconciler/reconciler.go#L121-L134
[prune-interval]: https://github.com/Arleee1/cilium/blob/5e402e4b4c5ae9662ff6cbda7d8d5aed8d1a7571/operator/pkg/ztunnel/cell.go#L60-L61
