Topics
===

* Kubernetes Volume, Persistent Volume and Storage Class
* GlusterFS
* Container native storage (CNS) with GlusterFS


Kubernetes Volume, Persistent Volume and Storage Class
---


## Volume

**_Why introduce Volume_**

Volume address a few problems

* Containers in a pod unable to share data
* User want to run stateful application in kubernetes but containers are ephemeral

_http://kubernetes.io/docs/user-guide/volumes/_

At its core, a volume is just a directory, possibly with some data pre-populated in it, which is accessible to all the containers in a pod. How that directory comes to be, the medium that backs it, its contents, what happens to it after a pod is terminated, are all determined by the particular volume type used.


### Volume Plugins

**Remote Storage (persistent network attached storage)**
* GCE Persistent Disk
* AWS Elastic Block Store
* Azure File Storage
* Azure Data Disk
* iSCSI
* NFS
* GlusterFS
* Ceph File and RBD
* Cinder
* ...

These are volumes that outlive the lifecycle of a pod

* The lifecycle of these volumes is independent from the lifecycle of an individual pod
* When a pod referencing one of these volumes is created, the storage is automatically made * available inside the containers of that pod (the volumes are automatically attached and mounted)
* When the pod is terminated, the volume is automatically unmounted and detached, if needed, but the data on the volume remains intact


**Ephemeral Storage**
* Empty dir (and tmpfs)
* Expose Kubernetes API
  * Secret
  * ConfigMap
  * DownwardAPI

These volumes are created when a pod starts, and is deleted when a pod is terminated

**_EmptyDir is shared scratch space between pods_**

When the pods starts, k8s creates an empty directory on the underlying node machine (either from the local disk or from an in-memory temp-fs) and mounts the directory into all the containers in the pod.

When the pod is terminated, the directory and its content is deleted.

**_Secret, ConfigMap, and DownwardAPI are concepts in the Kubernetes API_**

They’re mechanisms for sharing configuration and cluster information with applications.

Secret, ConfigMap, and DownwardAPI volumes are a mechanism for exposing these k8s API objects as files inside the containers of a pod. These volumes are essentially an emptyDir under the covers but are pre-populated with contents from the Kubernetes API server on creation instead of being empty.

**Other**
* Host path



### Volume in action

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sleeper
spec:
  volumes:
    - name: data
      gcePersistentDisk:
        pdName: test-disk
        fsType: ext4
  containers:
    - name: busybox
      image: gcr.io/google_containers/busybox
      command:
        - sleep
        - "6000"
      volumeMounts:
        - name: data
          mountPath: /data
          readOnly: false
```

* A pod directly references a GCE PD volume and use by all the containers in it.
* This volume is assumed to **already exist**

**What happened behind the scene**

When the pod is created, Kubernetes automatically calls out to the google cloud API and attaches the specified PD to the node that the pod is scheduled do. It then formats the disk if needed, and mounts it into the containers that make up the pod.

When the pod is moved to another node, K8s automatically detach the PD from the first node, and attach it to the 2nd node and repeats the steps to mount the volume into the pod.

## Persistent Volumes and Claims

**_Why introduce Persistent Volume?_**

TLDR: Storage Abstraction

User does not want to specify the volume type in the pod definition for various reason:

* write once, run anywhere
* avoid coupling application to infrastructure
* avoid vendor lock-in

PV/PVCs enable an abstraction between the consumption of storage and the implementation of storage. The PV object is a representation of a piece storage available on the cluster
And a PVC is a request for storage on behalf of an application trying to run on the cluster.

### PV

* a object represent a storage device in the cluster
* contains description and connection information of the storage it exposes
* not used directly in the pods
* lifecycle independent of any individual pod


```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name : test-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 200Gi
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    fsType: ext4
    pdName: test-disk
```

```sh
$ kubectl get pv
NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM      STORAGECLASS     REASON    AGE
test-pv          200Gi      RWO           Retain          Available                                         5m
```

### PVC

* request for storage by a user
* claims request specific size and access modes of storage
* used by the pod as storage device

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

```sh
$ kubectl get pv
NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM              STORAGECLASS     REASON    AGE
test-pv          200Gi      RWO           Retain          Bound       default/test-pvc                              5m
```
In PVC, you define the **minimum requirements** for the storage you need: the min disk capacity and required access modes.

Now we can revisit the pod definition and use the PVC to make the pod portable.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sleeper
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-pvc
  containers:
    - name: busybox
      image: gcr.io/google_containers/busybox
      command:
        - sleep
        - "6000"
      volumeMounts:
        - name: data
          mountPath: /data
          readOnly: false
```

## Dynamic Provisioning and Storage Classes

PV/PVC provides storage abstraction for developers so they don't have couple their app with the underlying storage infrastructure. However, the cluster admin still have to pre-provision all the storage devices and make them available to the cluster, in the form of PVs, manually.

Storage class is introduced to provide dynamic volume provisioning.

* Cluster/Storage admins “enable” dynamic provisioning by creating StorageClass objects
* StorageClass objects define the parameters used during creation.
* StorageClass parameters are opaque to Kubernetes so storage providers can expose any number of custom parameters for the cluster admin to use.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
parameters:
  type: pd-standard
provisioner: kubernetes.io/gce-pd
```

* The StorageClass object specifies the volume plugin to use for provisioning, and the parameters to pass to that plugin.
* The parameters for the storage class are a blob of key-value pairs, that the provisioner presumably understands
* The parameters are defined by the volume plugin and set by cluster admins, and are opaque to k8s otherwise.
* the parameters list is a blob that is passed through to the the provisioner, kubernetes doesn’t have to get into the business of enumerating every possible parameter for every storage system in its API.

Now when a new pvc is created with the storage class like so:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard
```

kubernetes automatically creates:

* a GCE PD
* a pv that represent the new GCE PD
* bound the user created pvc with the pv

```sh
$ kubectl get pv
NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                   STORAGECLASS     REASON    AGE
pvc-301590e7-c9ff-11e7-a205-42010a8001c9   2Gi        RWO           Delete          Bound     default/test-pvc1       standard                   13s
```

```sh
kubectl get pvc
NAME            STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS     AGE
test-pvc1       Bound     pvc-301590e7-c9ff-11e7-a205-42010a8001c9   2Gi        RWO           standard         1m
```

* dynamically provisioned volumes are created with a “delete” retention policy
* change the Retention Policy to “retain” instead of “delete" if you want to keep the dynamically provisioned volume

### Default Storage Classes

* Allow dynamic provisioning even when a StorageClass is not specified in PVC
* Pre-installed Default Storage Classes in 1.6:
  * Google Cloud (GCE/GKE): Standard GCE PD

```sh
kubectl get sc
NAME                 TYPE
standard (default)   kubernetes.io/gce-pd
```

Disable the default storage class

```sh
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

GlusterFS
---

![gluster_logo](https://github.com/neoseele/gluster-demo/raw/master/resources/glusterfs-square-logo.png)

GlusterFS is a scalable network filesystem. Its nice but why do we care?

According to this:
[Volume Plugin Access Modes Matrix](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes), only a hand-full of Volume Plugins supports **ReadWriteMany**.

* AzureFile
* CephFS
* Glusterfs
* NFS
* Quobyte
* PortworxVolume

When a user have a workload that requires X amount of pods writing to a single shared PV concurrently, their options are limited.

GlusterFS stands out as it has
* strong support from both community ([gluster.org](http://www.gluster.org/)) and vendor ([RHS](https://access.redhat.com/documentation/en-us/red_hat_gluster_storage/3.1/html/container-native_storage_for_openshift_container_platform_3.4/index))
* an associated RESTful based volume management framework ([heketi](https://github.com/heketi/heketi))
* containerized


## 1000 feet view

GlusterFS aggregates various storage servers over network interconnects into one large parallel network file system.

GlusterFS relies on an elastic hashing algorithm, rather than using either a centralized or distributed metadata model. This no-metadata server architecture ensures better performance, linear scalability, and reliability.

![Arch](https://github.com/neoseele/gluster-demo/raw/master/resources/RH_Gluster_Storage_diagrams_334434_0415_JCS_5.png)


This [doc](http://docs.gluster.org/en/latest/Quick-Start-Guide/Architecture/#overall-working-of-glusterfs) explains a bit more of how GlusterFS works:

* a gluster management daemon (glusterd) binary is created as soon as GlusterFS is installed in a node
* glusterd should be running in all participating nodes in the cluster
* after starting glusterd, a trusted server pool(TSP) can be created consisting of all storage server nodes (TSP can contain even a single node)
* bricks, which basically are directories, needed to be created in the participating nodes
* any number of bricks from this TSP can be clubbed together to form a volume
* this volume can be mount on a client machine like this. **IP or hostname** can be of any node in the TSP in which the required volume is created
  ```
  mount.glusterfs `<IP or hostname>`:`<volume_name>` `<mount_point>`
  ```

### Components

* node
* brick (any directory on an underlying disk filesystem)
* glusterd (the daemon is used for elastic volume management)
* glusterfsd (each brick has a matching glusterfsd)
*

CNS with GlusterFS
---

Since GlusterFS

* is a scale-out distributed filesystem that utilized the node's local storage
* is containerized
* has a RESTful based volume management API

We can now turn a GKE cluster (with local-ssd enabled) into a GlusterFS cluster. Workload running in the cluster will be able to use persistent ReadWriteMany storage based on local-ssd (via replicated GlusterFS volumes).

Thankfully, there is already a [open-source project](https://github.com/gluster/gluster-kubernetes) for this. However the deployment code requires XFS, which is not available in COS, I have to [improvise](https://github.com/neoseele/heketi/tree/ext4).

## Demo
