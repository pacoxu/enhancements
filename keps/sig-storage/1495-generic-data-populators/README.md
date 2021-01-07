# Generic Data Populators

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [VM Images](#vm-images)
    - [Backup/Restore](#backuprestore)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Release Signoff Checklist

**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core Kubernetes i.e., [kubernetes/kubernetes], we require the following Release Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

**Note:** Any PRs to move a KEP to `implementable` or significant changes once it is marked `implementable` should be approved by each of the KEP approvers. If any of those approvers is no longer appropriate than changes to that list should be approved by the remaining approvers and/or the owning SIG (or SIG-arch for cross cutting KEPs).

**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://github.com/kubernetes/enhancements/issues
[kubernetes/kubernetes]: https://github.com/kubernetes/kubernetes
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

In Kubernetes 1.12, we added the `DataSource` field to the PVC spec. The field was implemented
as a `TypedLocalObjectReference` to give flexibility in the future about what objects could be
data sources for new volumes.

Since then, we have allowed only two things to be the source of a
new volume -- existing PVCs (indicating a user's intent to clone the volume) and snapshots
(indicating the user's intent to restore a snapshot). Implementation of these two data sources
relies on a CSI plugin to do the actual work, and no generic implementation exists for cloning
volumes or restoring snapshots.

Since the original design of the `DataSource` API field, we have been aware of the desire to
populate volumes with data from other sources, both CSI-specific and generic (compatible with
any CSI plugin). For new CSI-specific data sources, the path forward is clear, but for other
sources of data, which I call "Generic Data Populators" we don't have a mechanism. The main
problem is that current API validation logic uses a white list of object types, driven by
feature gates for each object type. This approach won't scale to generic populators, which
will by their nature be too numerous and varied.

This proposal recommends that we relax validation on the `DataSource` field to allow
arbitrary object types to be data sources, and leave the implementation to controllers. This
flexibility will allow us to experiment with approaches for data populators.

## Motivation

### Goals

- Enable users to create pre-populated volumes in a manner consistent with current practice
- Enable developers to innovate with new an interesting data sources
- Avoid changing existing workflows and breaking existing tools with new behavior

### Non-Goals

- This proposal DOES NOT recommend any specific approach for data populators. The specific
  design of how a data populator should work will be handled in a separate KEP.
- We do not propose any new object types. Specifically, no special "populator"
  object type. Populators should function based on their own object types.

## Proposal

The validation for `DataSource` field should be removed entirely, in the fullness of time.

Short term, we should add a new alpha feature gate, which allows arbitrary objects to be
specified for the `DataSource` field, with the intention to eventually make this behavior
the standard. 

Populators will work by responding to PVC objects with a data source they understand,
and producing a PV with the expected data, such that ordinary Kubernetes workflows are
not disrupted. In particular the PVC should be attachable to the end user's pod the
moment it is bound, similar to a PVC created from currently supported data sources.

### User Stories

There are a long list of possible use cases around generic data populators, and I won't
try to detail them all here. I will detail a few that illustrate the challenges faced by
users and developers, but it's important to see these as a few examples among many.

#### VM Images

One use case was relevant to [KubeVirt](https://kubevirt.io/), where the Kubernetes
`Pod` object is actually a
VM running in a hypervisor, and the `PVC` is actually a virtual disk attached to a VM.
It's common for virtualization systems to allows VMs to boot from disks, and for disks
to be pre-populated with various OS images. OS images tend to be stored in external
repositories dedicated to that purpose, often with various mechanisms for retrieving
them efficiently that are external to Kubernetes.

One way to achieve this is to represent disk images as custom resources that point to
the image repository, and to allow creation of PVCs from these custom resources such
that the volumes come pre-populated with the correct data. Efficient population of the
data could be left up to a purpose-built controller that knows how to get the bits
where they need to be with minimal I/O.

#### Backup/Restore

Without getting into the details of how backup/restore should be implemented, it's
clear that whatever design one chooses, a necessary step is to have the user
(or higher level controller) create a PVC that points to the backup they want to
restore, and have the data appear in that volume somehow.

One can imagine backups simply being a special case of snapshots, in which case the
existing design is sufficient, but if you want anything more sophisticated, there
will inevitably be a new object type that represents a backup. While it's arguable
that backup should be something CSI plugins should be exclusively responsible for,
one can also argue that generic backup tools should also exist which can backup
and restore all kind of volumes. Those tools will be apart from CSI plugins and
yet need a way to populate volumes with restored volumes.

It's also likely that multiple backup/restore implementations will be developed,
and it's not a good idea to pick a winner at the Kubernetes API layer. It makes
more sense to enable developers to try different approaches by making the API allow
restoring from various kinds of things. 

### Implementation Details/Notes/Constraints

As noted above, the proposal is extremely simple -- just remove the validation on
the `DataSource` field. This raises the question of WHAT will happen when users
put new things in that field, and HOW populators will actually work with so small
a change.

It's first important to note that only consumers of the `DataSource` field today
are the various dynamic provisioners, most notably the external-provisioner CSI
sidecar. If the external-provisioner sidecar sees a data source it doesn't
understand, it simply ignores the request, which is both important for forward
compatibility, and also perfect for the purposes of a data populator. This allows
developers to add new types of data sources that the dynamic provisioners will
simply ignore, enabling a different controller to see these objects and respond
to them.

I will leave the details of how data populators will work for another KEP. There
are a few possible implementation that are worth considering, and this change
is a necessary step to enable prototyping those ideas and deciding which is
the best approach.  

### Risks and Mitigations

Clearly there is concern that bad things might happen if we don't restrict
the contents of the `DataSource` field, otherwise the validation wouldn't
have been added. The main risk that I'm aware of is that badly-coded dynamic
provisioners might crash if they see something they don't understand.
Fortunately, the external-provisioner sidecar correctly handles this case,
and so would any other dynamic provisioner designed with forward compatibility
in mind.

Removing validation of the field relinquishes control over what kind of
data sources are okay, and gives developers the freedom to decide. The biggest
problem this leads to is that users might attempt to use a data source that's
not supported (on a particular cluster), and they won't get any feedback
telling them that their request will never succeed. This is not unlike a
situation where a storage class refers to a provisioner that doesn't exist,
but it's something that will need to be solved eventually. 

Security issues are hard to measure, because any security issues would be the
result of badly designed data populators that failed to put appropriate
limits on user's actions. Such security issues are going to be present with
any new controller, though, so they don't seem relevant to this change. The
main thing to realize is that the `DataSource` field is a "local" typed
object reference, so no matter what, the object in that field has to either
be in the same namespace as the PVC that references it, or it must be a
non-namespaced object. This seems like an appropriate and desirable
limitation for security reasons.

If we think about who can install populators, the RBAC required for a
populator to operate requires at minimum, the ability to either create or
modify PVs. Also the CRD for the data source type needs to be installed.
This means that populators will generally be installed by cluster admins
or similarly-powerful users, and those users can be expected to understand
the uses and implications of any populators they chose to install. 

## Design Details

### Test Plan

The minimal test for this feature is to create a PVC with a data source
that's not a PVC or snapshot, and verify that the data source reference
becomes part of the PVC API object. Any very simple CRD would be okay
for this purpose. We would expect such a PVC to be ignored by existing
dynamic provisioners.

### Graduation Criteria

#### Alpha -> Beta Graduation

- Before going to beta, we need a clear notion of how data populators should
  work.
- We will need a simple and lightweight implementation of a data populator
  that can be used by the E2E test suite to exercise the functionality.
- Automated tests that create a volume from a data source handled by a
  populator, to validate that the data source functionality works, and that
  any other required functionality for data population is working.
- We will need to see several implementations of working data populators that
  solve real world problems implemented in the community.

#### Beta -> GA Graduation

- Distributions including data populators as part of their distros (possibly
  a backup/restore implementation layered on top)
- Allowing time for feedback

### Upgrade / Downgrade Strategy

Data sources are only considered at provisioning time -- once the PVC becomes
bound, the `DataSource` field becomes merely a historical note.

On upgrade, there are no potential problems because this change merely
relaxes an existing limitation.

On downgrade, there is a potential for unbound (not yet provisioned) PVCs to
have data sources that never would have been allowed on the lower version. In
this case we might want to revalidate the field and possibly wipe it out on
downgrade. 

### Version Skew Strategy

No issues

## Implementation History

- The idea of data populators has been discussed abstractly in SIG-storage
  since 2018 at least.
- John Griffith did the original work to propose something like this, but
  that work got scaled down to just PVC clones.
- Ben Swartzlander picked up the populator proposal developed 2 prototypes
  in December 2019.
- New KEP proposed January 2020
