# KEP-2189: default container behavior

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

<!-- /toc -->

## Summary

Many Kubectl commands make requests to specific containers, `kubectl logs` and `kubectl exec` are most commonly
used.

All commands that need to specify container name:

- kubectl attach - Attach to a running container
- kubectl cp - Copy files and directories to and from containers.
- kubectl exec - Execute a command in a container

These three commands above are similar. If omitted, the first container in the pod will be chosen.


- kubectl logs - Print the logs for a container in a pod

`kubectl logs` is a little different. Not default value for it. 
However it support to logs all containers with `--all-containers`.

https://github.com/kubernetes/kubernetes/pull/87809 already add an annotation named
`kubectl.kubernetes.io/default-logs-container` for log to use as default.

We don't have a general default container name attribute for pod, and this would change pod spec and not acceptable.
Why not use `kubectl.kubernetes.io/default-container` as the default container name annotation for all four commands
above?


**Note:** No server-side changes are required for this, all Request and Response template expansion is performed on
the client side.

## Motivation


As the default log annotation is added, the motivation of it is like below
> Kubectl provides a number of commands to simplify working with Kubernetes by making requests to
> Resources and SubResources.  These requests are mostly static, with fields filled in by user
> supplied flags.  Today the commands are compiled into the client, which as the following challenges:

Sidecars are populor now.
Alternatively, a way to generic describe the default container for anything that happens to need a "default", which could also be used by external tooling.

Issues https://github.com/kubernetes/kubernetes/issues/96986 opened by @howardjohn 
> However, it gets worse because aside from a warning, this couples the default exec container to the container ordering. 
> The container ordering also happens to have an impact on container startup ordering. We have started offering an option 
> to inject our sidecar as the first container (previously, it was the last one), which has resulted in users running 
> kubectl exec and getting the "wrong" container.


### Goals


- Provide a way to know which is the default container in pod for cli: kubectl


### Non-Goals

- If the cli is not kubectl, we don't determine which is the default container.


## Proposal

A single and generic annotation for all above commands like `kubectl.kubernetes.io/default-container` is a good choice to avoid needing many new annotations in the future.

As logs and exec are frurently used by users, we start from these two, but there would be more in the future.

### Risks and Mitigations

- It is not compatible with `kubectl.kubernetes.io/default-logs-container` for kubectl log. Hence, this annotation will be deprecated in 1.21 and will be removed in 1.23.

## Design Details

**Publishing Data:**

Beta:  default container annotation

- in discussion, https://github.com/kubernetes/kubernetes/issues/95293
- (WIP, will drop this design and use the generic annotation later)  Users might specify the `kubectl.kubernetes.io/default-container` annotation in a Pod to preselect container for kubectl exec and all kubectl commands.
WIP in https://github.com/kubernetes/kubernetes/pull/97099


**Data Command Structure:**


**Example Command:**
Users might specify the `kubectl.kubernetes.io/default-container` annotation in a Pod to preselect container for kubectl exec and all kubectl commands.

A example Pod yaml is like below:

```
apiVersion: v1
kind: Pod
metadata:
 annotations:
    kubectl.kubernetes.io/default-logs-container: sidecar-container
<snip>
```

### Test Plan


### Graduation Criteria

- At least 2 release cycles pass to gather feedback and bug reports during
- Documentations, add it to [well known annotations docs](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/)


### Upgrade / Downgrade Strategy


### Version Skew Strategy


## Production Readiness Review Questionnaire


### Feature Enablement and Rollback

### Monitoring Requirements

### Dependencies

### Scalability


### Troubleshooting


## Implementation History

WIP in https://github.com/kubernetes/kubernetes/pull/97099 in 1.21.

## Drawbacks

It is not generic for other clients like go-client for kubernetes.

## Alternatives

Use `-c` or `--container` option in kubectl commands.
