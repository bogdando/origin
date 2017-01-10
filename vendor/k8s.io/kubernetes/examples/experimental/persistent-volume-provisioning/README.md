<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->


<!-- END MUNGE: UNVERSIONED_WARNING -->

## Persistent Volume Provisioning

This example shows how to use experimental persistent volume provisioning.

### Pre-requisites

This example assumes that you have an understanding of Kubernetes administration and can modify the
scripts that launch kube-controller-manager.

### Admin Configuration

The admin must define `StorageClass` objects that describe named "classes" of storage offered in a cluster. Different classes might map to arbitrary levels or policies determined by the admin. When configuring a `StorageClass` object for persistent volume provisioning, the admin will need to describe the type of provisioner to use and the parameters that will be used by the provisioner when it provisions a `PersistentVolume` belonging to the class.

The name of a StorageClass object is significant, and is how users can request a particular class, by specifying the name in their `PersistentVolumeClaim`. The `provisioner` field must be specified as it determines what volume plugin is used for provisioning PVs. 2 cloud providers will be provided in the beta version of this feature: EBS and GCE. The `parameters` field contains the parameters that describe volumes belonging to the storage class. Different parameters may be accepted depending on the `provisioner`. For example, the value `io1`, for the parameter `type`, and the parameter `iopsPerGB` are specific to EBS . When a parameter is omitted, some default is used.

#### AWS

```yaml
kind: StorageClass
apiVersion: extensions/v1beta1
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  zone: us-east-1d
  iopsPerGB: "10"
```

* `type`: `io1`, `gp2`, `sc1`, `st1`. See AWS docs for details. Default: `gp2`.
* `zone`: AWS zone. If not specified, a random zone from those where Kubernetes cluster has a node is chosen.
* `iopsPerGB`: only for `io1` volumes. I/O operations per second per GiB. AWS volume plugin multiplies this with size of requested volume to compute IOPS of the volume and caps it at 20 000 IOPS (maximum supported by AWS, see AWS docs).
* `encrypted`: denotes whether the EBS volume should be encrypted or not. Valid values are `true` or `false`.
* `kmsKeyId`: optional. The full Amazon Resource Name of the key to use when encrypting the volume. If none is supplied but `encrypted` is true, a key is generated by AWS. See AWS docs for valid ARN value.

#### GCE

```yaml
kind: StorageClass
apiVersion: extensions/v1beta1
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  zone: us-central1-a
```

* `type`: `pd-standard` or `pd-ssd`. Default: `pd-ssd`
* `zone`: GCE zone. If not specified, a random zone in the same region as controller-manager will be chosen.

#### GLUSTERFS
```yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/glusterfs
parameters:
  endpoint: "glusterfs-cluster"
  resturl: "http://127.0.0.1:8081"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  gidMin: "40000"
  gidMax: "50000"
```

* `endpoint`: `glusterfs-cluster` is the endpoint name which includes GlusterFS trusted pool IP addresses. This parameter is mandatory. We need to also create a service for this endpoint, so that the endpoint will be persisted. This service can be without a selector to tell Kubernetes we want to add its endpoints manually.  Please note that, glusterfs plugin looks for the endpoint in the pod namespace, so it is mandatory that the endpoint and service have to be created in Pod's namespace for successful mount of gluster volumes in the pod.
* `resturl` : Gluster REST service/Heketi service url which provision gluster volumes on demand. The general format should be `IPaddress:Port` and this is a mandatory parameter for GlusterFS dynamic provisioner. If Heketi service is exposed as a routable service in openshift/kubernetes setup, this can have a format similar to
`http://heketi-storage-project.cloudapps.mystorage.com` where the fqdn is a resolvable heketi service url.
* `restauthenabled` : Gluster REST service authentication boolean that enables authentication to the REST server. If this value is 'true', `restuser` and `restuserkey` or `secretNamespace` + `secretName` have to be filled. This option is deprecated, authentication is enabled when any of `restuser`, `restuserkey`, `secretName` or `secretNamespace` is specified.
* `restuser` : Gluster REST service/Heketi user who has access to create volumes in the Gluster Trusted Pool.
* `restuserkey` : Gluster REST service/Heketi user's password which will be used for authentication to the REST server. This parameter is deprecated in favor of `secretNamespace` + `secretName`.
* `secretNamespace` + `secretName` : Identification of Secret instance that containes user password to use when talking to Gluster REST service. These parameters are optional, empty password will be used when both `secretNamespace` and `secretName` are omitted.

When both `restuserkey` and `secretNamespace` + `secretName` is specified, the secret will be used.

Example of a secret can be found in [glusterfs-provisioning-secret.yaml](glusterfs-provisioning-secret.yaml).

* `gidMin` + `gidMax` : The minimum and maximum value of GID range for the storage class. A unique value (GID) in this range ( gidMin-gidMax ) will be used for dynamically provisioned volumes. These are optional values. If not specified, the volume will be provisioned with a value between 2000-2147483647 which are defaults for gidMin and gidMax respectively.

Reference : ([How to configure Heketi](https://github.com/heketi/heketi/wiki/Setting-up-the-topology))

Create endpoints

As in example [glusterfs-endpoints.json](../../volumes/glusterfs/glusterfs-endpoints.json) file, the "IP" field should be filled with the address of a node in the Glusterfs server cluster. It is fine to give any valid value (from 1 to 65535) to the "port" field.

Create the endpoints,

```sh
$ kubectl create -f examples/volumes/glusterfs/glusterfs-endpoints.json
```

You can verify that the endpoints are successfully created by running

```sh
$ kubectl get endpoints
NAME                ENDPOINTS
glusterfs-cluster   10.240.106.152:1,10.240.79.157:1
```

We need also create a service for this endpoints, so that the endpoints will be persisted. It is possible to create `service` without a selector to tell Kubernetes we want to add its endpoints manually. For an example service file refer [glusterfs-service.json](../../volumes/glusterfs/glusterfs-service.json).

Use this command to create the service:

```sh
$ kubectl create -f examples/volumes/glusterfs/glusterfs-service.json
```

#### OpenStack Cinder

```yaml
kind: StorageClass
apiVersion: extensions/v1beta1
metadata:
  name: gold
provisioner: kubernetes.io/cinder
parameters:
  type: fast
  availability: nova
```

* `type`: [VolumeType](http://docs.openstack.org/admin-guide/dashboard-manage-volumes.html) created in Cinder. Default is empty.
* `availability`: Availability Zone. Default is empty.

### User provisioning requests

Users request dynamically provisioned storage by including a storage class in their `PersistentVolumeClaim`.
The annotation `volume.beta.kubernetes.io/storage-class` is used to access this experimental feature. It is required that this value matches the name of a `StorageClass` configured by the administrator.
In the future, the storage class may remain in an annotation or become a field on the claim itself.

```
{
  "kind": "PersistentVolumeClaim",
  "apiVersion": "v1",
  "metadata": {
    "name": "claim1",
    "annotations": {
        "volume.beta.kubernetes.io/storage-class": "slow"
    }
  },
  "spec": {
    "accessModes": [
      "ReadWriteOnce"
    ],
    "resources": {
      "requests": {
        "storage": "3Gi"
      }
    }
  }
}
```

### Sample output

This example uses GCE but any provisioner would follow the same flow.

First we note there are no Persistent Volumes in the cluster.  After creating a storage class and a claim including that storage class, we see a new PV is created
and automatically bound to the claim requesting storage.


```
$ kubectl get pv

$ kubectl create -f examples/experimental/persistent-volume-provisioning/gce-pd.yaml
storageclass "slow" created

$ kubectl create -f examples/experimental/persistent-volume-provisioning/claim1.json
persistentvolumeclaim "claim1" created

$ kubectl get pv
NAME                                       CAPACITY   ACCESSMODES   STATUS    CLAIM                        REASON    AGE
pvc-bb6d2f0c-534c-11e6-9348-42010af00002   3Gi        RWO           Bound     default/claim1                         4s

$ kubectl get pvc
NAME      LABELS    STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
claim1    <none>    Bound     pvc-bb6d2f0c-534c-11e6-9348-42010af00002   3Gi        RWO           7s

# delete the claim to release the volume
$ kubectl delete pvc claim1
persistentvolumeclaim "claim1" deleted

# the volume is deleted in response to being release of its claim
$ kubectl get pv

```



<!-- BEGIN MUNGE: IS_VERSIONED -->
<!-- TAG IS_VERSIONED -->
<!-- END MUNGE: IS_VERSIONED -->


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/examples/experimental/persistent-volume-provisioning/README.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->