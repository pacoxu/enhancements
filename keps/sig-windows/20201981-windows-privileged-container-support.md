<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# KEP-1981: Windows Privileged Containers and Host Networking Mode

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
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
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. KEP editors and SIG Docs
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->
Privileged containers are containers that are enabled with similar access to the host as processes that run on the host directly. With privileged containers, users can package and distribute management operations and functionalities that require host access while retaining versioning and deployment methods provided by containers. Linux privileged containers are currently used for a variety of key scenarios in Kubernetes, including kube-proxy (via kubeadm), storage, and networking scenarios. Support for these scenarios in Windows currently requires workarounds via proxies or other implementations. This proposal aims to extend the Windows container model to support privileged containers. This proposal also aims to enable host network mode for privileged networking scenarios. Enabling privileged containers and host network mode for privileged containers would enable users to package and distribute key functionalities requiring host access.


## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->
The lack of privileged container support within the Windows container model has resulted in separate workarounds and privileged proxies for Windows workloads that are not required for Linux workloads. These workarounds have provided necessary functionality for key scenarios such as networking, storage, and device access, but have also presented many challenges, including increased available attack surfaces, complex change and update management, and scenario specific solutions. There is significant interest from the community for the Windows container model to support privileged containers and host network mode (which enable pods to be created in the host’s network compartment/namespace, as opposed to getting their own) to transition off such workarounds and align more closely with Linux support and operational models.

Furthermore, since kube-proxy cannot be run as a privileged daemonset, it must either be run with a proxy or directly on the host as a service. In the case that it is run as a service, the admin kube config must be stored on the Windows node which poses a security concern. This is also true for networking daemons such as Flannel.


### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->
- To provide a method to build, launch, and run a Windows-based container with privileged access to host resources, including the host network service, devices, disks (including hostPath volumes), etc.
- To enable access to host network resources for <b>privileged</b> containers and pods with host network mode


### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->
- To provide access to host network resources for <b>non-privileged</b> containers and pods          
- To provide a privileged mode for Hyper-V containers, or a method to run privileged process containers within a Hyper-V isolation boundary. This is a non-goal as running a Hyper-V container in the root namespace from within the isolation boundary is not supported. 
- To enable privileged containers for Docker. This will only be for Containerd


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. The "Design Details" section below is for the real
nitty-gritty.
-->

### Use case 1: Privileged Daemon Sets
Privileged daemon sets are used to deploy networking (CNI), storage (CSI), and device plugins, kube-proxy, and other agents to Linux nodes. Currently, similar set-up and deployment operations utilize wins or proxies (i.e. CSI-proxy, HNS-Proxy) for Windows nodes. With Windows privileged containers, privileged daemon sets will also be available to deploy desired plugins and agents to Windows nodes. For networking scenarios, host network mode will enable these privileged deployments to access and configure host network resources. 

Enablement of both host network mode and privileged containers will allow users to configure network policies between pods and the hosts. Privileged containers will also enable service mesh support on Windows by enabling the creation of an init container that can configure HNS. 

### Use case 2: Node plugin containers
Windows privileged containers would also enable deployment of single privileged containers to Windows nodes. Other functionalities that could be enabled with privileged containers in or out of daemon set deployment include device enumeration, monitoring add-ons, among others. 

Some interesting scenario examples:
- Cluster API
- CSI Proxy
- Logging Daemons


### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->
- Host network mode support is only targeted for privileged containers and pods.
- Privileged pods can only consist of privileged containers. Standard Windows Server containers or other non-privileged containers will not be supported. This is because containers in a Kubernetes pod share an IP. For the privileged containers with host network mode, this container IP will be the host IP. As a result, a pod cannot consist of a privileged container with the host IP and an unprivileged Windows Server container(s) sharing a vNic on the host with a different IP, or vice versa.
- We are currently investigating service mesh scenarios where privileged containers in a pod will need host networking access but run alongside non-privileged containers in a pod. This would require further changes and investigation.


### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->
Most of the fundamental changes to enable this feature for Windows containers is dependent on changes within hcsshim, which serves as the runtime (container creation and management) coordinator and shim layer for containerd. However:
- Several upstream changes are required to support this feature in Kubernetes, including changes to containerd, OCI, CRI, and kubelet. The identified changes include (see CRI and Kubelet Implementation Details below for more details on changes):
    - Containerd: enabling host network mode for privileged containers and pods ([working prototype demo](https://drive.google.com/file/d/1WQO2NDX34Z1FPbW7jEymhcPMY4AZWQSE/view)). Prototype is done using containerd runtimehandler but this proposal is to use cri-api.
    - OCI spec: https://github.com/opencontainers/runtime-spec
        - [TBD]
    - CRI-api:
        - Adding a privileged field to the runtime spec and pass through to containers
        - Pass security context and flag of runtime spec to podsandbox spec (not currently supported, open issue: https://github.com/kubernetes/kubernetes/issues/92963) 
    - Kubelet: Pass privileged flag and windows security context to runtime spec and other PSP changes (see below).
- There are risks that changes at each of these levels may not be supported. 
    - If containerd changes are not supported, host network mode will not be enabled.This would restrict the scenarios that privileged containers would enable, as CNI plugins, network policy, etc. rely on host network mode to enable access to host network resources.
    - If CRI changes to enable a privileged flag are not supported, there would be a less-ideal workaround via annotations in the pod container spec. 
    - The CRI changes may make an annotation in the OCI spec until the OCI updates are included.

### PSP support 
Additionally, privileged containers may impact other pod security policies (PSPs) outside of allowPrivilegedEscalation. We will provide guidance similar to [Pod Security Standards](https://v1-18.docs.kubernetes.io/docs/concepts/security/pod-security-standards/) for Windows privileged containers.  There is an [analysis for non-privileged containers](https://github.com/kubernetes/kubernetes/issues/64801#issuecomment-668897952) which can be augmented with the details below.  There is an open question if PSP should be updated or the recommendation is to use an out of tree enforcement tool such as GateKeeper.  The anticipated impacted PSPs include:

<table>
  <tr>
   <td>Use case 
   </td>
   <td>Field name
   </td>
   <td>Applicable
   </td>
   <td>Scenario
   </td>
   <td>Priority
   </td>
  </tr>
  <tr>
   <td>Running of privileged containers
   </td>
   <td>privileged
   </td>
   <td>yes
   </td>
   <td>Required.
   </td>
   <td>Alpha 
   </td>
  </tr>
  <tr>
   <td>Usage of host namespaces
   </td>
   <td>HostPID, hostIPC
   </td>
   <td>no
   </td>
   <td>No support for pid and ipc since jobs don’t provide isolation at that level.
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>Usage of host networking and ports
   </td>
   <td>hostnetwork
   </td>
   <td>yes
   </td>
   <td>Will be in host network by default initially. Support to set network to a different compartment may be desirable in the future.
   </td>
   <td>Beta
   </td>
  </tr>
  <tr>
   <td>Usage of volume types
   </td>
   <td>Volumes
   </td>
   <td>no
   </td>
   <td>Not applicable.
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>Usage of the host filesystem
   </td>
   <td>Allowed host paths
   </td>
   <td>no
   </td>
   <td>Job objects have full access to write to the root file system. Current design does not have a way to control access to read only.
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>Allow specific FlexVolume drivers
   </td>
   <td>Flex volume
   </td>
   <td>no
   </td>
   <td>Not applicable.
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>Allocating an FSGroup that owns the pod's volumes
   </td>
   <td>Fsgroup (file system group)
   </td>
   <td>no
   </td>
   <td>The privileged container can be tied to run as a particular user that determines access to different fsgroups.
   </td>
   <td>N/A 
   </td>
  </tr>
  <tr>
   <td>The user and group IDs of the container
   </td>
   <td>Runasuser, runasgroup, supplementalgroup
   </td>
   <td>no
   </td>
   <td>Assigning users to groups would have to occur at node provisioning, or via a privileged container deployment.
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>Allowprivilegedescalation, default
   </td>
   <td>no
   </td>
   <td>Privilege via job objects is not granularly configurable.
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>Linux capabilities
   </td>
   <td>Capabilities
   </td>
   <td>no
   </td>
   <td>Windows OS has a concept of “capabilities” (referred to as “privileged constants” but they are not supported in the platform today.
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>Restrictions that could be applied to Windows Privileged Containers
   </td>
   <td>Other restrictions for job objects
   </td>
   <td>TBD
   </td>
   <td>There are restrictions that could be enabled via the job object, i.e. <a href="https://docs.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_basic_ui_restrictions">UI restrictions</a>
   </td>
   <td>N/A
   </td>
  </tr>
  <tr>
   <td>Use GMSA with privileged containers
   </td>
   <td>GMSA – would need to implement
   </td>
   <td>yes
   </td>
   <td>Required for auth to domain controller.
   </td>
   <td>GA
   </td>
  </tr>
</table>


## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->
### Overview
Windows privileged containers will be implemented with [Job Objects](https://docs.microsoft.com/en-us/windows/win32/procthread/job-objects), a break from the previous container model using server silos. Job objects provide the ability to manage a group of processes as a group, and assign resource constraints to the processes in the job. Job objects have no process or file system isolation, enabling the privileged payload to view and edit the host file system with the correct permissions, among other host resources. The init process, and any processes it launches or that are explicitly launched by the user, are all assigned to the job object of that container. When the init process exits or is signaled to exit, all the processes in the job will be signaled to exit, the job handle will be closed and the storage will be unmounted.

![Privileged Container Diagram](Privileged.png)

#### Networking
- The container will be in the host’s network namespace (default network compartment) so it will have access to all the host’s network interfaces and have the host's IP as well.
#### Resource Limits
- Resource limits (disk, memory, cpu count) will be applied to the job and will be job wide. For example, with a limit of 10 MB is set for the job, if every process in the jobs memory allocations added up exceeds 10 MB this limit would be reached. This is the same behavior as other Windows container types. These limits would be specified the same way they are currently for whatever orchestrator/runtime is being used.
#### Container Lifecycle 
- The container's lifecycle will be managed by the container runtime just like other Windows container types.
#### Container users 
-  The privileged container should be able to run as any user that's available on the host or in the domain of the host machine. Password accounts are being investigated.
#### Container Mounts
- Directory mounts (i.e. directory C:\test mapped to C:\test in the privileged container) are still being investigated. Note that file system mapping in the current implementation would expose both the host filesystem and the container filesystem to the processes in the privileged container. We are still investigation how external mounts such as secrets or storage accounts would be added. The default visibility of all the host filesystem may present backwards compatibility challenges in the future if host path mounting is enabled in Windows. Currently the only method to restrict would be via the user in a way similar to [UID or GID](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) in the PSP. Currently however, this host path mounting is unlikely to be enabled in the future as a restriction of Job Objects.  
#### Container Images
- A slim base image may be required to satisfy hierarchy requirements from HCS. It was found that the graphdriver calls expect certain files to be in a windows image when un-tarring. These files were found to be simply several registry hive files in /windows/system32/config (which can be empty).
- We are currently targeting to release a simple image alongside this feature including the above mentioned files, which can be used instead of traditional Windows base images of server core or otherwise. Server core and others that container the necessary files would also work as privileged container base images. 
#### Container Image Build/Definition
- This is another ongoing area of investigation and open to feedback. 
 
### CRI Implementation Details
We will need to add a privileged field to the runtime spec. We can model this after the Linux pod security context and container security context that is a boolean that is set to “true” for privileged containers. References:
- [Linux SandboxSecurityConfig](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1alpha2/api.proto#L293)
- [LinuxSandboxSecurityContext](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1alpha2/api.proto#L28)
- [LinuxContainerSecurityContext](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1alpha2/api.proto#L612)

In windows, the same privileged field would need to be added in the windows pod security context (open issue) and the container security context will need to be augmented with the privileged flag.

The new WindowsPodSandboxConfig:
```
message WindowsPodSandboxConfig {.
	WindowsSandboxSecurityContext security_context = 1;
}
```

A new WindowsSandboxSecurityContext:
```
message WindowsSandboxSecurityContext {
	string run_as_user = 1;
	bool privileged = 2;
}
```

The new field for the WindowsContainerSecurityContext:
```
message WindowsContainerSecurityContext {
	string run_as_username = 1;
	string credential_spec = 2;
  bool privileged = 3;
}
```
### Kubelet Implementation Details

#### <b>Privileged Flag</b>
There are no kubernetes API changes required.  Kubelet can use the privileged flag from SecurityContext api and pass that to the new privileged flag in the CRI layer.  

Add functionality to Kuberuntime_sandbox to split out the linux sandbox creation and add windows sandbox creation (WIP PR). 

https://github.com/kubernetes/kubernetes/blob/a9f1d72e1de6450b890a0c0e451725468f54f515/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L136 

#### <b>Host Network Mode</b>

If using Windows privileged containers the Host Network Mode will also be enabled, as the pod will automatically get the host IP.  The PodSpec has a hostNetwork boolean field:

https://github.com/kubernetes/kubernetes/blob/d20fd4088476ec39c5ae2151b8fffaf0f4834418/pkg/kubelet/container/helpers.go#L227-L229

https://github.com/kubernetes/kubernetes/blob/a9f1d72e1de6450b890a0c0e451725468f54f515/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L98 

In the case of windows this field would need to be addressed.  During the alpha stage the proposal for this field would be to add documentation that note the field is only applicable to Linux.  In addition the check for hostnetwork mode will need to be updated to check the privileged flag on windows PodSpec.  This will be implemented in the following way, with validation loosened in the future if required:
 - this pod must run on a windows host, and kubelets must reject it if not on windows hosts
 - all pods marked privileged on windows must have host network enabled, if not the pod does not validate

In beta there is a possibility to enable the privileged container to be a part of a different network component.  If this feature is enabled we will use the existing Pod HostNetwork field to enable/disable.

#### <b> ContainerD Support Only</b>
There are no plans to update Docker to have support for Privileged containers due to requirements for support HCSv2.  The additional fields for docker would be ignored. The default value should be false.

Need to investigate if the choice of base image would cause errors when pulling with docker.

#### <b> Feature Gates</b>
Although there are no changes required to the kubernetes API, the ability to pass the the privileged flag to the CRI should be behind a feature gate in Kubelet:  

https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-stages


### Test Plan


<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation, and anything particularly
challenging to test, should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

Alpha
- Preliminary analysis/testing of known kubernetes scenarios:
    - [List]

Beta 
- Testing and validation of key scenarios identified in the alpha analysis
- Test grids
  - Validate running kubeproxy as a daemon set
  - [List]


### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. The KEP
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

#### Removing a Deprecated Flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include 
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md
-->

Alpha
- Version of containerd: 	v1.5
- Version of Kubernetes: 	Target 1.20 or 1.21  
- Version of OS support: 	1809/Windows 2019 LTSC and 2004
- Alpha Feature Gate for passing privilege flag to CRI

Beta
- Go through PSP Linux test (e2e: validation & conformance) and make them relevant for Windows (which apply, which dont and where we need to write new tests). 
- Provide guidance similar to Pod Security Standards for Windows privileged containers
- Containerd: 			v1.5
- Kubernetes 			Target 1.21 or 1.22
- OS support: 			1809/Windows 2019 LTSC and 2004
- Beta Feature Gate for passing privilege flag to CRI

GA:
- [need feedback]
- Remove feature gate for passing privileged flag


### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

- Windows: This implementation requires no backports for OS components. 
- Kubernetes: [need feedback on]
- Containerd: [need feedback]

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

N/A 

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/20190731-production-readiness-review-process.md.

The production readiness review questionnaire must be completed for features in
v1.19 or later, but is non-blocking at this time. That is, approval is not
required in order to be in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.

-->

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**
  - [ ] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name:
    - Components depending on the feature gate:
  - [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

* **Does enabling the feature change any default behavior?**
  Any change of default behavior may be surprising to users or break existing
  automations, so be extremely careful here.

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?**
  Also set `disable-supported` to `true` or `false` in `kep.yaml`.
  Describe the consequences on existing workloads (e.g., if this is a runtime
  feature, can it break the existing applications?).

* **What happens if we reenable the feature if it was previously rolled back?**

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling or disabling feature
  gates. However, unit tests in each component dealing with managing data, created
  with and without the feature, are necessary. At the very least, think about
  conversion tests if API types are being modified.

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g., what if some components will restart
   mid-rollout?

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**
  Describe manual testing that was done and the outcomes.
  Longer term, we may want to require automated upgrade/rollback tests, but we
  are missing a bunch of machinery and tooling and can't do that now.

* **Is the rollout accompanied by any deprecations and/or removals of features, APIs, 
fields of API types, flags, etc.?**
  Even if applying deprecation policies, they may still surprise some users.

### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
  checking if there are objects with field X set) may be a last resort. Avoid
  logs or events for this purpose.

* **What are the SLIs (Service Level Indicators) an operator can use to determine 
the health of the service?**
  - [ ] Metrics
    - Metric name:
    - [Optional] Aggregation method:
    - Components exposing the metric:
  - [ ] Other (treat as last resort)
    - Details:

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  At a high level, this usually will be in the form of "high percentile of SLI
  per day <= X". It's impossible to provide comprehensive guidance, but at the very
  high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code

* **Are there any missing metrics that would be useful to have to improve observability 
of this feature?**
  Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
  implementation difficulties, etc.).

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  Think about both cluster-level services (e.g. metrics-server) as well
  as node-level agents (e.g. specific version of CRI). Focus on external or
  optional services that are needed. For example, if this feature depends on
  a cloud provider API, or upon an external software-defined storage or network
  control plane.

  For each of these, fill in the following—thinking about running existing user workloads
  and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?**
  Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
  focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)

* **Will enabling / using this feature result in introducing new API types?**
  Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)

* **Will enabling / using this feature result in any new calls to the cloud 
provider?**

* **Will enabling / using this feature result in increasing size or count of 
the existing API objects?**
  Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)

* **Will enabling / using this feature result in increasing time taken by any 
operations covered by [existing SLIs/SLOs]?**
  Think about adding additional work or introducing new steps in between
  (e.g. need to do X to start a container), etc. Please describe the details.

* **Will enabling / using this feature result in non-negligible increase of 
resource usage (CPU, RAM, disk, IO, ...) in any components?**
  Things to keep in mind include: additional in-memory state, additional
  non-trivial computations, excessive access to disks (including increased log
  volume), significant amount of data sent and/or received over network, etc.
  This through this both in small and large cases, again with respect to the
  [supported limits].

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**
  For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.

* **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

- Use Containerd Runtimehandlers and K8s RuntimeClasses - Runtimehandlers are using the prototype.  Adding the ability to the CRI provides kubelet to have more control over the security context and and fields that it allows through giving additional checks (such as runasnonroot) 

- Use annotations on CRI to pass privileged flag to Containerd - Adding the field to the CRI spec allows for the existing CRI calls to work as is. The resulting code is cleaner and doesn’t rely on magic strings.  There is currently a PR for adding the SecurityFields to the CRI API adding Sandbox level security support for window containers.  The Runasusername will be required for privileged containers to make sure every container (including pause) runs as the correct user to limit access to the file system.

## Open Questions
- What’s the future of plug-ins that will be impacted 
- What will be the future CSI-proxy and other plug-ins that will be impacted?
  - CSI-proxy and HNS-proxy are likely to be impacted
- Container base image support
  - Is “from scratch” required
  - Would a slimmer “privileged base image” be more desirable than using stand server core
- Container image build differences with traditional windows server and impacts on image use and distribution
- Should PSP be updated with latest checks or should out-of-tree enforcement tool be used?
  - PSP will be depreciated and documentation and guidance should be produced for [Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/). Implementations in out-of-tree enforcement should be favored and POC/impementation in gatekeeper would be a great way to demonstrate.
- Scheduling checks 
- Privileged containers in the same network compartment as the non-privileged pod, and otherwise init privileged containers may be able to still access the host network
