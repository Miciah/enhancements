---
title: gateway-api-crd-life-cycle-management
authors:
  - "@Miciah"
  - "@shaneutt"
reviewers:
  - "@shaneutt" # Gateway API maintainer.
  - "@dgn" # OSSM maintainer.
  - "@JoelSpeed" # SME for a similar issue and solution in CAPI.
  - "@knobunc" # Staff engineer for Networking, involved in past discussions.
approvers:
  - "@JoelSpeed"
  - "@knobunc"
api-approvers:
  - None
creation-date: 2025-01-22
last-updated: 2025-01-27
tracking-link:
  - https://issues.redhat.com/browse/NE-1946
status: provisional
see-also:
  - "/enhancements/ingress/gateway-api-with-cluster-ingress-operator.md"
---


# Gateway API CRD Life-Cycle Management

This enhancement describes how the [Ingress Operator](https://github.com/openshift/cluster-ingress-operator) manages Gateway API
CRDs.

## Summary

A key goal of Gateway API is to nurture a diverse ecosystem of implementations.
These implementations, including OpenShift's, all rely on the same Gateway API
CRDs.  To these ends, OpenShift must manage the life-cycle of these CRDs in a
way that enables Red Hat to support the product, minimizes friction for the
end-user, provides consistency, and avoids causing conflicts with other
implementations that rely on these CRDs.

## Motivation

Gateway API is an evolving API, and its CRDs change over time.  For core
OpenShift, we must ensure that we have a version of the CRDs that is compatible
with our product, and we must handle upgrading these CRDs for our own needs.  We
must also avoid unnecessarily causing friction for Red Hat layered products,
partner products, and other third-party implementations.

### User Stories

#### Using OpenShift's Gateway API implementation on a new OpenShift cluster

As a cluster-admin, I want to install a new OpenShift 4.19 cluster, and then use
OpenShift's Gateway API implementation to configure ingress to workload on this
cluster.

_TBD_

#### Using OpenShift's Gateway API implementation on an upgraded cluster

As a cluster-admin, I want to upgrade my cluster from OpenShift 4.18 (which
*doesn't* manage the Gateway API CRDs' life-cycle) to OpenShift 4.19 (which
*does*).

_TBD_

#### Using a third-party Gateway API implementation

As a cluster-admin, I want to install a third-party Gateway API implementation
on my OpenShift 4.19 cluster, and I want to use the third-party implementation
without interference from OpenShift.

_TBD_

#### Future OpenShift upgrades

As a cluster-admin, I want to upgrade my cluster from OpenShift 4.19 (which
manages the Gateway API CRDs' life-cycle) to OpenShift 4.20 (which includes a
newer version of these CRDs).

_TBD_

### Goals

- Ensure Gateway API CRDs are installed on new and upgraded OpenShift 4.19 clusters.
- Ensure the installed Gateway API CRDs are compatible with OpenShift's needs.
- Detect and warn if an incompatible version of the CRDs is installed.
- Describe the issue of dead fields and potential risks thereof.

### Non-Goals

- Automatically replace incompatible CRDs that some other agent installed.
- Provide a pre-upgrade check in OpenShift 4.18 for incompatible CRDs.
- Provide an explicit override for the cluster-admin to take CRD ownership.
- Implement an admission webhook to block updates to dead fields.
- Solve CRD life-cycle management in OLM.
- Solve OLM subscription management.
- Solve CRD life-cycle management for OSSM or Istio resources.

## Proposal

### CRD Life-cycle management

OpenShift's [Cluster Ingress Operator (CIO)] will manage the _entire_
life-cycle of the Gateway API CRDs from here onward. The CIO will provide a
[Validating Admission Policy (VAP)] to block updates from sources other than
the CIO, and will hold the CRDs at a specific `$CRD_VERSION`. In effect, this
means that the Gateway API resources will now be treated like core APIs.

> **Note**: the `$CRD_VERSION` selected for any OCP release version will be
> based on a corresponding release of [OpenShift Service Mesh (OSSM)] which is
> the implementation of Gateway API we will be using for first party Gateway
> API support on the cluster.

[Cluster Ingress Operator (CIO)]:https://github.com/openshift/cluster-ingress-operator
[Validating Admission Policy (VAP)]:https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/
[OpenShift Service Mesh (OSSM)]:https://github.com/openshift-service-mesh

#### CRD Deployment

The CIO will now check for the presence of the CRDs on the cluster, for
which there are two scenarios:

1. The CRDs are not present, so we create them
2. The CRDs are already present, so we need to take over management

The former is extremely straightforward: the CIO will deploy the VAP (to start
restricting who can manage the CRDs) and then will simply deploy the version of
the CRDs that meets the needs of the corresponding version of OSSM.

The latter situation has more complexities. We'll refer to this process as "CRD
Succession", and cover it's implications and logic below.

#### CRD Succession

Taking over the management of the Gateway API CRDs from a previous entity
can be turmultuous. Downtime, disruption and/or information loss can occur
during succession for reasons including (but not limited to):

* we will delete all experimental APIs (`TCPRoute`, `UDPRoute` and `TLSRoute`)
* we will delete old versions of APIs (e.g. `v1beta2.HTTPRoute`)
* we will drop all experimental fields (e.g. from `Gateway`, `HTTPRoute`)
* if the CRDs were version v0.x.x our upgrade to v1.x.x may break in-use fields
* other actors may be actively managing the CRDs and we will start blocking this

_TBD: think more about the L4 route situation. It's definitely simpler for us
      if we just remove them, but do we need to provide any additional provisions?
      talk with PMs about this too.

> **Note**: It is an unfortunate downside of how the Gateway API project in
> upstream delivers standard and experimental features on top of the same
> groups and kinds that leads us to need to do this heavier cleanup. We are
> tracking and supporting [an effort] to separate experimental into its own
> group in upstream, which we expect to help move towards a better overall
> experience for users who want experimental Gateway API features going
> forward.

Given the above: **we simply can not anticipate all of the negative effects**
succession will have on existing implementations. In order to make succession
a safe process, we will need **consent from a cluster admin for the platform to
take over management of any existing Gateway API CRDs**. That admin will be
responsible for ensuring the safety of existing implementions.

Because we require the consent as described above, we will **employ an Admin
Gate to request consent**. Using an Admin Gate has advantages:

* it requires a human to make a choice to accept succession of the CRDs, which
  neatly fills our requirement of receiving consent. It allows us to use the
  Admin Gate description to provide a very clear message about the implications
  of accepting the gate.
* as a required pre-upgrade check, it allows us to ensure that only one mode of
  Gateway API CRD management can be expected on clusters on 4.19 and beyond.
  This will provide consistency and reduce complexity for integrations, who can
  simply rely on the APIs being present and managed by the platform thereafter.

The description of the Admin Gate will read something close to this:

```text
Starting with 4.19 the OpenShift Container Platform now provides automated
management of Gateway API resources. This service is provided to enable platform
level support of Gateway API and also allows third party implementations to
depend on the presence of the APIs as a service provided by the platform going
forward.

The implication for you is that all CRDs for the Gateway API project need to be
transferred to control by the platform in order to complete this upgrade. We
have detected Gateway API CRDs on this cluster, and you must verify the safety
of any implementations and users of Gateway API which are currently active on
the cluster, as we can not do this automatically.

Downtime and information loss can occur if you do not fully prepare existing
Gateway API implementations before proceeding with the upgrade. The CRDs will
be upgraded to version $CRD_VERSION, which you will need to ensure running
implementations can support. Experimental resources such as TCPRoute, UDPRoute
and TLSRoute will be removed entirely. Older versions (prior to version 1) of
APIs such as v1beta2.HTTPRoute will be removed. Experimental fields will be
removed (if present) from any v1 APIs. This is not an exhaustive list of things
to prepare for. See $DOCUMENTATION_URL for further details.
```

We will provide accompanying documentation that goes into more details about
all the situations the user will need to check for.

_TBD: verify with PMs that we do indeed want the admingate solution.
_TBD: document how this is going to work if you're coming from older versions.

[an effort]:https://github.com/kubernetes-sigs/gateway-api/discussions/3497

### Dead fields

_TBD_: Describe the issue and how we will address it in 4.19.

### Workflow Description

> Explain how the user will use the feature. Be detailed and explicit.  Describe
> all of the actors, their roles, and the APIs or interfaces involved. Define a
> starting state and then list the steps that the user would need to go through to
> trigger the feature described in the enhancement. Optionally add a
> [mermaid](https://github.com/mermaid-js/mermaid#readme) sequence diagram.
> 
> Use sub-sections to explain variations, such as for error handling,
> failure recovery, or alternative outcomes.

**cluster-admin** is a human user responsible for managing a cluster.

1. Start with a 4.18 cluster with conflicting CRDs.
2. Upgrade to 4.19.
3. Check clusteroperators, see a conflict.
4. Run some `oc` command.
5. Check the ingress clusteroperator again.  Now everything should be dandy.

_TBD: Fill in the details for handing ownership of CRD life-cycle management to
the Ingress Operator in the case of a conflict, using Server-Side Apply._

### API Extensions

None.

### Topology Considerations

#### Hypershift / Hosted Control Planes

Hypershift runs the Ingress Operator on the management cluster but configured
with a kubeconfig to manage resources on the guest cluster.  This means that the
Ingress Operator manages the Gateway API CRDs on the guest cluster, the same as
on standalone clusters.

#### Standalone Clusters

For standalone clusters, the Ingress Operator manages the Gateway API CRD
life-cycle so that Gateway API can be configured and used post-install.

#### Single-node Deployments or MicroShift

The CRDs themselves use minimal resources.  Creating a GatewayClass CR can cause
the Ingress Operator to install OpenShift Service Mesh, which in turn installs
Istio and Envoy (see the [gateway-api-with-cluster-ingress-operator](gateway-api-with-cluster-ingress-operator.md)
enhancement proposal), which use considerable resources.  For Single-Node
OpenShift, the cluster-admin might be advised to pay particular attention to
OSSM's resource requirements and the cluster's resource constraints before
attempting to use Gateway API.

MicroShift does not run the Ingress Operator and has its own design for
supporting Gateway API (see the MicroShift [gateway-api-support](../microshift/gateway-api-support.md)
enhancement).

### Implementation Details/Notes/Constraints

> What are some important details that didn't come across above in the
> **Proposal**? Go in to as much detail as necessary here. This might be
> a good place to talk about core concepts and how they relate. While it is useful
> to go into the details of the code changes required, it is not necessary to show
> how the code will be rewritten in the enhancement.

### Risks and Mitigations

#### Dead fields

If an installed Gateway API CRD includes some field that is not implemented by
the installed Gateway API implementation, then updates to this field may be
silently ignored.  In the context of a security feature, ignoring this field
could result in a configuration that inadvertently exposes workload.

Potential mitigation strategies:

- Handle with an admission webhook (see [Admission webhook](#admission-webhook) under [Alternatives](#alternatives).
- Work upstream to address this issue.  _TODO: Link to a relevant upstream discussion._
- Defer the issue until we allow some newer CRD version than we initially allow.

_TBD: Fill in more details._

### Drawbacks

> The idea is to find the best form of an argument why this enhancement should
> _not_ be implemented.
> 
> What trade-offs (technical/efficiency cost, user experience, flexibility,
> supportability, etc) must be made in order to implement this? What are the reasons
> we might not want to undertake this proposal, and how do we overcome them?
> 
> Does this proposal implement a behavior that's new/unique/novel? Is it poorly
> aligned with existing user expectations?  Will it be a significant maintenance
> burden?  Is it likely to be superceded by something else in the near future?

_TBD_

## Open Questions [optional]

> This is where to call out areas of the design that require closure before deciding
> to implement the design.  For instance,
>  > 1. This requires exposing previously private resources which contain sensitive
>   information.  Can we do this?

_TBD_

## Test Plan

> **Note:** *Section not required until targeted at a release.*
> 
> Consider the following in developing a test plan for this enhancement:
> - Will there be e2e and integration tests, in addition to unit tests?
> - How will it be tested in isolation vs with other components?
> - What additional testing is necessary to support managed OpenShift service-based offerings?
> 
> No need to outline all of the test cases, just the general strategy. Anything
> that would count as tricky in the implementation and anything particularly
> challenging to test should be called out.
> 
> All code is expected to have adequate tests (eventually with coverage
> expectations).

The Ingress Operator will have E2E tests to simulate the user stories outlined
in this EP:

- Delete the CRDs; verify that the operator re-installs them.
- Install incompatible CRDs with `metadata.managedFields` set to indicate that the operator *did not* install them; verify that the operator reports the appropriate `Degraded` status.
- Install older CRDs with `metadata.managedFields` set to indicate that an older version of the operator *did* install them; verify that the operator updates them.

_TBD_

## Graduation Criteria

Note that Gateway API CRD life-cycle management will be part of the
[gateway-api-with-cluster-ingress-operator](gateway-api-with-cluster-ingress-operator.md) enhancement, which defines its
own graduation criteria.

### Dev Preview -> Tech Preview

N/A.

### Tech Preview -> GA

- End-to-end tests in the Ingress Operator.
- Upgrade testing with conflicting CRDs.
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

### Removing a deprecated feature

N/A.

## Upgrade / Downgrade Strategy

> If applicable, how will the component be upgraded and downgraded? Make sure this
> is in the test plan.
> 
> Consider the following in developing an upgrade/downgrade strategy for this
> enhancement:
> - What changes (in invocations, configurations, API use, etc.) is an existing
>   cluster required to make on upgrade in order to keep previous behavior?
> - What changes (in invocations, configurations, API use, etc.) is an existing
>   cluster required to make on upgrade in order to make use of the enhancement?
> 
> Upgrade expectations:
> - Each component should remain available for user requests and
>   workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
>   disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
>   this should be identified and discussed here.
> - Micro version upgrades - users should be able to skip forward versions within a
>   minor release stream without being required to pass through intermediate
>   versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
>   as an intermediate step.
> - Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
>   steps. So, for example, it is acceptable to require a user running 4.3 to
>   upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
> - While an upgrade is in progress, new component versions should
>   continue to operate correctly in concert with older component
>   versions (aka "version skew"). For example, if a node is down, and
>   an operator is rolling out a daemonset, the old and new daemonset
>   pods must continue to work correctly even while the cluster remains
>   in this partially upgraded state for some time.
> 
> Downgrade expectations:
> - If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
>   misbehaving, it should be possible for the user to rollback to `N`. It is
>   acceptable to require some documented manual steps in order to fully restore
>   the downgraded cluster to its previous state. Examples of acceptable steps
>   include:
>   - Deleting any CVO-managed resources added by the new version. The
>     CVO does not currently delete resources that no longer exist in
>     the target version.

## Version Skew Strategy

> How will the component handle version skew with other components?
> What are the guarantees? Make sure this is in the test plan.
> 
> Consider the following in developing a version skew strategy for this
> enhancement:
> - During an upgrade, we will always have skew among components, how will this impact your work?
> - Does this enhancement involve coordinating behavior in the control plane and
>   in the kubelet? How does an n-2 kubelet without this feature available behave
>   when this feature is used?
> - Will any other components on the node change? For example, changes to CSI, CRI
>   or CNI may require updating that component before the kubelet.

_TBD: Do we describe version skew with layered products here?_

## Operational Aspects of API Extensions

> Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
> admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
> especially how they impact the OCP system architecture and operational aspects.
> 
> - For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
>   Indicators) an administrator or support can use to determine the health of the API extensions
> 
>   Examples (metrics, alerts, operator conditions)
>   - authentication-operator condition `APIServerDegraded=False`
>   - authentication-operator condition `APIServerAvailable=True`
>   - openshift-authentication/oauth-apiserver deployment and pods health
> 
> - What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
>   API availability)
> 
>   Examples:
>   - Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
>   - Fails creation of ConfigMap in the system when the webhook is not available.
>   - Adds a dependency on the SDN service network for all resources, risking API availability in case
>     of SDN issues.
>   - Expected use-cases require less than 1000 instances of the CRD, not impacting
>     general API throughput.
> 
> - How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
>   automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
>   this enhancement)
> 
> - Describe the possible failure modes of the API extensions.
> - Describe how a failure or behaviour of the extension will impact the overall cluster health
>   (e.g. which kube-controller-manager functionality will stop working), especially regarding
>   stability, availability, performance and security.
> - Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
>   and add them as reviewers to this enhancement.

_TBD: Do we need to describe anything here?_

## Support Procedures

### Conflicting CRDs

If the Ingress Operator detects the presence of a conflicting version of the
Gateway API CRDs, it updates the ingress clusteroperator to report a `Degraded`
status condition with status `True` and a message explaining the situation:

_TBD: Insert example output from `oc get clusteroperators/ingress -o yaml`._

In this situation, the cluster-admin is expected to verify that workload would
not be broken by handing life-cycle management of the CRDs over to the Ingress
Operator:

_TBD: Insert `oc` command to make the CRD ownership transition._

Then the Ingress Operator takes ownership and updates the CRDs:

_TBD: Insert example `oc get clusteroperators` and `oc get crds` commands._

### Overriding the Ingress Operator

_TBD: Should we describe how to turn off the Ingress Operator so that the
cluster-admin can override the CRDs, or describe how Server-Side Apply enables
the cluster-admin to take over the CRDs?_

## Alternatives

### Use an admin-ack gate to block upgrades if incompatible CRDs exist

One option considered was to add logic in OpenShift 4.18's Ingress Operator to
detect conflicting Gateway API CRDs.  This logic would block upgrades from 4.18
to 4.19 if conflicting CRDs were detected.  Then 4.19's Ingress Operator could
unconditionally take ownership of the CRDs' life-cycle.

This has the advantage of providing a warning *before* upgrade.  In contrast,
the solution proposed in this enhancement allows the upgrade and then reports a
`Degraded` status condition *after* the upgrade.  However, in either case, the
cluster-admin is responsible for resolving the conflict.  Thus the admin-ack
gate adds complexity without significantly improving the user experience.

We conclude that the effort of adding an admin-ack gate isn't worth the effort.

### Use a fleet evaluation condition to detect clusters with incompatible CRDs

Another option considered was to add a [fleet evaluation condition](../dev-guide/cluster-fleet-evaluation.md) to tell
us how many clusters have conflicting CRDs already installed.  This could help
us decide whether implementing upgrade-blocking logic (such as the
aforementioned admin-ack gate) would be beneficial.  However, given time
constraints, and given that we need to handle conflicts in any case, we have
concluded that the fleet evaluation condition would not be of much benefit.

### Admission webhook

An admission webhook could be implemented for the Gateway API CRDs to prevent
writes to any field that is in the CRD but isn't implemented by our Gateway API
implementation.

Note that using a webhook for this purpose would run into consistency issues and
race conditions because the webhook would need to cross-validate multiple
resources.  Specifically, the webhook would need to check which gatewayclasses
specified our controller name; then the webhook would check *only* resources
(gateways, httproutes, etc.) associated with those gatewayclasses for fields
that our controller would not recognize.  Consistency issues could arise, for
example, if an object were created and subsequently updated to reference a
gatewayclass, or if a gatewayclass were created (or its controller name were
updated) after resources that referenced that gatewayclass by name had already
been created.

_TBD: Fill in details._

We conclude that this can be further evaluated and, if appropriate, implemented
post-GA if the need arises to allow newer CRD versions than the version that our
Gateway API implementation recognizes.

### Provide an API for explicitly overriding CRD life-cycle management

Inspired by CAPI's mechanism.  This could be useful in a procedure for upgrading
a cluster with cluster-admin-owned CRDs to a cluster with operator-managed CRDs.

_TBD: Fill in details._

### Validate and allow a range of CRD versions

As a way to offer more flexibility for third-party implementations, we
considered defining a range of allowed CRD versions.  On the one hand, this
approach has the minor advantage of avoiding dead fields unless the
cluster-admin *needs* a newer CRD that has dead fields.

On the other hand, allowing a range adds complexity, requires more testing, and
still must be constrained to avoid security-problematic dead fields.  This added
complexity has questionable value and could delay the feature.  Therefore we
conclude that it is best to pin the CRDs to a specific version for at least the
initial GA release of the Gateway API feature.

### Package CRDs as operator manifests that Cluster Version Operator owns

_TBD_

## Infrastructure Needed [optional]

No new infrastructure is required for this enhancement.
