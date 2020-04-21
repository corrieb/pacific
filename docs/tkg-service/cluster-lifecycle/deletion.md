# Cluster Deletion

Before reading this section, it's worthwhile reviewing [The Basics](creation-basics.md) to establish a good grounding of the various controllers, objects and the relationship between them

## Summary

One of the real benefits to Guest Clusters is the ephemeral nature of them. Once you have a TanzuKubernetesCluster YAML that works for you, it's relatively trivial to stand one up, run something in it for a period of time and then tear it down again. 

Of course you don't have to do this, but a key design principle of Guest Clusters is that they should be cheap to create, cheap to maintain and cheap to destroy. 

Once you've run the gauntlet of [cluster creation](creation-milestones.md), cluster deletion sounds like it should be really straightforward - just blow away the cluster and all of its dependencies and start over. However, deletion and cleanup can be more complex than might initially be appreciated.

Understanding how cluster deletion works is invaluable if you find yourself in a situation where you have to troubleshoot it.

### How To Delete A Cluster

It's interesting to note that our user docs don't tell you how to delete a cluster. Maybe it seems too obvious. But it's worth addressing the ideal scenario and what you should expect to see.

At its most simple, deletion is just a reversal of creation. If we consider that creation is often as simple as:

`kubectl apply -f test-cluster.yaml`

Deletion can be as simple as:

`kubectl delete -f test-cluster.yaml` 
or 
`kubectl delete -n ben-test tkc test-cluster`

#### Delete is a Blocking Call - Give it Time!

Note that the default deletion protocol by kubectl is that it blocks until deletion is completed. Well, you might think, all I'm doing is deleting an object in an API Server (the TanzuKubernetesCluster) so who cares if it's synchronous or asynchronous - it should just complete immediately, right?

Well, actually no. The `kubectl delete` call will block until absolutely everything is gone and we designed it that way. That means that it might _appear_ that deletion has hung when in fact - it may well be busy going through the motions of tearing down the cluster.

#### Deleting a Namespace

There's nothing preventing you from deleting a namespace with Guest Clusters in it from the vSphere client. And it _should_ work. While it might seem to have the same effect - a blocking call from the vSphere client, the way it works under the covers is very different.

As we'll see below, cluster deletion is a complex dance of interdependencies, not unlike a domino run. All you have to do is topple the first domino and everything else should cascade from there. When you delete a namespace however, the Kubernetes namespace controller takes a tally of all the dominos in the run and attempts to topple them asynchronously in a non-deterministic order. The vSphere service that handles deletion then sits by polling for the namespace to have been removed.

So by all means delete a namespace with Guest Clusters in it, but by deleting the Guest Clusters before the namespace, you're taking a more deterministic and easier to triage path to successful cleanup.

#### Deletion Controls

Kubernetes Clusters are complex beasts and we've tried to put safeguards in place to help prevent inadventent problems due to accidental deletion.

As such, you cannot delete Guest Cluster nodes from the vSphere UI. You can't even power them off. All of that has to be done by manipulating the TanzuKuberenetesCluster via kubectl.

There is also a Webhook in place to prevent you from deleting or editing CAPI, CAPW, CAPBK or VirtualMachine objects in your supervisor namespace. All of those objects are read-only. The control point is the TanzuKuberenetesCluster only. This is why it's so important that deletion _works_.

### The Object Graph

This section describes the interdependencies between objects and how this impacts cluster deletion.

#### Owner References

I alluded above to a domino effect that happens when you delete a TKC. This is exactly what happens and is achieved through a Kubernetes feature called Owner References. An object can be _owned_ by another and if the owner is deleted, that deletion can cascade down to the owned object

You can see an owner reference by typing `kubectl describe <object>` and looking for a clause that looks like this:

```
  Owner References:
    API Version:           cluster.x-k8s.io/v1alpha2
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  MachineSet
    Name:                  test-cluster-workers-hphzf-54569b9cb4
    UID:                   f6bd0f3d-ba58-409c-9ffe-1061d531e0ae
```
This example here is a worker CAPI Machine object that has an owner. The owner is a MachineSet and the owner reference gives enough information for any caller to be able to identify the MachineSet object. If this MachineSet were to be deleted, the CAPI Machines it owns will also be deleted.

What this means in effect is that there is a graph of owner references that cascades down from TanzuKubernetesCluster all the way down to the VirtualMachines. The graph looks roughly like this:



#### Finalizers






