# KEP-3675: Pluggable Resource Management 
Container Compute Interface (CCI) Driver Extension

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
    - [Alpha Goals](#alpha-goals)
    - [Beta &amp; Post-Beta Goals](#beta--post-beta-goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Custom workloads, such as Telco/HPC/AI/ML](#custom-workloads-such-as-telcohpcaiml)
    - [Power optimization of workloads](#power-optimization-of-workloads)
    - [Research of new resource management patterns within the cloud](#research-of-new-resource-management-patterns-within-the-cloud)
    - [User-specific Drivers](#user-specific-drivers)
    - [Roles](#roles)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
    - [Compute Specification Option](#compute-specification-option)
      - [User API for Alpha: Policy-based resource specification - no scheduler impact](#user-api-for-alpha-policy-based-resource-specification---no-scheduler-impact)
      - [Options for User APIs in Post-Alpha:](#options-for-user-apis-in-post-alpha)
      - [CCI Resource Manager Architecture](#cci-resource manager-architecture)
      - [Allocation &amp; Container Removal Flow](#allocation--container-removal-flow)
      - [Post-Alpha &amp; GA Architectural Considerations](#post-alpha--ga-architectural-considerations)
  - [Test Plan](#test-plan)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha to Beta](#alpha-to-beta)
    - [Beta to GA](#beta-to-ga)
    - [Deprecation](#deprecation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [Operational Requirements for Alpha:](#operational-requirements-for-alpha)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Challenges and Considerations](#challenges-and-considerations)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes


[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Kubernetes compute management is tightly integrated with the Kubelet and the 
existing suite of resources managers including the topology manager, CPU manager, 
memory manager, and device manager. While these managers have added functionality that 
has addressed a varied set of use cases, they do present the community with several challenges.

Adding a new capability to one of these managers is slow moving and difficult 
due to their complex interactions, the level of prudence required given the potential
impacts, the implied necessity to make the extension work in as many scenarios as
possible, as well as the overhead heaped onto the sig-node maintainers. More 
details on the challenges discussed with the community have been captured in 
the [CPU Management Kubelet Use Cases & Current State document](https://docs.google.com/document/d/1U4jjRR7kw18Rllh-xpAaNTBcPsK5jl48ZAVo7KRqkJk).  

As a result, adding optimizations necessary to support some more niche use cases, 
to improve total cost of ownership for deployments with more exacting requirements 
on resources or to provide a mechanism to progressively roll-out the benefits of 
innovations related to newer CPU architectures is difficult and high cost to community resources.

This proposal aims to address the challenges by introducing a new Container Compute 
Interface (CCI). The CCI is conceptually equivalent to the [CSI](https://github.com/kubernetes/design-proposals-archive/blob/main/storage/container-storage-interface.md) in that it will 
delineate between the responsibility of the Kubelet and the responsibility of lower-level, 
compute focused drivers for resource assignment capabilities, such as CPU and memory.

We propose an iterative implementation approach:

In the first stage (alpha) we propose adding a new component to the kubelet called the CCIManager 
that focuses exclusively on providing a mechanism for pluggable CPU management policies (as a replacement
to the builtin CPUManager and its corresponding policies). Enablement of the CCIManager will be blocked by a feature gate, 
which, when enabled, will trigger all CPU Management decisions to flow through the CCIManager instead of the builtin CPUManager. 
As such, the CPUManager policy MUST be set to none if this new feature gate is enabled. This will
be enforced by the kubelet, causing it to "fail fast" if this invariant does not hold true.

For now, only a single CCI driver will be allowed to run on a node at any given time. Details of how we plan 
to enforce this requirement as well as how we plan to (temporarily) handle bootstrapping a node that does not yet have 
a pluggable CPU Management policy running can be found in Section [Throubleshooting]. In a later phase, we will provide a proper mechanism 
to handle bootstrapping as well as cases where a CCI driver becomes temporarily unavailable.

The implication of CCI and the addition of the proposed CCI Manager 
will be to allow compute capabilities such as CPU and memory to be managed outside
of the Kubelet via pluggable CCI Drivers.  This will allow users to augment the
current implementation of Kubelet managed resources with support for more complex
use-cases without having to modify the Kubelet. The CCI extensions can coexist 
with the existing CPU and memory allocation technique available to Kubernetes users today.

Although our alpha implementation provides an alternate route for managing CPUs through pluggable CPU management drivers, the long-term goal is to allow these drivers to be used side-by-side with the built-in policies. Pods will be able to choose which policy they want their CPUs allocated with, and the machinery in the kubelet will take care to ensure the proper CPU Management policy is invoked (be that from a builtin policy or from a driver).

These changes will allow the community to disaggregate the long-term 
stability and advancement of the Kubelet from the task of improving the compute 
resource management while keeping pace with the needs of 
specialized use cases and the advancements in the compute vendor ecosystem.

## Motivation

Users have varied workloads; the current set of available configurations
for CPU, memory, and topology remain limited.  Additionally, the number of managers
becoming internal to the Kubelet continues to increase; we should find a more 
dynamic and pluggable way of handling these resources.  

Operating systems work by allowing drivers and pluggable resources.  This includes various
policies in how CPU, memory, and devices are allocated.  Kubernetes can be viewed as being 
an the operating system of the cloud.  Allowing specialty modules to address the use cases
directly, rather than continuing to add complexity by continuing to modify the kubelet, 
will allow the greatest scope of abilities while halting continued increases of complexity 
within the core of the Kubelet.

Users would like to be able to address the following use cases:

* The ability to allow vendors to release vendor-specific managers for their hardware
  and provide an interface while not having vendor-specific code within Kubelet.
* Differentiate between types of cores and memory. <br>
  Note - Dynamic resouce allocation does this with regular resources today.  We seek to
  extend this ability.
* Differentiate between different configurations of cores and memory, for instance cores
  designated as performance versus those designated as efficiency cores 
* Have custom driver to optimize for particular types of workloads.  These drivers
  may be built for performance, power reduction, cpu savings, et cetera.  <br>
  Note:  Currently, there are very limited sets of available topology policies.  Every
  new policy must be approved and be lockstep with the Kuberenetes release process.
* Be able to hot-plug and test new resource managers.  
* Be able to remove some of the complexity with current setup and, over time, reduce
  the amount of code contained within Kubelet.  Instead, build a library with specific
  needs.
* Have a faster path to desired changes, without potentially impacting the core of 
  Kubernetes with every policy change. <br>
  Note that current solutions have been cropping up to allow for resource management
  outside the core Kubelet. These solutions require turning off the Kubelet functionality 
  and overriding current Kubelet allocation, such as [CPU Pooler](https://github.com/nokia/CPU-Pooler). 
  We should provide a path otherwise. Many of these solutions are custom to particular companies. 
  Many are not open source. The community should give a process to allow this functionality in core
  kubelet at a granular level, same as we have in many systems, such as HPC or telco.
* Leverage additional parts from the pod spec information which could help make allocation decisions
without the need to query the k8s controlplane.
* Be able to do research, with minimum toil, on new policies and resource management strategies

This design will also use gRPC, which is used for many other pluggable components within Kubernetes.  
The hope is that it will also, as a side effect, allow a path to simplify the Kubelet as it is 
today and move the complexity into external drivers.

### Goals

#### Alpha Goals

1.  Provide Initial infrastructure to support CCI Drivers and k8s Deployments requiring CCI Drivers.
1.  Provide a feature gate to enable CCI Resource Manager. The feature will require CPU Manager policy set to None.
1.  Provide a CCI test driver which can demonstrate CCI Driver implementation requirementes and several resource management use-cases for cpu.
1.  Provide documentation of the CCI Resource Manager and the provided test driver plus illustration of the currently covered use-cases, including a sample driver.
1.  Provide end-to-end tests using the sample driver.
1.  Support seamless k8s start.
1.  CCI Driver results (cpusets) are passed back to kubelet and allocated.
1.  Handle resource management state for cpu through CCI. 
1.  Support proper fail mechanisms of Pods if CCI driver is not available - Error message + retry 
1.  Continue cluster operation if CCI driver is not available
1.  Podspecs:  Will be able to support current pod specs.  While there may be additional 
extensibility in the future, the current pod specs will still work.
1.  Guarantee that cpu resources managed by CCI are visible for the scheduler to be able to 
    correctly assign nodes.  This can be limited as far as simple CPU counts for this version.

#### Beta & Post-Beta Goals

1.  Support standard pod deployments not requiring CCI Drivers through CPU & Memory Manager: all QoS types
1.  Support hint providers for topology manager
1.  Interoperability with Device Plugins, DRA, et cetera. 
1.  Identify and provide a minimal in-cluster operational core (from existing CPU Manager, Memory Manager, Topology Manager) either through in-tree libraries or through further KEPs.
1.  Gather feedback on the feature through surveys.
1.  E2E testing including any new components, such as memory and topology.
1.  Adding support for e2e tests in testgrid.
1.  Health identification mechanisms and automatic health-repair procedures for CCI Drivers.  
1.  Optimize on-node resource usage (approaches to avoid waste of resources because of fragmentation).
1.  Consider whether useful to support multiple CCI drivers.
1.  Consider optimal binpacking of resources (we may optimize resource utilization in Beta).
1.  Guarantee that cpu and memory resources managed by CCI are visible for the scheduler to be able to 
    correctly assign nodes.  This will be expanded as necessary, dependant on use cases, from the base
    group for alpha (topology, type, et cetera).

### Non-Goals

* Break any existing use cases for topology manager, memory manager, cpu manager, or 
device manager.
* Creating any more latency than there is today for scheduling:  We should be 
careful not to add any latency into scheduling over what exists today for default behavior.
* This KEP does not cover reservation of CPU and memory resources on the scheduler side.


## Proposal

This proposal contains a path to update the kubelet to support more nuanced
uses of cpu and memory and further on-node resource management enhancements.
The proposed Container Compute Interface gives users a way to manage these 
resources without having to either bypass or hack the kubelet CPU and memory
controllers. The CCI will allow for seamless integration of new and interesting
CPU and memory policies; this allows for innovation for better workload performance 
and sustainability for datacenters.

### User Stories

#### Custom workloads, such as Telco/HPC/AI/ML

Custom workloads often have a desire for a mix of types of cores. For instance, a workload
should be able to have some number of static cores and some number of shared cores.
A node should be able to allow both types of cores, without having to have one setting
or another, and be able to pull from these pulls accordingly. Additionally, there is
a need to have some high-priority cores for higher performance and other lower-priority
cores for other less-sensitive parts of a workloads. In these use cases, the workloads 
may also have particular types of NUMA splits required.

Users in these fields want fine-grained control over their resources. Users in this 
field need to be able to adapt clusters to the workload.  

#### Power optimization of workloads

Cores should be able to be quickly spun up or down according to resource requirements.
Additionally, the nodes should be as densely packed as possible while respecting resource
attributes. There should also be the ability to choose between efficiency cores and 
performance cores within newer architectures, according to workload requirements.

As energy becomes more expensive for datacenters, maximizing the use of resources is
desired. Users should be able to specify whether a workload needs the fastest available
cores or is content to run on more power efficient cores.

#### Research of new resource management patterns within the cloud

There are a variety of modifiers that can be placed around cores. Static cores,
isolated cores, shared cores, efficiency cores, and performance cores are only the
beginning of unexplored spaces.  Being able to experiment with various patterns in 
research without having to be an expert in how to modify Kubelet and its multiple 
internal managers is a decisive benefit to the research community.

Users, analog to the use case for Telco/HPC/AI/ML, similarly need fine-grained
control over their core placement decisions.

#### User-specific Drivers

A user may have very specific allocation patterns they require. This sort of capability
may be rare and not belong upstream in mainstream Kubernetes, but there should still
be a simple way to allow users to meet their specific experiments.

#### Roles

1. Cluster operator or a system administrator is responsible to enable CCIResourceManager feature gate on kubelet and install a CCI Driver.
1. CCI Drivers can be vendor specific or community drivers. The drivers are provided/implemented by vendors or the kubernetes community.
1. User specify a request of resources through policy or attribute-based request.

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

As the feature is stand-alone and behind a feature gate, this should not
impact standard Kubelet behavior.  In the event someone is using the CCI
Drivers, the alpha version will default to standard best effort QoS until
the CCI Driver is deployed.  After deployment, the management will be reliant on
the CCI Driver implementation.

## Design Details

#### Compute Specification Option

The CCI model combined with some of the capabilities introduced in the Dynamic 
Resource Allocation (DRA) KEP [3], offers the ability to transition compute 
resource allocation behavior from being a cluster-admin dominated configuration
to one that allows users with precise compute requirements to articulate the
compute attributes that they need. 

In several domains such Streaming, Telco, HPC, and AI/ML, users require
fine-grained compute resource control to maximize their key performance 
indicators.  They need an API option that can be made available through
upstream Kubernetes components and specific compute resource allocation 
drivers.

##### User API for Alpha: Policy-based resource specification - no scheduler impact

The first option which we will implement is a resource allocation configuration for Kubernetes
Pods through policy mechanism defined as annotation. The CCI Driver will process the policy for each pod/container. 

Example with 4 containers in the pod: 

annotation:
  cci.resources.alpha.kubernetes.io/pod: cci-default
  cci.resources.alpha.kubernetes.io/container.container1: cci-exclusive
  cci.resources.alpha.kubernetes.io/container.container2: cci-shared
  cci.resources.alpha.kubernetes.io/container.container3: other-policy

container1: 
  resources:
    requests:
      cpu: <>
      memory: <>
    limits:
      cpu: <>
      memory: <>

container2: 
  resources:
    requests:
      cpu: <>
      memory: <>
    limits:
      cpu: <>
      memory: <>    

Pros:
  * Policy per container/pod available as a user choice
  * New policies can be provided to users through CCI Drivers without kubelet impact
  * Resource quotas still supported 
Cons:
  * User is required to understand the policies

##### Options for User APIs in Post-Alpha:  

Attributed-based resource specification 
 
To realize an attribute-based mechanism for compute resources we are considering multiple options for post-alpha stage, such as:

Annotation-based: 

annotation:
  cci.resources.alpha.kubernetes.io/pod: cci-default
  cci.resources.alpha.kubernetes.io/container.container1: sibling-cores-required
  cci.resources.alpha.kubernetes.io/container.container2: no-sibling-cores
  cci.resources.alpha.kubernetes.io/container.container3: other-attributes

container1: 
  resources:
    requests:
      cpu: <>
      memory: <>
    limits:
      cpu: <>
      memory: <>
        
container2: 
  resources:
    requests:
      cpu: <>
      memory: <>
    limits:
      cpu: <>
      memory: <> 

Pros:
  * Does not require scheduler extension
  * Easy to process in CCI driver
  * No API extension required
  * Resource quotas still supported 
  * More precision vs. policy only approach

Cons:
  * Better suited for specialized use-cases
  * No attribute awareness in the scheduler (could be addressed through DRA driver-subject of post alpha)
  * Attributes are CCI driver specific (consider standartization within sig-node)

DRA-inspired Atrribute-Claims:

This example presents a set of per-core attributes in a claim that enables precision beyond
a policy-only based approach: 

  # A resource request which consists of:
  # 2 exclusive cores (no other processes running on them)
  # 	* pairs of cores shall be SMT siblings
  # 	* they shall run at 2.1 GHz
  # 	* there are no IRQs on the cores
  # 6 exclusive cores (no other processes running on them)
  # 4 shared cores (other processes can run on them)
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-compute-claim-parameters
    namespace: default
  data:
    cores: |
      2
      6
      4
    compute_attributes: |
      exclusive, smt-sibling-required, 2.1GHz, no-irq
      exclusive
      shared
  apiVersion: resource.k8s.io/v1alpha1
  kind: ResourceClaimTemplate
  metadata:
    name: example-compute-claim-parameters-template
    namespace: default
  spec:
    metadata:
      labels:
        app: inline-resource
    spec:
      resourceClassName: CPUressources
      parametersRef:
        kind: ConfigMap
        name: example-compute-claim-parameters


Pros:
  * Can be used to expose attribute-based configurations

Cons:
  * Needs a solution for resource quotas (possibly through additional resource specificationin pod spec)
  * Requires Controller which can deal with CPU and memory resources
  * Scalability 

##### CCI Resource Manager Architecture

The extension consists of an optional CCI driver inside Pod spec which
indicates if a CCI driver exists, then it shall process compute resources.
The association betweed pod and CCI Driver can be also realised as a resource class 
argument (will be considered as an option during the implementation). 

In the alpha version of the CCI Resource Manager applies drivers to manage resource
allocations for Pods. Post-alpha we will consider if we need to explicitly mark which pods are to be handled by an allocation 
driver.  The overall architecture consists of a single CCI Resource Manager component – the CCI Resource
Manager inside the container manager, an optional Pod spec argument to identify which 
Pods to be processed by the manager(post-alpha), a compute resource store – a component used to store
the resource sets(cpusets, memory affinity configuration) for the resource drivers. The CCI Resource Manager 
is enabled via a feature gate. After turning on the feature gate all pod allocation requets for cpu 
and memory will be handled by the CCI Resource Manager.

![image](CCIArchAlpha.jpg)<br>
Figure 1: CCI Resource Manager Architecture inside Kubelet

    Pod Spec “CCIDriverName Extension”
    type PodSpec {
       …
    	 // CCIResourceDriver is an optional parameter which can identify if a Pod resource manager (cpu an memory)
	    // has to be handled by a pluggable CCI resource driver 
	    // This is an alpha field and requires enabling the
    	 // CCIResourceManager feature gate.
	    //
    	 // This field is immutable.
	    //
	    // +featureGate=CCIResourceManager
	    // +optional
	    CCIResourceDriver string `json:"cciResourceDriver,omitempty" protobuf:"bytes,40,opt,name=cciResourceDriver"`
     …}

For alpha phase CCIResource Manager will be used for all pods as long as the CCIResourceManager feature gate is enabled. Post-alpha we consider two options to associate CCI Drivers with pods: a new attribute in resource class object inspired by DRA or a pod spec option cciResourceDriver as shown above.

1. CCI Resource Manager<br>
These changes will need a separate resource manager component inside the container 
manager which will handle drivers management tasks, including registration and 
invocations of registered drivers.  These tasks will be managed via grpc for Pods who 
require one or more resource management driver(s) to get successfully allocated.  In 
the alpha version we provide two entry points  on admission, addContainer 
and removeContainer lifetime events (drivers have to implement Admit, AddResource 
and RemoveResource handlers accordingly). 

2. CCI Resource Store<br>
The CCI Resource Manager uses a store to keep track of resourcesets which can include 
cpusets and memory allocated by CCI Drivers. This information is gathered 
per container. The gathered information will be used to provide data for on-node available resources. The store offers the 
following interface to manage resource sets.

        +AddResource(Pod, container, resourceset)
        +RemoveResource(container)
        +AvailablResources(): resourceset


4. CCI Driver Interface<br>
The initial interface of resource management drivers is very simple and consists 
of three functions:

        +CCIAdmit(Pod, container, cci spec, *available resources) : <err, resourceset>
        +CCIAddContainerResource (Pod, container, cgroup, containerid): <err>
        +CCIRemoveContainerResource (containerid): err

`CCIAdmit` function provides information if a given container belonging to a
Pod and having a specific CCI spec can be admitted to next stage of the allocation 
pipeline. The admission will return a reserved resource-set or error. In case of 
successful admission the resource set will be stored in the CCI Storeand allocated. In the case of failure the error is reported
back to user and the Pod allocation is cancelled. In the admission function we pass a list of available cpusets which can allow us to inform drivers about system-reserved resources.

`CCIAddContainerResource` function is called before container start with a given 
Pod name and container name, cgroup of Pod and the container id. The driver than
performs an allocation of the admitted resource-set.  In case of failure of the
driver and error is returned.

`CCIRemoveContainerResource` function is called to free the cpu resources taken
by a container. The functions returns nil if the operation was successful or an
error if the operation failed on the driver side.

    /* 
    CCIDriver is a grpc service interface for CCI resource kubelet drivers.
    The interface provides admission, allocation and cleanup entrypoints.
    Drivers are registered  as kubelet plugins and will be called by the 
    CCI Resource Manager for pods which are associated with this driver.
    */
    service CCIDriver {
      //cciAdmit admission function to reserve resources for a given container
      rpc CCIAdmit(AdmitRequest) returns (AdmitResponse) {}
  
      //cciAddContainerResource allocation entry point for admitted containers
      rpc CCIAddContainerResource (AddResourceRequest)
        returns (AddResourceResponse) {}

      //cciRemoveContainerResource clea ubelet dted resources for a given container
      rpc CCIRemoveContainerResource (RemoveResourceRequest)
        returns (RemoveResourceResponse) {}
    }

    message AdmitRequest{
        // currently available cpus
        string availablecpus = 1;
        // pod identfier 
        string pod = 2;
        // container identfier
        string container = 3;
        // cci spec for the container, subset of podspec
        string cci_spec = 4;
    }

    message ResourceSet{
        // admitted cpuset
        string cpuset = 1;
        // flag if exclisve
        bool exclusive = 2;
    }

    message AdmitResponse {
      // allocated resource set
      ResourceSet rset = 1;
      // error object 
      string err = 2;
    }

    message AddResourceRequest{
        // pod cgroup
        string cgroup = 1;
        // pod identifier
        string pod = 2;
        // container identifier
        string container = 3;
        // container id
        string container_id = 4;
    }



    message AddResourceResponse {
      // error object 
      string err = 1;
    }

    message RemoveResourceRequest {
        // id of the container to be removed
        string container_id = 1;
    }

    message RemoveResourceResponse {
        // error object
        string err = 1;
    }

5.	CCI Drivers Factory API<br>
The KEP includes a new staged API which enabled the CCI driver creation. The 
API can be used by a driver implementor to start a driver and automatically
register it against Kubelet's CCI Resource Manager.

6.	CCI Drivers Registration<br>
The CCI driver registration can be handled by the kubelet plugin framework. 
This approach is already used with device plugins and DRA plugins. The approach
will be sufficient the cover drivers registration, status and health-check functionality.


##### Allocation & Container Removal Flow

Admission and adding a Container:<br>
![image](CCIAllocSeqAlpha.jpg)<br>
Figure 2: Sequence of container allocation which uses CCI driver<br><br>


The lifetime events are triggered by the container manager and internal lifecycle manager in 
exact order for admitting, adding, and removing containers. As shown on Figure 2 an admit 
container is invoked for containers inside pods requiring a resource drivere.
On success, the resource set result is added to the CCI 
store. If the operation 
fails an error will be reported back to the user (in alpha we will go back to best-effort QoS). 
All blocking rpc calls are configured in alpha with a reasonable timeout. After getting an actual container id, the internal
lifecycle loop triggers the add container call.

Container Removal:<br>
![image](CCIRemoveSeqAlpha.jpg)<br>

Figure 3: Container removal sequence diagram involving CCI drivers<br><br>
The container removal case is described as a sequence in Figure 3. After registering
a removal event in the internal container lifecycle, the CCI manager is triggered 
and invokes the CCI Driver to free any resources takes by the container. On
success, the CCI store will be also cleaned and a new available resource set will 
be updated. All blocking rpc calls are configured with
a reasonable timeout in alpha.

##### Post-Alpha & GA Architectural Considerations

After the initial alpha implementation we will consider the integration of the 
Container Compute Interface “CCI” with exisiting CPU and Memory manager stack
This can be achieved either by using already existing managers and introducing state 
synchronization or through a common code base which can be invoked 
from within the CCI Resource Manager.

A possible integation with topology manager could follow by implementing the topology hints 
interface for the CCI Resource Manager which will use data extracted from CCI Store. This will 
require such data to be provided by CCI Drivers and the CCI store has to be extended to 
be able to handle it correctly for serving the corresponding topology hints calls.

 
### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Unit tests

###### Alpha

*	CCI Resource Manager: target code cvg >=80%
*	CCI Store: target code cvg >=80%
*	CCI Drivers Factory API: target code cvg >=80%

###### BETA

*	CPU Manager None and Static Policy Integration with CCI
* Pod Admission Race tests
* Introduce fail-safety tests
*	Performance/Scalabilty tests


##### Integration tests

###### Alpha

* CCI Resource Manager Integration test for cpu use-cases: driver-based allocation and best-effort QoS 
* CCI State Consistency integrateion test
* Kubelet restart integration test
* System-reserved resources test


###### BETA

* CPU Manager None, Static Policy Integration with CCI
* CPU, Memory, Topology and CCI Resource Manager Integration test
*	Further integration tests with Device Manager and DRA
*	Integration test including static QoS and driver-based allocation

##### e2e tests

###### Alpha

*	E2E tests including a test CCI Driver
* E2E tests for showcasing resource allocations for cpu through CCI

###### BETA

* End-to-End tests to cover all cci resource allocation use-cases
* End-to_End tests with device plugins and DRA
* Performance and resource utilization tests


### Graduation Criteria

#### Alpha to Beta

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled
- Integrate API feedback from users and community
- Proven cross-components consistency (ideally via tests)
- Handling of topology manager and memory manager use-cases
- Finish Identified Code refactoring of common building blocks (look at common pieces in all plugin-frameworks  in kubelet) 
- Look to what makes sense to leave inside Kubelet due to latency and use case requirements. Introduce community CCI drivers repo
Similar to Kubernetes Scheduler Plugin repo
- Have CCI Drivers specific to common use cases or environments
- Smooth integration with scheduler extensions/ plugins …

#### Beta to GA

- Gather feedback from developers and surveys and integrate it
- Successful adoption across users
- Proven Components Correctness
- Performance tests completion
- N examples of real-world usage

#### Deprecation


### Upgrade / Downgrade Strategy

TBD in Beta
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

### Version Skew Strategy

TBD in Beta
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

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### Alpha
Before having a driver in the system, or if one is not available, we default to
best effort QoS for incoming pods.

A newly installed CCI Driver becomes responsible to handle incoming pods.  It is 
responsible for reading the state from the kubelet on startup.

#### Beta
Pods without an available attached CCI Driver we will fail and be retried until
the driver comes up or failed after a configurable timeout.

#### Operational Requirements for Alpha:

* Enable CCI Feature Gate
* Disable Memory manager 
* Disable Topology Manager
* Set CPU Manager policy to None


###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: CCIResourceManager
  - Components depending on the feature gate: CCIResourceManager, Kubelet
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

###### Does enabling the feature change any default behavior?

No


###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes, a deletion of pods requiring cci driver will be recommended

###### What happens if we reenable the feature if it was previously rolled back?

Running pods should not be impacted.  New pods should be handled by the CCI Driver.

###### Are there any tests for feature enablement/disablement?

Yes.

### Rollout, Upgrade and Rollback Planning

The usual Kubernetes upgrade and downgrade strategy applies for in-tree components. 
Vendors must take care that upgrades and downgrades work with the drivers that
they provide to customers
  
Pods which require CCI Driver will fail starting and report an error due to driver 
unavailability.  If a CCI Driver fails, Pods which were about to be allocated
that were tied to the driver will fail. Based on requirements a fallback mechanism 
can be implemented where such Pods fallback to a given std. QoS Model.

###### How can a rollout or rollback fail? Can it impact already running workloads?

Failure of CCI drivers will have impact only over pods requiring CCI driver for 
resource management. All other pods will not be impacted. THe impacted pods will
an error message showing failure of the driver.


###### What specific metrics should inform a rollback?

Unhealthy CCI drivers which can't get to a ready state and repeated failures 
to allocate pods in cases of free resources.


###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

TBD in Beta


###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

TBD in Beta


### Monitoring Requirements

TBD in Beta


###### How can an operator determine if the feature is in use by workloads?

Pods will be running on the cluster which are associated to CCI drivers. 
Additionally, available resources will be updated.


###### How can someone using this feature know that it is working for their instance?

- [ ] Events
  - Event Reason: 
- [x] API .status
  - Condition name: CCI Driver readiness and CCI Status field can be introduced
  - Other field: 
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Pods not using CCI Drivers continue to work as before. Consistent state handling for 
pods which require a CCI Driver.


###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->
TBD in Beta
- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->
TBD in Beta

### Dependencies

None
<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

A CCI Driver (daemonset) will be required to handle pods which need CCI Driver resource management.

### Scalability

The CCI Resource Manager approach resembles device manager protocol. The scalability and performance impact will be similar to the case of handling device plugins.

Further performance benchmarks will be done in Beta Phase


###### Will enabling / using this feature result in any new API calls?

For Pods requiring CCI Driver the admission, container add and container removal operations will be handled via 
API call to external CCI Driver. Remaining pods should be handled as before.

###### Will enabling / using this feature result in introducing new API types?

A helper API to create CCI Drivers (CCI Driver Factory) can be integrated as staged repo.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No 

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

This will not be true for Alpha.  For Beta, where there may be multiple drivers, it may be prudent to consider this.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No  

### Troubleshooting
  
In alpha phase we propose to handle bootstrapping and failures due unavailable drivers 
by going back to best-effort QoS. In post-alpha phase we will consider solutions of the bootstrapping and driver failure problem
though an association mechanism between pods and drivers. If a pod does not require a driver CCI Resource Manager can 
provide allocation for this pod without and driver invocations, which will allow a smooth bootstrapping procedure of the cluster.

If a CCI Driver fails, Pods associated with a driver which were about to be allocated will fail. Based on 
requirements a fallback mechanism can be implemented where such Pods fallback to 
a given std. QoS Model. 

![image](CCITroubleshooting.jpg)<br>

Figure 4: Troubleshooting Flows: Handling of pods requiring CCI driver and standard pods wihtout CCI Driver requirements in the case of unhealthy driver.<br><br>

To illustrate this lets consider some of the flows described on Figure 4. We distriguish 
between pods handled by CCI drivers(group 1 pods) and standard Pods(group 2 pods). Until the CCI driver is not started the pod admission of group 1 pods will fail with an admission error due to driver unavailabilty. Group 2 pods are not impaceted by and they
can be deployed on the cluster. After the CCI Driver comes online  the admission of 
group 1 pods can continue. The admission transactions of group 1 and 2 are mutually exclusive. In later time the driver becomes unhealthy and the admission of second pod 
from group 1 will fail.

###### How does this feature react if the API server and/or etcd is unavailable?

Feature lives completly in kubelet and will not be impacted directly. 

###### What are other known failure modes?

TBD

###### What steps should be taken if SLOs are not being met to determine the problem?

We plan to provide sufficient logging information inside kubelet log to help determine
problems with CCI Resource Management Stack. Additionally CCI Driver implementations should 
consider integrating proper logging mechanism and error messages to help determine any 
issues during operation.

## Challenges and Considerations
###### Checkpoint assignments

The driver must be able to pick up where it left off in case the Kubelet restarts for any reason. With regards to cpusets, the CPU Manager can reassign those values in the event of a Kubelet restart as the CPU Manager state also keeps track of Pods covered by CCI drivers. Information specific to the driver can be handled by the driver itself using a checkpoint file.

## Implementation History

TBD

## Drawbacks

TBD in Beta.

## Alternatives

  We could choose, instead, to continually extend existing Kubelet managers.  This is
  already complicated and becomes more so as users want more and more specialty use
  cases.  Additionally, chip companies are coming up with increasingly complicated 
  architecture and the community will not want to spend time and resources supporting
  odd changes particular for particular chipsets.  Rather, we should choose to reduce
  complexity within the Kubelet over time.

  One example how current managers could be lavareged for that can be seen in the [CPU Pooler](https://github.com/nokia/CPU-Pooler) 
  project where a device plugin is used to handle CPU and memory management. In contrast 
  to this work our CCI approach concentrates on CPU and memory and will not require to do initial reservation of system resources to handle user applications. 
  The realisation of CCI is clearly separated from other components and will enable a functional and seamless integration with other device plugins and DRA plugins. 
  In later phase of CCI development will consider handling CPU and memory resources also for pods
  not requiring drivers to run, by leveraging existing code base (post-alpha goal - 
  it can also include refactoring of existing code base). Further, CCI approach will cover required interfaces 
  to identify available CPU and memory resource so that correct scheduling can be performed. 
   
## Infrastructure Needed 
  
We may choose to add in a repo that allows donations of drivers specific to particular
use cases, similar to the repo currently available for Kubernetes scheduler Plugins.  This 
will allow a central place for the community to donate useful content.
