# `TanzuKubernetesCluster` - Rolling Update

## General Notes

### Update vs. Upgrade

For the sake of terminology, it's important that we distinguish between cluster
_update_ and cluster _upgrade_.

**Update**, or **Rolling Update**, is the general form that the controller
lifecycle is designed for - it includes not just changes to Kubernetes
versions, but also changes to the `VirtualMachine's` resources (e.g. disk size,
CPUs, memory, etc.), changes to `VirtualMachine's` configuration, changes to
the flags passed to Kubernetes components, and so on. Each of these changes
rolls out in the same fashion: the [**Rolling Update**](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/).

**Upgrade** is thus a specific form of update: one which _increments_ the
underlying versions - whether that be a Kubernetes version, the OS version, or
the add-on version, which we currently tie together into a single _distribution
version_. There are a handful of compatibility requirements that we need to be
aware of when changing versions, particularly around enforcing compatible
version skews with Kubernetes and Etcd (e.g. `kubelet` mustn't exceed the
`kube-apiserver` version), and those pre-checks are enforced during the rolling
update.

**Downgrade** is, accordingly, a specific form of update: one which
_decrements_ the underlying versions. As with upgrade, there are certain
requirements this imparts on the controller - for example, to maintain
Kubernetes skew compatibility, the workers (and thus `kubelets`) need to be
downgraded prior to the control plane (and thus `kube-apiserver`). This is
currently an unsupported operation, but will be handled with the same rolling
update strategy.

### Performing an Update

Most rolling updates will be initiated by one (or any combination of) of three
modifications to the `TanzuKubernetesCluster`:

* Changing the **distribution version** for the cluster
  (`.spec.distribution.version` or `.spec.distribution.fullVersion`).

```
### Note: when using the version hint we have to null out the fullVersion
### or there will be a mismatch between the discovered version and the
### existing fullVersion
$ read -r -d '' PATCH <<'EOF'
spec:
  distribution:
    fullVersion: null
    version: v1.16.7
EOF

### Note: the quotes around the variable are important to preserve newlines
$ kubectl patch --type=merge tanzukubernetescluster my-tkc --patch "$PATCH"
tanzukubernetescluster.run.tanzu.vmware.com/my-tkc patched
```

* Changing the `VirtualMachineClass` for the node pools
  (`.spec.topology.controlPlane.class` or `.spec.topology.workers.class`).

```
$ read -r -d '' PATCH <<'EOF'
spec:
  topology:
    controlPlane:
      class: best-effort-xlarge
    workers:
      class: best-effort-xlarge

### Note: the quotes around the variable are important to preserve newlines
$ kubectl patch --type=merge tanzukubernetescluster my-tkc --patch "$PATCH"
tanzukubernetescluster.run.tanzu.vmware.com/my-tkc patched
```

* Changing the `StorageClass` for the node pools
  (`.spec.topology.controlPlane.storageClass` or
  `.spec.topology.workers.storageClass`).

```
$ read -r -d '' PATCH <<'EOF'
spec:
  topology:
    controlPlane:
      storageClass: gc-storage-profile
    workers:
      storageClass: gc-storage-profile

### Note: the quotes around the variable are important to preserve newlines
$ kubectl patch --type=merge tanzukubernetescluster my-tkc --patch "$PATCH"
tanzukubernetescluster.run.tanzu.vmware.com/my-tkc patched
```

> **Note:** While these are the most common ways in which a rolling update will
> be initiated, they are not the only ones. Changes in any of the elements which
> contribute configuration can also initiate a rolling update.
>
> For example, renaming or replacing the `VirtualMachineImage` that corresponds
> to a **distribution version** will initiate a rolling update as the system
> attempts to get all nodes running on the new (or newly renamed) image.
>
> Additionally, rolling out an update to the
> `vmware-system-tkg-controller-manager` might introduce new pre-baked values
> into the manifest generator and the controller will initiate a rolling update
> in order to deploy those values.

## Rolling Updates

There are two distinct controllers responsible for performing a rolling update:
the **Add-ons Controller** and the **`TanzuKubernetesCluster` Controller**.
Within those two controllers there are three key stages to a rolling update:
updating add-ons, updating the control plane, and updating the workers. These
stages occur in order, with pre-checks that prevent a step from beginning until
the preceding step has completed.

> **Note:** In certain scenarios, some of the stages may not be required. For
> example, changing the `VirtualMachineClass` for the worker node pool
> (`.spec.topology.workers.class`) does not require that updates be rolled out
> to add-ons or the control plane, so that rolling update will only affect the
> worker nodes. Similarly, a change that only affects the control plane nodes
> will only be rolled out on the control plane nodes.
>
> The pre-checks are configured to pass only when there is no difference
> between the expected and desired state of the previous stage's work.

### The Add-ons Controller

[![add-ons reconciliation diagram](images/addon-reconciliation.png)](images/addon-reconciliation.png)

The add-ons controller manages the lifecycle of two classes of add-ons:

* **Core add-ons:** these add-ons interact with and provide functionality to
  the cluster itself, such as CoreDNS, kube-proxy, the Container Storage
  Interface (CSI), the Container Network Interface (CNI), and the Cloud
  Provider Interface (CPI). Some of these, (i.e. the CPI) are required for the
  lifecycle of the CAPI controllers.

* **Extension add-ons:** these are add-ons which provide value to the end-users
  of the workload cluster, but are not necessarily required for operations of
  the cluster. At the moment, this covers add-ons such as the Authentication
  Service.

During normal operation the add-ons controller will continuously reconcile the
add-on configuration in a workload cluster. For the most part, the
configuration of these add-ons are provided to the add-ons controller by the
way of annotations on the `VirtualMachineImage` (specifically, the image that
corresponds to the distribution version specified in the
`TanzuKubernetesCluster`).

For example, with a `TanzuKubernetesCluster` configured to use `v1.16.7`:
```
apiVersion: run.tanzu.vmware.com/v1alpha1
kind: TanzuKubernetesCluster
metadata:
  name: my-tkc
spec:
  topology:
    controlPlane:
      count: 3
      class: best-effort-xlarge
      storageClass: gc-storage-profile
    workers:
      count: 1
      class: best-effort-xlarge
      storageClass: gc-storage-profile
  distribution:
    fullVersion: v1.16.7+vmware.1-guest.1
    version: v1.16.7
  settings:
    network:
      cni:
        name: calico
      services:
        cidrBlocks:
        - 198.51.100.0/12
      pods:
        cidrBlocks:
        - 192.0.2.0/16
      serviceDomain: cluster.local
```

The corresponding `VirtualMachineImage` would look like this (with significant
redaction for readability):
```
apiVersion: vmoperator.vmware.com/v1alpha1
kind: VirtualMachineImage
metadata:
  annotations:
    vmware-system.guest.kubernetes.addons.authsvc: '{"version": "0.1-63-ge899b1f",
      "value": "...",
      "type": "inline"}'
    vmware-system.guest.kubernetes.addons.pvcsi: '{"name": "pvcsi",
      "value": "...",
      "type": "inline"}'
    vmware-system.guest.kubernetes.addons.vmware-guest-cluster: '{"name": "vmware-guest-cluster",
      "value": "...",
      "type": "inline"}'
    vmware-system.guest.kubernetes.distribution.image.version: '{
      "distribution":{"version":"v1.16.7+vmware.1-guest.1"},
      "kubernetes":{"imageRepository":"vmware","version":"v1.16.7+vmware.1"},
      "etcd":{"imageRepository":"vmware","version":"v3.3.15_vmware.6"},
      "coredns":{"imageRepository":"vmware","version":"v1.6.2_vmware.6"}}'
  name: photon-k8sd-v1.16.7-vmware.1-guest.1
spec:
  imageSourceType: Content Library
  osInfo:
    type: other3xLinux64Guest
  productInfo:
    fullVersion: v1.16.7+vmware.1-guest.1
    product: vSphere Kubernetes Service - Guest Cluster Image
    vendor: VMware Inc.
    version: v1.16.7+vmware.1-guest.1
  type: ovf
```

Each of the annotations prefixed with `vmware-system.guest.kubernetes.addons.`
include a JSON object containing a few elements: an optional `version`, a `name`,
a `type`, and, most importantly, a `value`. `value` contains a series of YAML
manifests that will ultimately be created or updated on the workload cluster.

For some core add-ons such as CoreDNS and kube-proxy (which are traditionally
configured by `kubeadm`), the container version will be taken from the
distribution annotation (i.e.
`vmware-system.guest.kubernetes.distribution.image.version`), which contains a
JSON object forming the basis of the **distribution version**.

#### Add-ons Update

Given that forwards compatibility with a particular Kubernetes version is not
enforceable, and that certain add-ons like CoreDNS or kube-proxy are developed
upstream, add-on updates are rollout first so that backwards compatible
versions can roll out in synchronization with the new nodes joining the
cluster.

When the distribution version is changed on a `TanzuKubernetesCluster`, the
add-ons controller will pull the new object manifests and create or update
those objects in the workload cluster. However, since each
`VirtualMachineImage` is pre-loaded with the particular container image for an
add-on and since there is (currently) no centralized registry that workload
clusters can pull those container images from, it's important to prevent the
add-ons' `Pods` from being spun down and replaced with `Pods` that will be
unable to find the necessary container image. To that end, built-in update
strategies are applied and configured on the applications.

##### `DaemonSets`

For `DaemonSets`, we configure the
[`.spec.updateStrategy.type`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#daemonsetupdatestrategy-v1-apps)
to `OnDelete`. This ensures that the existing `Pods` will not be spun down on
older nodes (thereby keeping the application available), but that new nodes
entering the cluster will receive the new configuration and spin up the
appropriate `Pods`. As the old nodes exit the cluster, so too will the `Pods`
containing the stale configuration.

##### `Deployments`

For `Deployments` we configure four things:

* [`.spec.strategy.type`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#deploymentstrategy-v1-apps)
  is configured to `RollingUpdate`
* [`.spec.strategy.rollingUpdate.maxUnavailable`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#rollingupdatedeployment-v1-apps)
  is set to `0`.
* [`.spec.strategy.rollingUpdate.maxSurge`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#rollingupdatedeployment-v1-apps)
  is set to `1`.
* [`.spec.template.spec.nodeSelector`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#podspec-v1-core)
  is set to target only `Nodes` where the label
  `run.tanzu.vmware.com/kubernetesDistributionVersion` contains the appropriate
  distribution version (i.e.
  `run.tanzu.vmware.com/kubernetesDistributionVersion=v1.16.7+vmware.1-guest.1`).
  This label is set at creation time for every node in the cluster and will not
  be mutated (short of an end-user `cluster-admin` overwriting the label).

The `nodeSelector` ensures that the new `Pods` only target the nodes which will
have the container image available, while the `maxUnavailable` and `maxSurge`
settings ensure that the old `Pods` will not be spun down until there is a
viable replacement.

##### Deleting Objects

Currently, the add-ons controller does not clean up unused add-on objects.
Hypothetically, should an add-on cease using a `ConfigMap` between versions,
that `ConfigMap` will remain in the workload cluster, no longer managed by
the add-ons controller. While it is unlikely to be harmful, these stale objects
should be able to be safely removed - and if they're removed erroneously,
they will be reinstated the next time the add-ons controller reconciles that
particular `TanzuKubernetesCluster`.

#### Add-on Statuses

The `TanzuKubernetesCluster` reports the status of each of the managed add-ons
in its status (`.status.addons`). Each add-on has only a few possible statuses:
`pending`, `applied`, `error`, and `unmanaged`.

* `pending`: the add-ons controller has not yet begun reconciling the add-on in
  the workload cluster, generally because the workload cluster is not yet
  available.
* `applied`: the add-ons controller has successfully reconciled the add-on
  manifest in the workload cluster.
* `error`: the add-ons controller has experienced an error attempting to
  reconcile the add-on in the workload cluster. The `lastErrorMessage` field
  will contain further information about the error. Often, these errors
  might be temporary (e.g. a networking issue when speaking to the workload
  cluster) and will be resolved when the cluster is next reconciled.
* `unmanaged`: the add-ons controller was unable to manage the add-on. Currently,
  this only applies to the Authentication Service, which in earlier clusters,
  was not manaed as an add-on, but explicitly applied during initialization
  of the workload cluster (as a `postKubeadm` command).

As an example (pruned to only the relevant status entries:

```
apiVersion: run.tanzu.vmware.com/v1alpha1
kind: TanzuKubernetesCluster
metadata:
  ...
spec:
  ...
status:
  addons:
    authsvc:
      name: authsvc
      status: applied
      version: 0.1-63-ge899b1f
    cloudprovider:
      name: vmware-guest-cluster
      status: applied
      version: v1.15.5+vmware.1-guest.1
    cni:
      name: calico
      status: applied
      version: v1.15.5+vmware.1-guest.1
    csi:
      name: pvcsi
      status: applied
      version: v1.15.5+vmware.1-guest.1
    dns:
      name: CoreDNS
      status: applied
      version: v1.3.1_vmware.4
    proxy:
      name: kube-proxy
      status: applied
      version: v1.15.5+vmware.1
    psp:
      name: defaultpsp
      status: applied
      version: v1.15.5+vmware.1-guest.1
  ...
```

##### Error States

The add-ons controller is fairly simple in structure and, assuming well-formed
`VirtualMachineImages`, does not have many catastrophic failure scenarios. Each
add-on is reconciled independently from the other, so an error reconciling one
add-on will not necessarily prevent the next from being reconciled. Most errors
will be transitory and will eventually be cleared. However, since the add-ons
controller is running in the management cluster and speaking to a separate
workload cluster, that does introduce the possibility for permanent network
reachability issues.

Additionally, should malformed `VirtualMachineImages` enter the content
library, either through end-users adding their own images, corrupted downloads,
or erroneous publications to the CDNs, it is possible that the controller will
be unable to unmarshal the YAML manifests from the annotations, leading to an
inability to operate on certain add-ons.

### The `TanzuKubernetesCluster` Controller

The `TanzuKubernetesCluster` controller manages the lifecycle of the cluster
infrastructure by translating the top-level `TanzuKubernetesCluster` object
into a series of CAPI, CAPW, and CABPK objects. More detailed information about
that can be found in the [creation basics documentation](creation-basics.md).

#### Update Detection

During object creation and sync, the controller leverages an internal manifest
generator that composes the configuration taken from the
`TanzuKubernetesCluster`, the **distribution version** (or, more specifically,
the annotation on the corresponding `VirtualMachineImage`), and pre-configured
defaults (which are hardwired into the manifest generator). After the initial
creation phase, the synchronization phase will regularly regenerate these
manifests and compare them against the existing objects. If it detects that
there is an actionable difference (i.e. a difference in the `spec` of an
object), it will initiate a rolling update of the corresponding nodes.


#### Control Plane Update

[![control plane reconciliation diagram](images/control-plane-reconciliation.png)](images/control-plane-reconciliation.png)

##### Control Plane Pre-check

Before initiating a control plane update, there are a series of pre-checks that
must be passed.

* **Add-ons are updated:** all of the managed add-ons must have had the
  appropriate versions (`TanzuKubernetesCluster.status.addons.<addon>.version`)
  applied to the workload cluster. This version should correspond to the
  version recorded in that add-on's `VirtualMachineImage` annotation, but if
  that is not specified (as in older versions), it will report the distribution
  version by default.
* **There is no other control plane update in progress:** the orchestration
  of the control plane update does not allow for multiple updates to be
  triggered simultaneously.
* **There is no failed control plane update:** if the control plane update
  fails, it is expected that this will be remediated by manual intervention
  from an end-user. Until that has been accomplished, we cannot begin another
  update, because it is unclear if the cluster is in healthy shape.

> **Note:** This pre-check will prevent concurrent upgrades of Kubernetes
> versions. That means there will potentially be a lag between achieving the
> desired state if the distribution version is changed while another upgrade
> is in progress.

##### Cluster API Upgrade Tool

[ClusterAPI Upgrade Tool](https://github.com/vmware/cluster-api-upgrade-tool)
is a standalone tool that orchestrates a rolling update for control plane nodes
provisioned using ClusterAPI version v1alpha2. At a high level, this tool
performs the same form of rolling update that the `MachineDeployment`
controller executes for workers, but with a few extra steps to manage `etcd`
and to preserve the cluster's certificate `Secrets` (which CABPK ties to the
initialization node's `KubeadmConfig`):

* Create a new bootstrap config (`KubeadmConfig`) and apply a patch that
  mutates it to the newly desired configuration.
* Create a new infrastructure reference (`WCPMachine`) and apply a patch that
  mutates it to the newly desired configuration.
* Create a new `Machine`, wire it to refer to the previously created references.
* Wait for the new `Machine` to be created and become healthy.
* Rewire the cluster certificate `Secrets` `.metadata.ownerReferences` so they're
  owned by the new `KubeadmConfig` and won't be garbage collected when the old
  node is removed.
* Remove the old node from `etcd`.
* Delete the old `Machine`.

During this process, the tool will leverage annotations referring to an
`upgrade-id` so that it can pick up where it left off in the event of an error.

When an update to the control plane is initiated, this tool will be deployed as
a [`Job`](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)
(and thus `Pod`) alongside the `vmware-system-tkg-controller-manager` in the
management cluster. The `Job` and `Pods` do not necessarily have a clear name
associated with them, but their target cluster is identifiable through labels
(e.g. `cluster.x-k8s.io/cluster-name=my-tkc`):

```
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    upgrade-id: "1280807"
    upgrade-version: v1.16.7+vmware.1
  creationTimestamp: "2020-04-28T07:05:09Z"
  labels:
    cluster.x-k8s.io/cluster-name: my-tkc
  name: update-1280807
  namespace: vmware-system-tkg
  ownerReferences:
  - apiVersion: run.tanzu.vmware.com/v1alpha1
    kind: TanzuKubernetesCluster
    name: my-tkc
    uid: 208025ff-0cc9-4c88-b22d-5a43b1c33831
  resourceVersion: "1280904"
  selfLink: /apis/batch/v1/namespaces/vmware-system-tkg/jobs/update-1280807
  uid: 29cf33c2-32f4-4ff7-9dab-2d05bb3ea622
spec:
  backoffLimit: 3
  completions: 1
  parallelism: 1
  selector:
    matchLabels:
      controller-uid: 29cf33c2-32f4-4ff7-9dab-2d05bb3ea622
  template:
    metadata:
      annotations:
        upgrade-id: "1280807"
        upgrade-version: v1.16.7+vmware.1
      creationTimestamp: null
      labels:
        cluster.x-k8s.io/cluster-name: my-tkc
        controller-uid: 29cf33c2-32f4-4ff7-9dab-2d05bb3ea622
        job-name: update-1280807
    spec:
      containers:
      - args:
        - --kubernetes-version=$(VERSION)
        - --upgrade-id=$(UPGRADE_ID)
        - --cluster-name=$(CLUSTER_NAME)
        - --cluster-namespace=default
        - --infrastructure-patches=[{"op":"replace","path":"/spec/imageName","value":"photon-k8sd-v1.16.7-vmware.1-guest.1"},{"op":"replace","path":"/spec/className","value":"best-effort-xlarge"},{"op":"replace","path":"/spec/storageClass","value":"gc-storage-profile"}]
        - --kubeadm-configmap-patches=[{"op":"replace","path":"/dns/imageRepository","value":"vmware"},{"op":"replace","path":"/dns/imageTag","value":"v1.6.2_vmware.6"},{"op":"replace","path":"/etcd/local/imageRepository","value":"vmware"},{"op":"replace","path":"/etcd/local/imageTag","value":"v3.3.15_vmware.6"},{"op":"replace","path":"/imageRepository","value":"vmware"},{"op":"replace","path":"/kubernetesVersion","value":"v1.16.7+vmware.1"}]
        - --bootstrap-patches=[{"op":"replace","path":"/spec/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-labels","value":"run.tanzu.vmware.com/kubernetesDistributionVersion=v1.16.7_vmware.1-guest.1"}]
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: CLUSTER_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels['cluster.x-k8s.io/cluster-name']
        - name: UPGRADE_ID
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations['upgrade-id']
        - name: VERSION
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations['upgrade-version']
        image: cluster-api-upgrade-tool:latest
        imagePullPolicy: IfNotPresent
        name: upgrade
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      hostNetwork: true
      nodeSelector:
        node-role.kubernetes.io/master: ""
      restartPolicy: Never
      schedulerName: default-scheduler
      securityContext:
        runAsGroup: 1000
        runAsUser: 1000
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: kubeadmNode
        operator: Equal
        value: master
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
status:
  active: 1
  failed: 2
  startTime: "2020-04-28T07:05:09Z"
```

This `Job` runs one `Pod` at a time, waiting for each one to complete or fail.
In the event the `Pod` fails, the `Job` will launch a new `Pod` that will
attempt the upgrade again. It will do this up to four times before giving up
and marking the `Job` as failed. If any attempt succeeds, it will mark the
`Job` as having completed successfully (through the use of
[`.status.conditions`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#jobcondition-v1-batch)).

##### Error Scenarios

###### Pod Failed

Many times the `Pods` may error out due to a simple timeout - for example,
waiting for the infrastructure to become available or waiting for the `Node` to
receive a proper label. While this _may_ point to a more serious underlying
issue, the upgrade tool _can_ recover. For example:

```
$ kubectl -n vmware-system-tkg get pod -l job-name
NAME                                                    READY   STATUS    RESTARTS   AGE
update-1280807-22pkv                                    0/1     Error     0          37m
update-1280807-8q8df                                    0/1     Completed 0          22m
update-1280807-d2njt                                    0/1     Error     0          37m
update-1280807-mkc99                                    0/1     Error     0          37m
```

Unless the `Job` reports an failed condition, it doesn't necessarily indicate
a failure. In this case, it reports that the `Job` completed successfully:

```
$ kubectl -n vmware-system-tkg get job update-1280807 -o yaml
...
status:
  failed: 3
  conditions:
  - lastProbeTime: "2020-04-28T07:43:04Z"
    lastTransitionTime: "2020-04-28T07:43:04Z"
    status: "True"
    type: Complete
  startTime: "2020-04-28T07:05:09Z"
  succeeded: 1
```

An error, on the other hand, would look as such:

```
$ kubectl -n vmware-system-tkg get job update-1280807 -o yaml
...
status:
  failed: 4
  conditions:
  - lastProbeTime: "2020-04-28T07:55:12Z"
    lastTransitionTime: "2020-04-28T07:55:12Z"
    status: "True"
    type: Failed
  startTime: "2020-04-28T07:35:78Z"
```

###### Job Failed

When the `Job` itself has failed, the lifecycle controller is halted for that
particular cluster and the user must take manual steps to remediate. The best
way to begin debugging is by checking with the logs for the `Pods` and
determining whether the error was something transient (e.g. network timeouts,
infra deployment timeouts, etc.) or whether it's something more serious.

In the event that the errors are due to timeouts, it may be possible to simply
delete the `Job`. That will let the lifecycle controller attempt to initiate
another update.

**Example:** _Timed out waiting for the condition_

```
time="2020-04-14T14:14:11Z" level=info msg="Deleting machine" machine=default/test-cluster-e2e-script-control-plane-6hxsm upgrade=18553
time="2020-04-14T14:14:11Z" level=info msg="Waiting for machine to be deleted" machine=default/test-cluster-e2e-script-control-plane-6hxsm upgrade=18553
time="2020-04-14T14:14:31Z" level=info msg="Machine has been deleted" machine=default/test-cluster-e2e-script-control-plane-6hxsm upgrade=18553
time="2020-04-14T14:14:31Z" level=info msg="Removing machine from kubeadm ConfigMap" machine=default/test-cluster-e2e-script-control-plane-6hxsm upgrade=18553
time="2020-04-14T14:30:00Z" level=error msg="error removing machine from kubeadm ConfigMap" error="error getting kubeadm configmap from target cluster: Get https://192.168.123.2:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config: read tcp 172.26.0.211:38380->192.168.123.2:6443: read: no route to host" machine=default/test-cluster-e2e-script-control-plane-6hxsm upgrade=18553
Error: timed out removing machine default/test-cluster-e2e-script-control-plane-6hxsm from kubeadm ConfigMap: timed out waiting for the condition
timed out waiting for the condition
timed out removing machine default/test-cluster-e2e-script-control-plane-6hxsm from kubeadm ConfigMap
github.com/vmware/cluster-api-upgrade-tool/pkg/upgrade.(*ControlPlaneUpgrader).removeMachine
        /workspace/pkg/upgrade/control_plane.go:756
github.com/vmware/cluster-api-upgrade-tool/pkg/upgrade.(*ControlPlaneUpgrader).updateMachine
        /workspace/pkg/upgrade/control_plane.go:537
github.com/vmware/cluster-api-upgrade-tool/pkg/upgrade.(*ControlPlaneUpgrader).updateMachines
        /workspace/pkg/upgrade/control_plane.go:838
github.com/vmware/cluster-api-upgrade-tool/pkg/upgrade.(*ControlPlaneUpgrader).Upgrade
        /workspace/pkg/upgrade/control_plane.go:280
main.main.func1
        /workspace/main.go:71
github.com/spf13/cobra.(*Command).execute
        /go/pkg/mod/github.com/spf13/cobra@v0.0.5/command.go:826
github.com/spf13/cobra.(*Command).ExecuteC
        /go/pkg/mod/github.com/spf13/cobra@v0.0.5/command.go:914
github.com/spf13/cobra.(*Command).Execute
        /go/pkg/mod/github.com/spf13/cobra@v0.0.5/command.go:864
main.main
        /workspace/main.go:147
runtime.main
        /usr/local/go/src/runtime/proc.go:200
runtime.goexit
        /usr/local/go/src/runtime/asm_amd64.s:1337
```

**Example:** _Missing Secrets_

In a healthy cluster, there will be several different `Secrets` residing
alongside the `TanzuKubernetesCluster`, which containing the various
certificates used by the Kubernetes components in the workload cluster.

```
$ kubectl get secrets -l cluster.x-k8s.io/cluster-name=my-tkc
NAME             TYPE     DATA   AGE
my-tkc-ca        Opaque   2      2m19s
my-tkc-etcd      Opaque   2      2m19s
my-tkc-proxy     Opaque   2      2m19s
my-tkc-sa        Opaque   2      2m19s
```

These `Secrets` are owned by the `KubeadmConfig` for the initial control plane
`Machine`:

```
apiVersion: v1
data:
  tls.crt: ...
  tls.key: ...
kind: Secret
metadata:
  creationTimestamp: "2020-04-28T17:07:14Z"
  labels:
    cluster.x-k8s.io/cluster-name: my-tkc
  name: my-tkc-ca
  namespace: default
  ownerReferences:
  - apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
    kind: KubeadmConfig
    name: my-tkc-control-plane-wzt4d
    uid: 36dbddc8-df79-4ce1-b721-3daa34bdf90f
  resourceVersion: "1321219"
  selfLink: /api/v1/namespaces/default/secrets/my-tkc-ca
  uid: 76b45824-028d-46c8-87dd-c6f5c1e5f732
type: Opaque
```

During control plane upgrade, there is a dance where the replacement `Machine`
is brought up, and once it is healthy, the `OwnerReference` is updated to
reference the new `Machine's` `KubeadmConfig`. There have previously been two
scenarios where those `Secrets` were lost during the upgrade process because
the owner reference was not updated at the correct time and the Kubernetes
Garbage Collector swept them up ([#154](https://github.com/vmware/cluster-api-upgrade-tool/issues/154), [#158](https://github.com/vmware/cluster-api-upgrade-tool/issues/158)).

When this occurs, the log will record an error:

```
time="2020-04-14T23:25:38Z" level=info msg="Updating infrastructure reference" api-version=infrastructure.cluster.vmware.com/v1alpha2 kind=WCPMachine machine=default/my-tkc-control-plane-jts8w name=my-tkc-control-plane-jts8w upgrade-id=477124
time="2020-04-14T23:25:38Z" level=info msg="Updating bootstrap reference" api-version=bootstrap.cluster.x-k8s.io/v1alpha2 kind=KubeadmConfig machine=default/my-tkc-control-plane-jts8w name=my-tkc-control-plane-jts8w upgrade-id=477124
Error: secrets "my-tkc-ca" not found
secrets "my-tkc-ca" not found
github.com/vmware/cluster-api-upgrade-tool/pkg/upgrade.(*ControlPlaneUpgrader).updateBootstrapConfig
        /workspace/pkg/upgrade/control_plane.go:951
github.com/vmware/cluster-api-upgrade-tool/pkg/upgrade.(*ControlPlaneUpgrader).updateMachines
        /workspace/pkg/upgrade/control_plane.go:833
github.com/vmware/cluster-api-upgrade-tool/pkg/upgrade.(*ControlPlaneUpgrader).Upgrade
        /workspace/pkg/upgrade/control_plane.go:280
main.main.func1
        /workspace/main.go:71
github.com/spf13/cobra.(*Command).execute
        /go/pkg/mod/github.com/spf13/cobra@v0.0.5/command.go:826
github.com/spf13/cobra.(*Command).ExecuteC
        /go/pkg/mod/github.com/spf13/cobra@v0.0.5/command.go:914
github.com/spf13/cobra.(*Command).Execute
        /go/pkg/mod/github.com/spf13/cobra@v0.0.5/command.go:864
main.main
        /workspace/main.go:147
runtime.main
        /usr/local/go/src/runtime/proc.go:200
runtime.goexit
        /usr/local/go/src/runtime/asm_amd64.s:1337
```

At this point, recovering the `Secrets` will require decoding existing
bootstrap data off the `Machine` and manually creating the `Secrets` with their
expected contents.

```
$ kubectl get machine my-tkc-2-control-plane-7hrkx -o jsonpath="{.spec.bootstrap.data}" | base64 --decode
## template: jinja
#cloud-config

write_files:
-   path: /etc/kubernetes/pki/ca.crt
    owner: root:root
    permissions: '0640'
    content: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----

-   path: /etc/kubernetes/pki/ca.key
    owner: root:root
    permissions: '0600'
    content: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

-   path: /etc/kubernetes/pki/etcd/ca.crt
    owner: root:root
    permissions: '0640'
    content: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----

-   path: /etc/kubernetes/pki/etcd/ca.key
    owner: root:root
    permissions: '0600'
    content: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

-   path: /etc/kubernetes/pki/front-proxy-ca.crt
    owner: root:root
    permissions: '0640'
    content: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----

-   path: /etc/kubernetes/pki/front-proxy-ca.key
    owner: root:root
    permissions: '0600'
    content: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

-   path: /etc/kubernetes/pki/sa.pub
    owner: root:root
    permissions: '0640'
    content: |
      -----BEGIN PUBLIC KEY-----
      ...
      -----END PUBLIC KEY-----

-   path: /etc/kubernetes/pki/sa.key
    owner: root:root
    permissions: '0600'
    content: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

...
```

Each `Secret` is then of the form:

```
apiVersion: v1
data:
  tls.crt: ${BASE-64 ENCODED CERT OR PUBLIC KEY DATA}
  tls.key: ${BASE-64 ENCODED PRIVATE KEY DATA}
kind: Secret
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: my-tkc
  name: my-tkc-${CERT TYPE}
  namespace: default
  ownerReferences:
  - apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
    kind: KubeadmConfig
    name: my-tkc-control-plane-wzt4d
    uid: 36dbddc8-df79-4ce1-b721-3daa34bdf90f
type: Opaque
```

#### Worker Update

[![worker reconciliation diagram](images/worker-reconciliation.png)](images/worker-reconciliation.png)

##### Worker Pre-check

Before initiating a worker update, there are a series of pre-checks that must
be passed.

* All `Machines` labeled as a control plane node (containing the label
  `cluster.x-k8s.io/control-plane`) must have a Kubernetes version
  (`Machine.spec.version`) greater than or equal to the desired version of the
  workers.
  * This prevents the workers from exceeding the [Kubelet skew
  policy](https://kubernetes.io/docs/setup/release/version-skew-policy/#kubelet)
  with the control plane components.
  * This also ensures that the CAPI Upgrade Tool has completed operation, since
  the final operation is to remove the old `Machine`.

> **Note:** This pre-check only prevents workers from being upgraded while the
> control plane is being upgraded. This does _not_ prevent workers from
> receiving an update while control plane update is occuring. This means that
> simple changes, such as a `VirtualMachineClass` update can roll out
> simultaneously.

#### `MachineDeployment` Update

When an update to the worker node pool is initiated, the
`TanzuKubernetesCluster` Controller is responsible only for performing a few
steps:

* Generate a new `WCPMachineTemplate` with the new configuration.
* Generate a new `KubeadmConfigTemplate` with the new configuration.
* Update the `MachineDeployment's` infrastructure reference
  (`.spec.infrastructureRef`) and bootstrap reference
  (`.spec.bootstrap.configRef`) to target the newly generated templates.

After which, the CAPI's [`MachineDeployment`
Controller](https://github.com/kubernetes-sigs/cluster-api/blob/v0.2.9/controllers/machinedeployment_controller.go)
takes over:

* Generate a new `MachineSet` with the new configuration taken from the
  templates.
* Scale up the new `MachineSet` by `MachineDeployments'` `maxSurge` setting
  (`.spec.strategy.rollingUpdate.maxSurge`) (`1`, by default).
* Scale in the old `MachineSet` once the new replacements are healthy.
* Repeat the scaling-up and scaling-in until all replicas are present in the
  new `MachineSet` and the old one contains `0` replicas.

##### Error Scenarios

Most of the error scenarios are not going to be related directly to the rolling
update process, but will be generic errors that we can see during the normal
creation and scale-out phase as well, such as `kubeadm` timeouts.

#### `TanzuKubernetesCluster` Phases

Once an update has been initiated, there may be a short period of time before
the cluster begins to reconcile the update. Once the add-ons update is complete
and the control plane or worker updates can begin. While the control plane
upgrade `Job` is actively running or while there are replicas spread across
multiple `MachineSets` for the workers, the `TanzuKubernetesCluster` will
report the phase `updating`.

Should the control plane upgrade fail, the phase will move to `updateFailed`.
There is not currently detection for the worker update failing, though there is
work upstream to surface those conditions.
