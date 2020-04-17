# Cluster Creation Milestones

Before reading this section, first review [The Basics](creation-basics.md) to establish a good grounding of the various controllers, objects and the relationship between them

## The Happy Path

Now that we understand all of the players and the stage they perform on, let's look at how we gauge successful cluster creation.

### Cluster Creation Milestones

When watching Cluster Creation, there are key milestones we want to look for. A failure of any one of these milestones will end up with an incomplete, misconfigured or broken cluster.

1. TanzuKubernetesCluster (TKC) has been validated and successfully deployed. Passing validation is the first milestone
2. GCM has reconciled the TKC and created CAPi/CAPW objects. You should see all of the CAPI / CAPW objects created at once. However, VirtualMachines are only created and initialized when specific conditions are met
3. The first control plane node VirtualMachine is created by CAPW. This shows that CAPW is working correctly
4. The first control plane node is powered on. This means that there was enough resource in the vSphere namespace to satisfy the request for the node
5. The first control plane node has an IP address. This means that networking is functioning correctly, assuming the node is reachable
6. The first control plane node has a running API server. This means that the control plane of the first node has been initialized. This is not the same as the control plane being ready, as there are post-initialization steps.
7. The various post-configuration steps are applied to the control plane successfully. This includes CNI provider, Pod Security Policies etc.
8. The rest of the VirtualMachines are created, VMs are powered on and get IP addresses. Again, this means that there is capacity and networking is working.
9. The rest of the Nodes join the cluster correctly. If a node does not join the cluster, it will eventually be deleted and the system will automatically attempt to create a new one.
10. You can log into the Cluster

Now that we've established the critical steps, let's look in more depth at each one. How can you look for success and what are common causes of failure.

### Validation of the TKC YAML

In order to apply the TKC YAML, you need to be authenticated as a user to the Supervisor cluster namespace. The user needs to be authorized to create new workloads into that namespace.

Eg: 
```
kubectl vsphere login --vsphere-username Administrator@vsphere.local --server=10.185.26.144 --insecure-skip-tls-verify=true
kubectl config use-context ben-test
```

The TanzuKubernetesCluster YAML is the user-defined entrypoint into Cluster Creation. You can see an example in our user documentation [here](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-kubernetes/GUID-360B0288-1D24-4698-A9A0-5C5217C0BCCF.html) and a description of the valid configuration parameters [here](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-kubernetes/GUID-4E68C7F2-C948-489A-A909-C7A1F3DC545F.html).

```
apiVersion: run.tanzu.vmware.com/v1alpha1
kind: TanzuKubernetesCluster
metadata:
  name: test-cluster
  namespace: ben-test
spec:
  topology:
    controlPlane:
      count: 3
      class: best-effort-small 
      storageClass: gc-storage-profile
    workers:
      count: 3
      class: best-effort-small
      storageClass: gc-storage-profile
  distribution:
    version: v1.17.4
  settings:
    network:
      cni:
        name: calico
      services:
        cidrBlocks: ["198.51.100.0/12"]
      pods:
        cidrBlocks: ["192.0.2.0/16"]
      serviceDomain: "tanzukubernetescluster.local"
```

So what might be some common reasons for the TKC to fail validation, other than simple typos?

#### Kubernetes Version

The TKC has to specify a Kubernetes version that is actually available. You can query for the available versions by typing `kubectl get VirtualMachineImages` using a kubernetes client authenticated with the Supervisor cluster. You don't have to paste the full version - `v1.17.4` as an example should work fine.

If you're a vSphere Admin, you can also see the available versions by following the Content Library link on the Summary page of the workload namespace. You should see the available OVA templates listed.\

_Failure looks like this:_
```
admission webhook "version.mutating.tanzukubernetescluster.run.tanzu.vmware.com" denied the request: full version "v1.17.4+vmware.1-tkg.1.0dba899" does not match version hint "v1.17.5"
```

#### Node Configuration

The only currently valid values for `Control Plane count` are 1 and 3.

_Failure looks like this:_
```
admission webhook "default.validating.tanzukubernetescluster.run.tanzu.vmware.com" denied the request: control plane can only be either 1 or 3 nodes
```

The node `class` needs to be a VirtualMachineClass that's available to the namespace. Find out which are available by typing `kubectl get VirtualMachineClasses`.

_Failure looks like this:_
```
admission webhook "default.validating.tanzukubernetescluster.run.tanzu.vmware.com" denied the request: updates to immutable fields are not allowed, control plane class may not be changed
```

The node `storageClass` needs to be a StorageClass that's available to the namespace. These are made available to namespaces via the vSphere UI Workload Management section. Use `kubectl get StorageClasses` to see a valid list.

_Failure looks like this:_
```
admission webhook "default.validating.tanzukubernetescluster.run.tanzu.vmware.com" denied the request: updates to immutable fields are not allowed, storage class is not valid for control plane VM: StorageClass 'gc-storage-womble' is not assigned for namespace 'ben-test'
```
#### Network Settings

Network settings in the common case will look just like the ones above. As it stands, `calico` is the only valid CNI option. There are important caveats around choosing CIDR ranges, which are explained in the user documentation [here](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-kubernetes/GUID-4E68C7F2-C948-489A-A909-C7A1F3DC545F.html)

Note the `serviceDomain` will be the postfix of the DNS domain name of any services created in the cluster



