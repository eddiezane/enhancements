# Implement maxUnavailable in StatefulSet

## Table of Contents

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
  - [Implementation Details](#implementation-details)
    - [API Changes](#api-changes)
    - [Implementation](#implementation)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [Upgrades/Downgrades](#upgradesdowngrades)
  - [Tests](#tests)
- [Test Plan](#test-plan)
- [Graduation Criteria](#graduation-criteria)
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

The purpose of this enhancement is to implement maxUnavailable for StatefulSet during RollingUpdate. 
When a StatefulSet’s `.spec.updateStrategy.type` is set to `RollingUpdate`, the StatefulSet controller 
will delete and recreate each Pod in the StatefulSet. The updating of each Pod currently happens one at a time. With support for `maxUnavailable`, the updating will proceed `maxUnavailable` number of pods at a time. 

## Motivation

Consider the following scenarios:-

1. My containers publish metrics to a time series system. If I am using a Deployment, each rolling 
update creates a new pod name and hence the metrics published by this new pod starts a new time series 
which makes tracking metrics for the application difficult. While this could be mitigated, it requires 
some tricks on the time series collection side. It would be so much better, If we could use a 
StatefulSet object so my object names doesnt change and hence all metrics goes to a single time series. This will be easier if StatefulSet is at feature parity with Deployments.
2. My Container does some initial startup tasks like loading up cache or something that takes a lot of 
time. If we used StatefulSet, we can only go one pod at a time which would result in a slow rolling 
update. If StatefulSet supported maxUnavailable with value greater than 1, it would allow for a faster 
rollout since a total of maxUnavailable number of pods could be loading up the cache at the same time.
3. My Stateful clustered application, has followers and leaders, with followers being many more than 1. My application can tolerate many followers going down at the same time. I want to be able to do faster 
rollouts by bringing down 2 or more followers at the same time. This is only possible if StatefulSet 
supports maxUnavailable in Rolling Updates.
4. Sometimes I just want easier tracking of revisions of a rolling update. Deployment does it through 
ReplicaSets and has its own nuances. Understanding that requires diving into the complicacy of hashing 
and how ReplicaSets are named. Over and above that, there are some issues with hash collisions which 
further complicate the situation(I know they were solved). StatefulSet introduced ControllerRevisions 
in 1.7 which are much easier to think and reason about. They are used by DaemonSet and StatefulSet for 
tracking revisions. It would be so much nicer if all the use cases of Deployments can be met in 
StatefulSet's and additionally we could track the revisions by ControllerRevisions. Another way of 
saying this is, all my Deployment use cases are easily met by StatefulSet, and additionally I can enjoy
easier revision tracking only if StatefulSet supported `maxUnavailable`.

With this feature in place, when using StatefulSet with maxUnavailable >1, the user is making a 
conscious choice that more than one pod going down at the same time during rolling update, would not 
cause issues with their Stateful Application which have per pod state and identity. Other Stateful 
Applications which cannot tolerate more than one pod going down, will resort to the current behavior of one pod at a time Rolling Updates.

### Goals
StatefulSet RollingUpdate strategy will contain an additional parameter called `maxUnavailable` to 
control how many Pods will be brought down at a time, during Rolling Update.

### Non-Goals
NA

## Proposal

### User Stories

#### Story 1
As a User of Kubernetes, I should be able to update my StatefulSet, more than one Pod at a time, in a 
RollingUpdate manner, if my Stateful app can tolerate more than one pod being down, thus allowing my 
update to finish much faster. 

### Implementation Details

#### API Changes

Following changes will be made to the Rolling Update Strategy for StatefulSet.

```go
// RollingUpdateStatefulSetStrategy is used to communicate parameter for RollingUpdateStatefulSetStrategyType.
type RollingUpdateStatefulSetStrategy struct {
	// THIS IS AN EXISTING FIELD
        // Partition indicates the ordinal at which the StatefulSet should be
        // partitioned.
        // Default value is 0.
        // +optional
        Partition *int32 `json:"partition,omitempty" protobuf:"varint,1,opt,name=partition"`

	// NOTE THIS IS THE NEW FIELD BEING PROPOSED
	// The maximum number of pods that can be unavailable during the update.
        // Value can be an absolute number (ex: 5) or a percentage of desired pods (ex: 10%).
        // Absolute number is calculated from percentage by rounding down.
        // Defaults to 1.
        // +optional
        MaxUnavailable *intstr.IntOrString `json:"maxUnavailable,omitempty" protobuf:"bytes,2,opt,name=maxUnavailable"`
	
	...
}
```

- By Default, if maxUnavailable is not specified, its value will be assumed to be 1 and StatefulSets 
will follow their old behavior. This will also help while upgrading from a release which doesnt support maxUnavailable to a release which supports this field.
- If maxUnavailable is specified, it cannot be greater than total number of replicas.
- If maxUnavailable is specified and partition is also specified, MaxUnavailable cannot be greater than `replicas-partition`
- If a partition is specified, maxUnavailable will only apply to all the pods which are staged by the 
partition. Which means all Pods with an ordinal that is greater than or equal to the partition will be 
updated when the StatefulSet’s .spec.template is updated. Lets say total replicas is 5 and partition is set to 2 and maxUnavailable is set to 2. If the image is changed in this scenario, following
  are the possible behavior choices we have:-

  1. Pods with ordinal 4 and 3 will start Terminating at the same time(because of maxUnavailable). Once they are both running and ready, pods with ordinal 2 will start Terminating. Pods with ordinal 0 and 1 
will remain untouched due the partition. In this choice, the number of pods terminating is not always 
maxUnavailable, but sometimes less than that. For e.g. if pod with ordinal 3 is running and ready but 4 is not, we still wait for 4 to be running and ready before moving on to 2. This implementation avoids 
out of order Terminations of pods.
  2. Pods with ordinal 4 and 3 will start Terminating at the same time(because of maxUnavailable). When any of 4 or 3 are running and ready, pods with ordinal 2 will start Terminating. This could violate 
ordering guarantees, since if 3 is running and ready, then both 4 and 2 are terminating at the same 
time out of order. If 4 is running and ready, then both 3 and 2 are Terminating at the same time and no ordering guarantees are violated. This implementation, guarantees, that always there are maxUnavailable number of Pods Terminating except the last batch.
  3. Pod with ordinal 4 and 3 will start Terminating at the same time(because of maxUnavailable). When 4 is running and ready, 2 will start Terminating. At this time both 2 and 3 are terminating. If 3 is 
running and ready before 4, 2 wont start Terminating to preserve ordering semantics. So at this time, 
only 1 is unavailable although we requested 2.
  4. Introduce a field in Rolling Update, which decides whether we want maxUnavailable with ordering or without ordering guarantees. Depending on what the user wants, this Choice can either choose behavior 1 or 3 if ordering guarantees are needed or choose behavior 2 if they dont care. To simplify this further
PodManagementPolicy today supports `OrderedReady` or `Parallel`. The `Parallel` mode only supports scale up and tear down of StatefulSets and currently doesnt apply to Rolling Updates. So instead of coming up
with a new field, we could use the PodManagementPolicy to choose the behavior the User wants.

        1. PMP=Parallel will now apply to RollingUpdate. This will choose behavior described in 2 above.
           This means always maxUnavailable number of Pods are terminating at the same time except in 
           the last case and no ordering guarantees are provided.
        2. PMP=OrderedReady with maxUnavailable can choose one of behavior 1 or 3. 

NOTE: The goal is faster updates of an application. In some cases , people would need both ordering 
and faster updates. In other cases they just need faster updates and they dont care about ordering as 
long as they get identity.

Choice 1 is simpler to reason about. It does not always have maxUnavailable number of Pods in 
Terminating state. It does not guarantee ordering within the batch of maxUnavailable Pods. The maximum 
difference in the ordinals which are Terminating out of Order, cannot be more than maxUnavailable.

Choice 2 always offers maxUnavailable number of Pods in Terminating state. This can sometime lead to 
pods terminating out of order. This will always lead to the fastest rollouts. The maximum difference in the ordinals which are Terminating out of Order, can be more than maxUnavailable.

Choice 3 always guarantees than no two pods are ever Terminating out of order. It sometimes does that, 
at the cost of not being able to Terminate maxUnavailable pods. The implementationg for this might be 
complicated.

Choice 4 provides a choice to the users and hence takes the guessing out of the picture on what they 
will expect. Implementing Choice 4 using PMP would be the easiest.

#### Implementation

The alpha release we are going with Choice 4 with support for both PMP=Parallel and PMP=OrderedReady.
For PMP=Parallel, we will use Choice 2
For PMP=OrderedReady, we will use Choice 3 to ensure we can support ordering guarantees while also 
making sure the rolling updates are fast.


https://github.com/kubernetes/kubernetes/blob/v1.13.0/pkg/controller/statefulset/stateful_set_control.go#L504
```go
...
	 // we compute the minimum ordinal of the target sequence for a destructive update based on the strategy.
        updateMin := 0
	maxUnavailable := 1
        if set.Spec.UpdateStrategy.RollingUpdate != nil {
                updateMin = int(*set.Spec.UpdateStrategy.RollingUpdate.Partition)

		// NEW CODE HERE
		maxUnavailable, err = intstrutil.GetValueFromIntOrPercent(intstrutil.ValueOrDefault(set.Spec.UpdateStrategy.RollingUpdate.MaxUnavailable, intstrutil.FromInt(1)), int(replicaCount), false)
		if err != nil {
			return &status, err
		}
	}

	var unavailablePods []string
	// we terminate the Pod with the largest ordinal that does not match the update revision.
	for target := len(replicas) - 1; target >= updateMin; target-- {

		// delete the Pod if it is not already terminating and does not match the update revision.
		if getPodRevision(replicas[target]) != updateRevision.Name && !isTerminating(replicas[target]) {
			klog.V(2).Infof("StatefulSet %s/%s terminating Pod %s for update",
				set.Namespace,
				set.Name,
				replicas[target].Name)
			if err := ssc.podControl.DeleteStatefulPod(set, replicas[target]); err != nil {
				return &status, err
			}

			// After deleting a Pod, dont Return From Here Yet.
			// We might have maxUnavailable greater than 1
			status.CurrentReplicas--
		}

		// wait for unhealthy Pods on update
		if !isHealthy(replicas[target]) {
			// If this Pod is unhealthy regardless of revision, count it in 
			// unavailable pods
			unavailablePods = append(unavailablePods, replicas[target].Name)
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to update",
				set.Namespace,
				set.Name,
				replicas[target].Name)
		}
		
		// NEW CODE HERE
		// If at anytime, total number of unavailable Pods exceeds maxUnavailable,
		// we stop deleting more Pods for Update
		if len(unavailablePods) >= maxUnavailable {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for unavailable Pods %v to update, max Allowed to Update Simultaneously %v",
				set.Namespace,
				set.Name,
				unavailablePods,
				maxUnavilable)
			return &status, nil
		}

	}
...
```

### Risks and Mitigations
We are proposing a new field called `maxUnavailable` whose default value will be 1. In this mode, StatefulSet will behave exactly like its current behavior.
Its possible we introduce a bug in the implementation. The mitigation currently is that is disabled by default in Alpha phase for people to try out and give
feedback. 
In Beta phase when its enabled by default, people will only see issues or bugs when `maxUnavailable` is set to something greater than 1. Since people have
tried this feature in Alpha, we would have time to fix issues.


### Upgrades/Downgrades

We will default to 1 for maxUnavailable field in StatefulSet for backward compatibility

Downgrades

When downgrading from a release with this feature, to a release without maxUnavailable, there are two cases
 - If maxUnavailable is greater than 1, there are two more cases:-
   - If you're rolling back to a release that doesn't have this field - then there is even no way to discover it
   - If you're just disabling the feature (either together with downgrade to a release that has a field or without downgrade),the field should remain set 
        (unless someone will explicitly delete it later), but controller should ignore its behavior (and there shouldn't be a way to set it if the feature gate
         is switched off).
 - If maxUnavailable is less than equal to 1 -- in this case user wont see any difference in behavior

### Tests

- maxUnavailable =1, Same behavior as today with PodManagementPolicy as `OrderedReady` or `Parallel`
- Each of these Tests can be run in PodManagementPolicy = `OrderedReady` or `Parallel` and the Update
  should happen at most maxUnavailable Pods at a time in ordered or parallel fashion respectively.
- maxUnavailable greater than 1 without partition
- maxUnavailable greater than replicas without partition
- maxUnavailable greater than 1 with partition and staged pods less then maxUnavailable
- maxUnavailable greater than 1 with partition and staged pods same as maxUnavailable
- maxUnavailable greater than 1 with partition and staged pods greater than maxUnavailable
- maxUnavailable greater than 1 with partition and maxUnavailable greater than replicas

## Test Plan
For `Alpha`, unit tests and e2e tests will be added to test functionality at both
with feature flag enabled and disabled. Defaults will be verified so that users
who donot set this flag are not surprised at all.


## Graduation Criteria

- Alpha: Initial support for maxUnavailable in StatefulSets added. Disabled by default with default value of 1.
- Beta:  Enabled by default with default value of 1 with upgrade downgrade testedd at least manually.


## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: MaxUnavailableStatefulSet
  - Components depending on the feature gate: kube-apiserver and kube-controller-manager

###### Does enabling the feature change any default behavior?

No, the default behavior remains the same.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes this feature can be disabled. Once disabled, all existing  StatefulSet will
revert to the old behavior where rolling update will proceed one pod at a time.

###### What happens if we reenable the feature if it was previously rolled back?

We will restore the desired behavior for StatefulSets for which the maxunavailable field wasn't deleted after 
the feature gate was disabled.

###### Are there any tests for feature enablement/disablement?
yes, there are unit tests which make sure the field is correctly dropped 
on feature enable and disabled

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

A rollout or rollback of this feature can fail if there is a bug which causes the kube-apiserver or 
the kube-controller-manager to start crashing when the feature flag is enabled.


Yes, it can impact already running workloads.

If a rolling update is in progress for a StatefulSet, while this feature is being enabled in kube-apiserver
and kube-controller-manager, the StatefulSet controller can run into corner cases where it will take longer
for the controller to converge. This will only happen if after enabling the feature, the customer also sets
maxUnavailable to a number greater than 1,  but the invariants and the logic will ensure that there are never more than
maxUnavailable pods with the same identity and never more than maxUnavailable being deleted.

###### What specific metrics should inform a rollback?
TODO when we reach Beta

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?
Will be tested when graduating to Beta.


###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?
No

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?
If their StatefulSet rollingUpdate section has the field maxUnavailable specified with
a value different than 1.
The below command should show maxUnavailable value:
```
kubectl get statefulsets -o yaml | grep maxUnavailable
```

###### How can someone using this feature know that it is working for their instance?
TODO when we reach Beta

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

### Dependencies

###### Does this feature depend on any specific services running in the cluster?
NA

### Scalability

###### Will enabling / using this feature result in any new API calls?
It doesnt make any extra API calls.

###### Will enabling / using this feature result in introducing new API types?
No

###### Will enabling / using this feature result in any new calls to the cloud provider?
No


###### Will enabling / using this feature result in increasing size or count of the existing API objects?
A struct gets added to every StatefulSet object which has three fields, one 32 bit integer and two fields of type string.
The struct in question is IntOrString.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?
No

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?
The controller-manager will see very negligible and almost un-notoceable increase in cpu usage.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?
The RollingUpdate will fail or will not be able to proceed if etcd or apiserver is unavailable and
hence this feature will also be not be able to be used.

###### What are other known failure modes?
NA

## Implementation History

- KEP Started on 1/1/2019
- Implementation PR and UT by 8/30

## Drawbacks

NA

## Alternatives

- Users who need StatefulSets stable identity and are ok with getting a slow rolling update will continue to use StatefulSets. Users who
are not ok with a slow rolling update, will continue to use Deployments with workarounds for the scenarios mentioned in the Motivations
section.
- Another alternative would be to use OnDelete and deploy your own Custom Controller on top of StatefulSet Pods. There you can implement 
your own logic for deleting more than one pods in a specific order. This requires more work on the user but give them ultimate flexibility.

