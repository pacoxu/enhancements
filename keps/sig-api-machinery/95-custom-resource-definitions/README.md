# Graduate CustomResourceDefinitions to GA

## Table of Contents

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Defaulting and pruning for custom resources is implemented](#defaulting-and-pruning-for-custom-resources-is-implemented)
  - [CRD v1 schemas are restricted to a subset of the OpenAPI specification](#crd-v1-schemas-are-restricted-to-a-subset-of-the-openapi-specification)
  - [Generator exists for CRD Validation Schema v3 (Kubebuilder)](#generator-exists-for-crd-validation-schema-v3-kubebuilder)
  - [CustomResourceWebhookConversion API is GA ready](#customresourcewebhookconversion-api-is-ga-ready)
  - [CustomResourceSubresources API is GA ready](#customresourcesubresources-api-is-ga-ready)
  - [v1 API](#v1-api)
- [Test Plan](#test-plan)
  - [Integration tests for GA](#integration-tests-for-ga)
  - [e2e tests for GA](#e2e-tests-for-ga)
  - [Conformance tests for GA](#conformance-tests-for-ga)
  - [Scale Targets for GA](#scale-targets-for-ga)
- [Graduation Criteria](#graduation-criteria)
- [Post-GA tasks](#post-ga-tasks)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Summary

CustomResourceDefinitions (CRDs) are the way to extend the Kubernetes API to
include custom resource types that behave like the native resource types. CRDs
have been in Beta since Kubernetes 1.7. This document outlines the required
steps to graduate CRDs to general availability (GA).

## Motivation

CRDs are in widespread use as a Kubernetes extensibility mechanism and have been
available in beta for the last 5 Kubernetes releases. Feature requests and bug
reports suggest CRDs are nearing GA quality, and this KEP aims to itemize the
remaining improvements to move this toward GA.

### Goals

Establish consensus for the remaining essential improvements needed to move CRDs to GA.

Guiding principles:
* if a missing feature forces or encourages users to implement non-Kubernetes-like APIs, and therefore damages the ecosystem long term, it belongs on this list.
* If a feature cannot be added as an after-thought of a GA API, it or some preparation belongs on this list.


### Non-Goals

Full feature parity with native kubernetes resource types is not a GA graduation goal. See above guiding principles.

## Proposal

Objectives to graduate CRDs to GA, each of which is described in more detail further below:

* Defaulting and pruning for custom resources is implemented
* CRD v1 schemas are restricted to a subset of the OpenAPI specification (and there is a v1beta1 compatibility plan)
* Generator exists for CRD Validation Schema v3 (Kubebuilder)
* CustomResourceWebhookConversion API is GA ready
* CustomResourceSubresources API is GA ready

Bug fixes required to graduate CRDs to GA:

* See “Required for GA” issues tracked via the [CRD Project Board](https://github.com/orgs/kubernetes/projects/28).

For additional details on already completed features, see the [CRD Project Board](https://github.com/orgs/kubernetes/projects/28).

See [Post-GA tasks](#post-ga-tasks) for decided out-of-scope features.

### Defaulting and pruning for custom resources is implemented

Both defaulting and pruning and also read-only validation are blocked by the
OpenAPI subset definition (next point). 

See the [Pruning for CustomResources KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20180731-crd-pruning.md)
and the [Defaulting for Custom Resources KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190426-crd-defaulting.md).

### CRD v1 schemas are restricted to a subset of the OpenAPI specification

See [OpenAPI Subset KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190425-structural-openapi.md)

### Generator exists for CRD Validation Schema v3 (Kubebuilder)

See [kubebuilder’s
crd-gen](https://github.com/kubernetes-sigs/controller-tools/tree/master/pkg/crd)
and a more general
[crd-schema-gen](https://github.com/kubernetes-sigs/controller-tools/pull/183),
to be integrated into kubebuidler’s controller-tools.

### CustomResourceWebhookConversion API is GA ready

Currently CRD webhook conversion is alpha. We plan to take this to v1beta1 according to the
[CustomResourceDefinition Conversion Webhook's Graduation Criteria](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190425-crd-conversion-webhook.md#graduation-criteria).
We plan to then graduate this to GA as part of the CRD to GA graduation.

### CustomResourceSubresources API is GA ready

Currently custom resource subresources are v1beta1 and provide support for the
/status and /scale subresources. We plan to graduate this to GA as part of the
CRD to GA graduation.

### v1 API

The CRD `v1` API will be the same as the `v1beta1` but with all changes to the API from the GA tasks:

* Rename misnamed json field [JSONPath](https://github.com/kubernetes/kubernetes/blob/06bc7e3e0026ea25065f59f4bd305c0b7dbbc145/staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/types.go#L226-L227) to `jsonPath`
* [Replace top-level fields with per version fields](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/customresource-conversion-webhook.md#top-level-fields-to-per-version-fields)
* Restrict OpenAPI per [Vanilla OpenAPI Subset Design](https://docs.google.com/document/d/1pcGlbmw-2Y0JJs9hsYnSBXamgG9TfWtHY6eh80zSTd8)

## Test Plan

### Integration tests for GA

* Ensure CRD themselves are v1<->v1beta1 round trippable
* CRD versioning/conversion integration test coverage:
  * Ensure what is persisted in etcd matches the storage version
  * Set up a CRD, persist some data, changed the version, and ensure the previously persisted data is readable
  * Ensure discovery docs track a CRD through creation, version addition, version removal, and deletion
  * Ensure `spec.served` is respected
  * Ensure illegal inputs result in sensible errors, not panics, and that the system recovers appropriately
    * CRD mutation operations: invalid schemas, misconfigured webhooks, scope changes, ...
    * webhook conversion responses: conversion responses that do not align with request (uuid, list size and elements, ..)
* See also [Pruning for Custom Resources KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20180731-crd-pruning.md)
* See also [Vanilla OpenAPI Subset: structural schemas KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190425-structural-openapi.md)
* See also [Defaulting for Custom Resources KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190426-crd-defaulting.md)

### e2e tests for GA

* Custom Resources should be readable and writable at all available versions (test for round trip-ability)
* Introduce v1 API testing in addition to v1beta1 tests

### Conformance tests for GA

* Prepare a e2e test case candidates that we will request be promoted to Conformance (which is a process that happens after GA)
  * test case covering the basics of the CRD v1 API
  * test cases exemplifying CRD versioning for both `strategy: None` and `strategy: Webhook`.

### Scale Targets for GA

The scale targets for GA of custom resources are defined by the same [API call latency
SLIs/SLOs as the Kuberetes native types](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/api_call_latency.md#api-call-latency-slisslos-details).

The targets are defined by the below suggested maximum limits, which are organized the same way as the [Kubernetes native type thresholds](https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/thresholds.md#kubernetes-thresholds), with only one change:

- Since custom resources can be arbitrarily large, we have broken down the limit by custom resource object size.

**Custom Resource Definitions:**

| Suggested Maximum Limit: scope=cluster |
| --- |
| 500 |

_Note: The Custom Resource Definition suggested maximum limit was selected not
due to the above SLI/SLOs, but instead due to the latency OpenAPI publishing,
which is a background process that occurs asychroniously each time a Custom
Resource Definition schema is updated. For 500 Custom Resource Definitions it takes
slightly over 35 seconds for a definition change to be visible via the OpenAPI
spec endpoint._

**Custom Resources, Cluster Wide:**

Cluster wide limits for custom resources are storage bound and custom resources
share the storage space with all other objects. While determining the
appropriate storage limit for a cluster is out-of-scope for this document, once
a etcd storage limit selected, suggested maximum limits for custom resources
are:

| etcd storage limit | Suggested Maximum Limit: scope=cluster |
| --- | --- |
| 4GB | 40000 |
| 8GB | 80000 |

These limits aim to keep custom resource storage usage to less than half of the
total cluster storage capacity for custom resources of 50kb or less in size.

**Custom Resources per Definition:**

For each custom resource definition, the limit on the number of custom resources
can be found by taking the (median) object size of the custom resource and finding
the the matching row in this table:

| Object size | Suggested Maximum Limit: scope=namespace (5s p99 SLO) | Suggested Maximum Limit: scope=cluster (30s p99 SLO) |
| --- | --- | --- |
| <=10kb | 1500 | 10000 |
| (10kb - 25kb] | 600 | 4000 |
| (25kb - 50kb] | 300 | 2000 |

The cluster scope indicates the total number of custom resources for that
definition allowed in the entire cluster.

The namespace scope indicates the total number of custom resources for that
definition allowed in any particular namespace. The cumulative count of the
custom resource across all namespaces must not exceed the cluster limit.

Since, in practice, custom resources scale farther without conversion webhooks
within the SLI/SLOs (roughly 2x according to our scale tests), custom resource
definition authors should be careful to adhere to these limits so that in the
future a webhook converter may safely be added as part of a custom resource
version upgrade.

_Note: For custom resources of custom resource definitions using `scope: Namespaced`: the scope=namespace
suggested maximum limit indicates how many custom resource objects may be in each namespace,
and the scope=cluster suggested maximum limit indicates how many custom resource objects may
be in the cluster total. For custom resources of custom resource definitions using `scope: Cluster`: only
the scope=cluster suggested maximum limit applies._

**Conversion Webhooks:**

Conversion Webhook SLOs are defined from the perspective of the conversion
webhook. It does not include any api-server serialization/deserialization for
making the request to the webhook, but it does include network latency.

Given that the performance and scalability of conversion webhooks are the
responsibility of their author, Custom resource scale targets are applied only for
conversion webhooks that are within the following latencies for the above suggested
maximum limits.

| scope | object count limit | Expected conversion Webhook SLO: p99 latency |
| --- | --- | --- |
| resource | 1 | 50ms |
| namespace | 1500 (<=10kb), 600 (10-25kb) or 300 (25-50kb) | 1 seconds |
| cluster | 10000 (<=10kb), 4000 (10-25kb) or 2000 (25-50kb) | 6 seconds |

The scope=resource's higher "per-object" latency (50ms vs ~1.5ms for namespace
and cluster scope) is to accommodate for a request serving cost constant.

The above object size and suggested maximum limits in the Custom Resources per
Definition table applies to these conversion webhook SLOs. For example, for a
list request for 1500 custom resource objects that are 10k in size, the resource
scope SLO of 1 second for the conversion webhook applies.

**Scale Target Data**

GA custom resource scale targets were selected based on an [analysis of our current scale limits](https://docs.google.com/document/d/1tEstPQvzGvaRnN-WwGUWx1H9xHPRCy_fFcGlgTkB3f8).

We ran a month long survey of Custom Resource Definition scale needs across Kubernetes mailing lists, slack channels and social media.
Of the custom resource definitions surveyed, 96% are currently within these suggested maximum limits, 91% are within these limits for their anticipated future growth, and survey data provides useful guidance for our post-GA scalability work. See [survey of real-world custom resource usage](https://docs.google.com/document/d/1MTd_gDlpgBaT5sAKM4j6tQVeCFIT9J44RHzt2yWOK_g) for details.

As part of GA the suggested maximum limits and SLO documentation will be updated to make this clear and to
encourage CRD authors to provide concrete suggested maximum limits and SLIs/SLOs for their custom
resource kinds to their users that account for the per resource conversion cost
of their conversion webhook and/or size of their custom resources.

## Graduation Criteria

To mark these as complete, all of the above features need to be implemented. An
umbrella issue is tracking all of these changes. Also there need to be
sufficient tests for any of these new features and all existing features and
documentation should be completed for all features.

See [umbrella issue](https://github.com/kubernetes/kubernetes/issues/58682) for status.

## Post-GA tasks

See the [umbrella issue](https://github.com/kubernetes/kubernetes/issues/58682) for details on Post-GA tasks. The tasks at the time this KEP was written are:

* Human readable status from conditions for a CRD using additionalPrinterColumns (https://github.com/kubernetes/kubernetes/issues/67268)
* Consider changing the schema language in CRDs (https://github.com/kubernetes/kubernetes/issues/67840)
* Should support graceful deletion for storage (https://github.com/kubernetes/kubernetes/issues/68508)
* Enable arbitrary CRD field selectors by supporting a whitelist of fields in CRD spec (https://github.com/kubernetes/kubernetes/issues/53459)
* Support graceful deletion for custom resources (https://github.com/kubernetes/kubernetes/issues/56567)
* CRD validation webhooks (https://github.com/kubernetes/community/pull/1418)
* Allow OpenAPI references in the CRD valiation schema (https://github.com/kubernetes/kubernetes/issues/54579, https://github.com/kubernetes/kubernetes/issues/62872)
* Generate json-schema for use in the CRDs from the go types (https://github.com/kubernetes/kubernetes/issues/59154, https://github.com/kubernetes/sample-controller/issues/2, https://github.com/kubernetes/code-generator/issues/28)
* Support for namespace-local CRD (https://github.com/kubernetes/kubernetes/issues/65551)
* Support proto encoding for custom resources (https://github.com/kubernetes/kubernetes/issues/63677)
* labelSelectorPath should be allowed not be under .status (https://github.com/kubernetes/kubernetes/issues/66688)
* Support arbitrary non-standard subresources for CRDs (https://github.com/kubernetes/kubernetes/issues/72637)
* OpenAPI v3 is published for all resources, including custom resources

## Implementation History
