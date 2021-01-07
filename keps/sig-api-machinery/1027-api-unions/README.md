# Union types

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Go tags](#go-tags)
  - [OpenAPI](#openapi)
  - [Discriminator](#discriminator)
  - [Normalizing on updates](#normalizing-on-updates)
    - [&quot;At most one&quot; versus &quot;exactly one&quot;](#at-most-one-versus-exactly-one)
    - [Clearing all the fields](#clearing-all-the-fields)
  - [Backward compatibilities properties](#backward-compatibilities-properties)
  - [Validation](#validation)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
- [Future Work](#future-work)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Release Signoff Checklist

- [x] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [x] KEP approvers have set the KEP status to `implementable`
- [x] Design details are appropriately documented
- [x] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [x] Graduation criteria is in place
- [x] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

## Summary

Modern data model definitions like OpenAPI v3 and protobuf (versions 2 and 3)
have a keyword to implement “oneof” or "union". They allow APIs to have a better
semantics, typically as a way to say “only one of the given fields can be
set”. We currently have multiple occurrences of this semantics in kubernetes core
types, at least:
- VolumeSource is a structure that holds the definition of all the possible
  volume types, only one of them must be set, it doesn't have a discriminator.
- DeploymentStrategy is a structure that has a discrminator
  "DeploymentStrategyType" which decides if "RollingUpate" should be set

The problem with the lack of solution is that:
- The API is implicit, and people don't know how to use it
- Clients can't know how to deal with that, especially if they can't parse the
  OpenAPI
- Server can't understand the user intent and normalize the object properly

## Motivation

Currently, changing a value in an oneof type is difficult because the semantics
is implicit, which means that nothing can be built to automatically fix unions,
leading to many bugs and issues:
- https://github.com/kubernetes/kubernetes/issues/35345
- https://github.com/kubernetes/kubernetes/issues/24238
- https://github.com/kubernetes/kubernetes/issues/34292
- https://github.com/kubernetes/kubernetes/issues/6979
- https://github.com/kubernetes/kubernetes/issues/33766
- https://github.com/kubernetes/kubernetes/issues/24198
- https://github.com/kubernetes/kubernetes/issues/60340

And then, for other people:
- https://github.com/rancher/rancher/issues/13584
- https://github.com/helm/charts/pull/12319
- https://github.com/EnMasseProject/enmasse/pull/1974
- https://github.com/helm/charts/pull/11546
- https://github.com/kubernetes/kubernetes/pull/35343

This is replacing a lot of previous work and long-standing effort:
- Initially: https://github.com/kubernetes/community/issues/229, then
- https://github.com/kubernetes/community/pull/278
- https://github.com/kubernetes/community/pull/620
- https://github.com/kubernetes/kubernetes/pull/44597
- https://github.com/kubernetes/kubernetes/pull/50296
- https://github.com/kubernetes/kubernetes/pull/70436

Server-side [apply](http://features.k8s.io/555) is what enables this proposal to
become possible.

### Goals

The goal is to enable a union or "oneof" semantics in Kubernetes types, both for
in-tree types and for CRDs.

### Non-Goals

We're not planning to use this KEP to release the feature, but mostly as a way
to document what we're doing.

## Proposal

In order to support unions in a backward compatible way in kubernetes, we're
proposing the following changes.

Note that this proposes unions to be "at most one of". Whether exactly one is
supported or not should be implemented by the validation logic.

### Go tags

We're proposing a new type of tags for go types (in-tree types, and also
kubebuilder types):

- `// +union` before a structure means that the structure is a union. All the
  fields must be optional (beside the discriminator) and will be included as
  members of the union. That structure CAN be embedded in another structure.
- `// +unionDeprecated` before a field means that it is part of the
  union. Multiple fields can have this prefix. These fields MUST BE optional,
  omitempty and pointers. The field is named deprecated because we don't want
  people to embed their unions directly in structures, and only exist because of
  some existing core types (e.g. `Value` and `ValueFrom` in
  [EnvVar](https://github.com/kubernetes/kubernetes/blob/3ebb8ddd8a21b/staging/src/k8s.io/api/core/v1/types.go#L1817-L1836)).
- `// +unionDiscriminator` before a field means that this field is the
  discriminator for the union. Only one field per structure can have this
  prefix. This field HAS TO be a string, and CAN be optional.

Multiple unions can exist per structure, but unions can't span across multiple
go structures (all the fields that are part of a union has to be together in the
same structure), examples of what is allowed:

```
// This will have one embedded union.
type TopLevelUnion struct {
	Name string `json:"name"`

	Union `json:",inline"`
}

// This will generate one union, with two fields and a discriminator.
// +union
type Union struct {
	// +unionDiscriminator
	// +optional
	UnionType string `json:"unionType"`

	// +optional
	FieldA int `json:"fieldA"`
    // +optional
	FieldB int `json:"fieldB"`
}

// This also generates one union, with two fields and on discriminator.
type Union2 struct {
	// +unionDiscriminator
	Type string `json:"type"`
	// +unionDeprecated
    // +optional
	Alpha int `json:"alpha"`
	// +unionDeprecated
    // +optional
	Beta int `json:"beta"`
}

// This has 3 embedded unions:
// One for the fields that are directly embedded, one for Union, and one for Union2.
type InlinedUnion struct {
	Name string `json:"name"`

	// +unionDeprecated
	// +optional
	Field1 *int `json:"field1,omitempty"`
	// +unionDeprecated
	// +optional
	Field2 *int `json:"field2,omitempty"`

	Union  `json:",inline"`
	Union2 `json:",inline"`
}
```

### OpenAPI

OpenAPI v3 already allows a "oneOf" form, which is accepted by CRD validation
(and will continue to be accepted in the future). That oneOf form will be used
for validation, but is "on-top" of this proposal.

A new extension is created in the openapi to describe the behavior:
`x-kubernetes-unions`.

This is a list of unions that are part of this structure/object. Here is what
each list item is made of:
- `discriminator: <discriminator>` is set to the name of the discriminator
  field, if present,
- `fields-to-discriminateBy: {"<fieldName>": "<discriminateName>"}` is a map of
  fields that belong to the union to their discriminated names. The
  discriminatedValue will typically be set to the name of the Go variable.

Conversion between OpenAPI v2 and OpenAPI v3 will preserve these fields.

### Discriminator

For backward compatibility reasons, discriminators should be added to existing
union structures as an optional string. This has a nice property that it's going
to allow conflict detection when the selected union field is changed.

We also do strongly recommend new APIs to be written with a discriminator, and
tools (kube-builder) should probably enforce the presence of a discriminator in
CRDs.

The value of the discriminator is going to be set automatically by the apiserver
when a new field is changed in the union. It will be set to the value of the
`fields-to-discriminateBy` for that specific field.

When the value of the discriminator is explicitly changed by the client, it
will be interpreted as an intention to clear all the other fields. See section
below.

### Normalizing on updates

A "normalization process" will run automatically for all creations and
modifications (either with update or patch). It will happen automatically in order
to clear fields and update discriminators. This process will run for both
core-types and CRDs. It will take place before validation. The sent object
doesn't have to be valid, but fields will be cleared in order to make it valid.
This process will also happen before fields tracking (server-side apply), so
changes in discriminator, even if implicit, will be owned by the client making
the update (and may result in conflicts).

This process works as follows:
- If there is a discriminator, and its value has changed, clear all fields but
  the one specified by the discriminator,
- If there is no discriminator, or if its value hasn't changed,
  - if there is exactly one field, set the discriminator when there is one
    to that value. Otherwise,
  - compare the fields set before and after. If there is exactly one field
    added, set the discriminator (if present) to that value, and remove all
    other fields. if more than one field has been added, leave the process so
    that validation will fail.

#### "At most one" versus "exactly one"

The goal of this proposal is not to change the validation, but to help clients
to clear other fields in the union. Validation should be implemented for in-tree
types as it is today, or through "oneOf" properties in CRDs.

In other word, this is proposing to implement "at most one", and the exactly one
should be provided through another layer of validation (separated).

#### Clearing all the fields

Since the system is trying to do the right thing, it can be hard to "clear
everything". In that case, each API could decide to have their own "Nothing"
value in the discriminator, which will automatically trigger a clearing of all
fields beside "Nothing".

### Backward compatibilities properties

This normalization process has a few nice properties, especially for dumb
clients, when it comes to backward compatibility:
- A dumb client that doesn't know which fields belong to the union can just set
  a new field and get all the others cleared automatically
- A dumb client that doesn't know about the discriminator is going to change a
  field, leave the discriminator as it is, and should still expect the fields to
  be cleared accordingly
- A dumb client that knows about the discriminator can change the discriminator
  without knowing which fields to clear, they will get cleared automatically


### Validation

Objects have to be validated AFTER the normalization process, which is going to
leave multiple fields of the union if it can't normalize. As discussed in
drawbacks below, it can also be useful to validate apply requests before
applying them.

### Test Plan

There are mostly 3 aspects to this plan:
- [x] Core functionality, isolated from all other components: https://github.com/kubernetes-sigs/structured-merge-diff/blob/master/typed/union_test.go
- [x] Functionality as part of server-side apply: How human and robot interactions work: https://github.com/kubernetes-sigs/structured-merge-diff/blob/master/typed/union_test.go
- [x] Integration in kubernetes: https://github.com/kubernetes/kubernetes/pull/77370/files#diff-4ac5831d494b1b52c7c7be81e552a458

### Graduation Criteria

Since this project is a sub-project of Server-side apply, it will be introduced
directly as Beta, and will graduate to GA in a later release, according to the
criteria below.

#### Beta -> GA Graduation

- CRD support has been proven successful
- Core-types all implement the semantics properly
- Stable and bug-free for two releases

## Future Work

Since the proposal only normalizes the object after the patch has been applied,
it is hard to fail if the patch is invalid. There are scenarios where the patch
is invalid but it results in an unpredictable object. For example, if a patch
sends both a discriminator and a field that is not the discriminated field, it
will either clear the value sent if the discriminator changes, or it will change
the value of the sent discriminator.

Validating patches is not a problem that we want to tackle now, but we can
validate "Apply" objects to make sure that they do not define such broken
semantics.

## Implementation History

Here are the major milestones for this KEP:
- Initial discussion happened a year before the creation of this kep:
  https://docs.google.com/document/d/1lrV-P25ZTWukixE9ZWyvchfFR0NE2eCHlObiCUgNQGQ/edit#heading=h.w5eqnf1f76x5
- Points made in the initial document have been improved and put into this kep,
  which has approved by sig-api-machinery tech-leads
- KEP has been implemented:
  - logic mostly lives in sigs.k8s.io/structured-merge-diff
  - conversion between schema and openapi definition are in k8s.io/kube-openapi
  - core types have been modified in k8s.io/kubernetes
- Feature is ready to be released in Beta in kubernetes 1.15
