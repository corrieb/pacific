# Cluster Creation Milestones

Before reading this section, first review [The Basics](creation-basics.md) to establish a good grounding of the various controllers, objects and the relationship between them

## The Happy Path

Now that we understand all of the players and the stage they perform on, let's look at how we gauge successful cluster creation.

### Cluster Creation Milestones

When watching Cluster Creation, there are key milestones we want to look for. A failure of any one of these milestones will end up with an incomplete, misconfigured or broken cluster.

1. TanzuKubernetesCluster (TKC) has been validated and successfully deployed. Passing validation is the first milestone
2. TKG has reconciled the TKC and created CAPi/CAPW objects. You should see all of the CAPI / CAPW objects created at once. However, VirtualMachines are only created and initialized when specific conditions are met
3. The first control plane node VirtualMachine is created by CAPW. This shows that CAPW is working correctly
4. The first control plane node is powered on. This means that there was enough resource in the vSphere namespace to satisfy the request for the node
5. The first control plane node has an IP address. This means that networking is functioning correctly, assuming the node is reachable
6. The first control plane node has a running API server. This means that the control plane of the first node has been initialized. This is not the same as the control plane being ready, as there are post-initialization steps.
7. The various post-configuration steps are applied to the control plane successfully. This includes CNI provider, Pod Security Policies etc.
8. The rest of the VirtualMachines are created, VMs are powered on and get IP addresses. Again, this means that there is capacity and networking is working.
9. The rest of the Nodes join the cluster correctly. If a node does not join the cluster, it will eventually be deleted and the system will automatically attempt to create a new one.
10. You can log into the Cluster

Now that we've established the critical steps, let's look in more depth at each one. How can you look for success and what are common causes of failure.

### TKC Status

The TanzuKubernetesCluster has a comprehensive `Status` field that is your best gauge of the progress of cluster creation. Before we dive into the milestones, let's take a look at what the `Status` can tell us:

```
$ kubectl describe tkc test-cluster
Name:         test-cluster
Namespace:    ben-test
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"run.tanzu.vmware.com/v1alpha1","kind":"TanzuKubernetesCluster","metadata":{"annotations":{},"name":"test-cluster","namespac...
API Version:  run.tanzu.vmware.com/v1alpha1
Kind:         TanzuKubernetesCluster
Metadata:
  Creation Timestamp:  2020-04-17T22:57:08Z
  Finalizers:
    tanzukubernetescluster.run.tanzu.vmware.com
  Generation:        1
  Resource Version:  1056540
  Self Link:         /apis/run.tanzu.vmware.com/v1alpha1/namespaces/ben-test/tanzukubernetesclusters/test-cluster
  UID:               da0b4a25-2b3a-4239-ab4d-aa7efdc84dd6
Spec:
  Distribution:
    Full Version:  v1.17.4+vmware.1-tkg.1.0dba899
    Version:       v1.17.4
  Settings:
    Network:
      Cni:
        Name:  calico
      Pods:
        Cidr Blocks:
          192.0.2.0/16
      Service Domain:  tanzukubernetescluster.local
      Services:
        Cidr Blocks:
          198.51.100.0/12
  Topology:
    Control Plane:
      Class:          best-effort-small
      Count:          3
      Storage Class:  gc-storage-profile
    Workers:
      Class:          best-effort-small
      Count:          3
      Storage Class:  gc-storage-profile
Status:
  Addons:
    Authsvc:
      Name:    
      Status:  pending
    Cloudprovider:
      Name:    
      Status:  pending
    Cni:
      Name:    
      Status:  pending
    Csi:
      Name:    
      Status:  pending
    Dns:
      Name:    
      Status:  pending
    Proxy:
      Name:    
      Status:  pending
    Psp:
      Name:    
      Status:  pending
  Cluster API Status:
    Phase:  provisioning
  Node Status:
    test-cluster-control-plane-255dq:            pending
    test-cluster-control-plane-6mmvp:            pending
    test-cluster-control-plane-htfbl:            pending
    test-cluster-workers-qzvg2-cf9499b9d-swbk2:  pending
    test-cluster-workers-qzvg2-cf9499b9d-t8sgc:  pending
    test-cluster-workers-qzvg2-cf9499b9d-tfgxl:  pending
  Phase:                                         creating
  Vm Status:
    test-cluster-control-plane-255dq:            pending
    test-cluster-control-plane-6mmvp:            pending
    test-cluster-control-plane-htfbl:            pending
    test-cluster-workers-qzvg2-cf9499b9d-swbk2:  pending
    test-cluster-workers-qzvg2-cf9499b9d-t8sgc:  pending
    test-cluster-workers-qzvg2-cf9499b9d-tfgxl:  pending
Events:                                          <none>
```
This is what the status looks like when a TKC is first created. The most useful summary status field is the `Phase` which can also be seen if you just `kubectl get` the cluster

```
$ kubectl get tkc test-cluster

NAME           CONTROL PLANE   WORKER   DISTRIBUTION                     AGE     PHASE
test-cluster   3               3        v1.17.4+vmware.1-tkg.1.0dba899   2m21s   creating
```
The `VM Status` shows the progress of VirtualMachines as they go through their various lifecycle phases. Note that `pending` means that a VirtualMachine has not yet been created for the node.

The `Node Status` is the status from the perspective of Kubernetes. This will tell you whether the Guest Cluster control plane thinks that a node has joined successfully and is healthy.

The `ClusterAPI Status` is the status as reported by the CAPI controller to the Cluster object.

The `Addons` status is the individual status of all the post-configuration that gets applied to the control plane of the new cluster once it's available.

### 1. Validation of the TKC YAML

In order to apply the TKC YAML, you need to be authenticated as a user to the Supervisor cluster namespace. The user needs to be authorized to create new workloads into that namespace.

Eg: 
```
kubectl vsphere login --vsphere-username Administrator@vsphere.local --server=10.185.26.144 --insecure-skip-tls-verify=true
kubectl config use-context ben-test       ## the name of the supervisor namespace
kubectl apply -f test-cluster.yaml        ## the definition of the TKC
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

### 2. ClusterAPI Object Creation

The next phase in cluster lifecycle is the creation of the CAPI / CAPW objects by TKG.

If you see any entries in `Node Status` or `VM Status` in your TKC `Status`, ClusterAPI object creation succeeded. 

You can also type the following to see the CAPI Cluster objects in your namespace:

```
$ kubectl get clusters -A

NAME           PHASE
test-cluster   provisioned
```

The only conceivable reason for failure of this milestone is if the TKG controller is not running for some reason.

So how can we identify where these controllers are and check whether they're running?

```
$ kubectl get pods -n vmware-system-tkg 

NAME                                                   READY   STATUS    RESTARTS   AGE
vmware-system-tkg-controller-manager-986c97975-fjs96   2/2     Running   4          2d1h
vmware-system-tkg-controller-manager-986c97975-gc4td   2/2     Running   6          2d1h
vmware-system-tkg-controller-manager-986c97975-m68nj   2/2     Running   3          2d1h
```
Here you can see the TKG controller pods running. Unfortunately the question of which pod is currently elected leader is opaque to this summary. It's usually obvious from the logs, but you have to _get_ the logs to see.

Note that the restart counts are non-zero. This is not necessaily indicative of a problem. Controllers lean on the container restart logic under certain circumstances if pre-conditions aren't met for initialization. However if the restart count seems out of proportion to the rest of the system, this could be indicative of a persistent issue.

To view logs of a controller, type the following:

`$ kubectl logs -n vmware-system-tkg vmware-system-tkg-controller-manager-986c97975-fjs96 manager`

### 3. First Control Plane VirtualMachine is created

Cluster creation always begins by the creation of a single control plane node. Even if multiple nodes are selected, the initial control plane node will be the one to which other nodes are joined. This is distinguished in Kubeadm by the `InitConfiguration` and `JoinConfiguration` types.

You want to watch the `VM Status` of the TKG `Status`. You should see one of them go from `pending` to `created` (or most likely, `powered on`).

You can also type the following to see the VirtualMachine objects in your namespace:

```
$ kubectl get virtualmachines -A

NAME                               AGE
test-cluster-control-plane-wc8bt   5m3s
```
Just as with TKG, there are important controllers responsible for creating these VirtualMachine objects. You can find out more about these controllers and their roles [here](creation-basics.md)

```
$ kubectl get pods -n vmware-system-capw

NAME                                                              READY   STATUS    RESTARTS   AGE
vmware-system-capw-cabpk-controller-manager-78b4c4d679-8n94d      2/2     Running   4          2d2h
vmware-system-capw-cabpk-controller-manager-78b4c4d679-c5nt6      2/2     Running   2          2d2h
vmware-system-capw-cabpk-controller-manager-78b4c4d679-m7mbs      2/2     Running   3          2d2h
vmware-system-capw-capi-controller-manager-6c4b7b97c9-dkr8b       2/2     Running   2          2d2h
vmware-system-capw-capi-controller-manager-6c4b7b97c9-gzfpm       2/2     Running   5          2d2h
vmware-system-capw-capi-controller-manager-6c4b7b97c9-sq6fk       2/2     Running   2          2d2h
vmware-system-capw-v1alpha2-controller-manager-6ddb68b49b-7dn25   2/2     Running   4          2d2h
vmware-system-capw-v1alpha2-controller-manager-6ddb68b49b-jxkzv   2/2     Running   1          2d2h
vmware-system-capw-v1alpha2-controller-manager-6ddb68b49b-z5qwd   2/2     Running   7          2d2h
```
Here you can see 3 replicas each of the CAPI, CABPK and CAPW controllers. The ones marked `capw-v1alpha2-controller-manager` are the ones responsible for creating VirtualMachines.

### 4. First Control Plane VirtualMachine is powered on

Once the VirtualMachine object is created in the supervisor namespace, the VM operator controller should reconcile it, create a VM in vSphere with the supplied configuration and attempt to power it on.

Just as in milestone 3, the simplest way to verify that the control plane machine has powered on is by looking the TKC `Status`. You should see one `VM Status` showing `powered on`.

If after a period of time you don't see that happen, the two most likely error conditions are the following:

#### Lack of compute resource

The Guest Cluster nodes need to be allocated resource from the limits specified in the supervisor cluster namespace. If the VirtualMachineClass requested is Guaranteed, then the node VM will require a full reservation of CPU and memory. If that reservation cannot be satisfied, then the VM cannot be powered on. The VM operator controller will continue to try to power on the VM until resource becomes available.

TBD: Show how to identify this error

#### Issue with the VM Operator Controller

Just like TKG, CAPI, CABPK and CAPW there are 3 VM operator controllers that are responsible for reconciling VirtualMachine objects into vSphere operations. You can find out more about these controllers and their roles [here](creation-basics.md) 

```
$ kubectl get pods -n vmware-system-vmop

NAME                                                     READY   STATUS    RESTARTS   AGE
vmware-system-vmop-controller-manager-64cd7dd57f-ckw5j   2/2     Running   0          3h55m
vmware-system-vmop-controller-manager-64cd7dd57f-fvvk9   2/2     Running   0          3h55m
vmware-system-vmop-controller-manager-64cd7dd57f-xfn2d   2/2     Running   4          3h55m
```

As with the other controllers, if they're running and the restart count doesn't seem excessive, they should be working.

However note that the VM operator controllers need to be able to connect to and authenticate with the vCenter APIs. If there's a connectivity issue or an authentication issue, it would show up in the VM operator logs.

### 5. First Control Plane Node has an IP Address

Assuming the control plane node is booting, the next milestone is that it gets an IP address on the network. This will show as a VM being `ready` in the `VM Status` of the TKC object. 

Note that a `VM Status` can be `ready` even if the kubeadm bootstrap fails. This is why the `VM Status` is distinct from `Node Status`. The former is a vSphere perspective and the latter is a K8S perspective.

The control plane will have its own internal IP address and will be addressable externally via a load-balanced Virtual IP. While this isn't an in-depth look at networking, let's take a moment to examine how this is set up.

```
$ kubectl -n ben-test get services

NAMESPACE  NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
ben-test   test-cluster-control-plane-service   LoadBalancer   172.24.20.9      10.185.231.203   6443:31416/TCP   4h2m
```
This shows that there's an external load-balanced IP that maps to a supervisor namespace cluster-scoped service IP.

Let look at the endpoints for the service:

```
$ kubectl -n ben-test get endpoints
NAME                                 ENDPOINTS           AGE
test-cluster-control-plane-service   172.26.0.226:6443   4h7m
```
This shows the IP address of the node as vSphere sees it - the IP assigned to the VM NIC. Note that this IP address is allocated from the same CIDR as native pods or VMs deployed to the namespace. This is not an externally-accessible IP, hence why we recommend that if you need a shell into a node, you do it [via a jumpbox pod deployed to the same namespace](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-kubernetes/GUID-587E2181-199A-422A-ABBC-0A9456A70074.html).

The CAPI Cluster has an `API Endpoints` field that should show the external IP for the cluster and this is also visible in the  TKC `Status`.
