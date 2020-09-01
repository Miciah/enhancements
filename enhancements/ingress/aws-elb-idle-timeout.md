---
title: aws-elb-idle-timeout
authors:
  - "@Miciah"
reviewers:
  - "@danehans"
  - "@frobware"
  - "@knobunc"
  - "@sgreene570"
approvers:
  - "@knobunc"
creation-date: 2020-08-31
last-updated: 2020-08-31
status: implementable
see-also:
replaces:
superseded-by:
---

# Ingress AWS ELB Idle Connection Timeout

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement extends the IngressController API to allow the user to
configure the timeout period for idle connections to an IngressController that
is published using an AWS Classic Elastic Load Balancer.

## Motivation

The default connection timeout period for routes is 30 seconds.  It is possible
to set a custom timeout period for a specific route.  However, the external
load-balancer may impose a timeout period of its own.  In particular, AWS
Classic ELBs by default have timeout period of 60 seconds.  In order to allow
applications to use higher timeout periods, the external load-balancer's timeout
must be increased as well.

### Goals

1. Enable the cluster administrator to specify the connection timeout period for AWS Classic ELBs.

### Non-Goal

1. Introduce a platform-agnostic API (which would be impossible to implement on most platforms).

## Proposal

To enable the cluster administrator to configure the connection timeout period
for AWS Classic ELBs, the IngressController API is extended by adding an
optional `ConnectionIdleTimeoutSeconds` field with type `uint64` to
`AWSClassicLoadBalancerParameters`:

```go
// AWSClassicLoadBalancerParameters holds configuration parameters for an
// AWS Classic load balancer.
type AWSClassicLoadBalancerParameters struct {
	// connectionIdleTimeoutSeconds is the time interval, in seconds, that a
	// connection may be idle before the load balancer closes the
	// connection.  If not set, the timeout defaults to 60 seconds.
	//
	// +kubebuilder:validation:Optional
	// +optional
	ConnectionIdleTimeoutSeconds uint64 `json:"connectionIdleTimeoutSeconds"`
}
```

The following example configures a 2-minute timeout period:

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  endpointPublishingStrategy:
    type: LoadBalancerService
    loadBalancer:
      providerParameters:
        type: AWS
        aws:
          type: Classic
          classicLoadBalancer:
            connectionIdleTimeoutSeconds: 120
```

#### Validation

Omitting `spec.endpointPublishingStrategy`,
`spec.endpointPublishingStrategy.loadBalancer`,
`spec.endpointPublishingStrategy.loadBalancer.providerParameters`,
`spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws`,
`spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.classicLoadBalancer`,
or
`spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.classicLoadBalancer.connectionIdleTimeoutSeconds`
is valid and specifies the default behavior.  The API validates that any
provided value is an unsigned integer.

### User Stories

#### As a cluster administrator, I need my IngressController's ELB to have a 2-minute timeout.

To satisfy this use-case, the cluster administrator can set the
IngressController's
`spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.classicLoadBalancer.connectionIdleTimeoutSeconds`
field to `120`.

### Implementation Details

Implementing this enhancement requires changes in the following repositories:

* openshift/api
* openshift/cluster-ingress-operator

When the endpoint publishing strategy type is "LoadBalancerService", the ingress
operator creates the appropriate service.  If the
`spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.classicLoadBalancer.connectionIdleTimeoutSeconds`
field is specified, the operator adds the
`service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout`
annotation to the service with the specified value.

Before this enhancement, the ingress operator did not check the service's
annotations to determine if an update was needed.  Implementing this enhancement
includes making the operator update the service if its annotations do not match
the expected annotations.

### Risks and Mitigations

Adding platform-specific parameters poses a risk of API sprawl.  Mitigating
factors are that the API already has a field for Classic ELB-specific
parameters.  Moreover, the connection timeout period cannot be configured on
other platforms; that is, it is inherently platform-specific, and so trying to
make the API appear more generic would be misleading.

## Design Details

### Test Plan

The controller that manages the IngressController deployment and related
resources has unit test coverage; for this enhancement, the unit tests are
expanded to cover the additional functionality.

The operator has end-to-end tests; for this enhancement, the following test can
be added:

1. Create an IngressController that specifies a short timeout (for example, 10 seconds).
2. Create a pod with a simple HTTP application that sends static responses.
3. Create a route for this application.
4. Open a connection to this route and wait for the timeout.
5. Verify that the connection times out after 10 seconds.
6. Configure the IngressController with a longer timeout (for example, 120 seconds).
7. Configure the route with a 90-second timeout.
8. Open a connection to this route and wait for 70 seconds.
9. Send a request on the open connection.
10. Verify that a response is received.

### Graduation Criteria

N/A.

### Upgrade / Downgrade Strategy

On upgrade, the default ELB timeout period remains in effect.

Before this enhancement was added, the ingress operator did not update the
service's annotations when they changed.  Thus downgrading the operator may
leave any configured timeout period in effect until the IngressController is
deleted.

### Version Skew Strategy

N/A.

## Implementation History

## Alternatives

