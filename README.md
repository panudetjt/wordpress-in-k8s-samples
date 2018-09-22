# install wordpress with k8s
This is my note from [Using Persistent Disks with WordPress and MySQL](https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk) google cloud tutorial. if I done something wrong feel free to correct me. I will very appreciate it.

## Instruction

1. First, we need to create a Volume for storing MySQL and Wordpress data so we will create a PersistentVolumeClaim for claiming a volume to use in our pod. for describe every single line of `.yaml` file can see below.
```Shell Session
$ kubectl apply -f mysql-volumeclaim.yaml
$ kubectl apply -f wordpress-volumeclaim.yaml
```
you can see about `apply` command see in [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/)

2. TODO

## Describe .yaml
### mysql-volumeclaim.yaml, wordpress-volumeclaim.yaml
This is a definition of what [kind](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#persistentvolumeclaim-v1-core) of this file and its API version.
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
```
Every object kind MUST have the following metadata in a nested object field called ["metadata"](https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md#metadata). 
+ **namespace**: a namespace is a DNS compatible label that objects are subdivided into. The default namespace is 'default'. See [the namespace docs](https://kubernetes.io/docs/user-guide/namespaces/) for more.
+ **name**: a string that uniquely identifies this object within the current namespace (see [the identifiers docs](https://kubernetes.io/docs/user-guide/identifiers/)). This value is used in the path when retrieving an individual object.
+ **uid**: a unique in time and space value (typically an RFC 4122 generated identifier, see [the identifiers docs](https://kubernetes.io/docs/user-guide/identifiers/)) used to distinguish between objects with the same name that have been deleted and recreated
```yaml
metadata:
  name: mysql-volumeclaim
```
the **spec** field is the field that tell k8s what we what k8s to do in this file we have 
  + **accessModes**: AccessModes contains the desired access modes the volume should have. that can be
    + ReadWriteOnce – the volume can be mounted as read-write by a single node
    + ReadOnlyMany – the volume can be mounted read-only by many nodes
    + ReadWriteMany – the volume can be mounted as read-write by many nodes

    PersistentVolumeClaim use the same conventions as PersistentVolumes when requesting storage with specific access modes.
    
    More info: https://kubernetes.io/docs/concepts/storage/persistent-volumes
  + **resources**: PersistentVolumeClaim, like pods, can request specific quantities of a resource. In this case, the request is for storage. The same [resource model](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#resourcerequirements-v1-core) applies to both volumes and claims.
    + **requests**: Requests describes the minimum amount of compute resources required. If Requests is omitted for a container, it defaults to Limits if that is explicitly specified, otherwise to an implementation-defined value.
    
      More info: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#resourcerequirements-v1-core
    
      I cannot find what resources that I can requests.  So I assume I can only request storage for this ["kind"](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#persistentvolumeclaim-v1-core). I think the unit that we request we can use same as the unit of [the memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory).


```yaml
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
```