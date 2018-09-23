# Install wordpress with k8s
This is my note from [Using Persistent Disks with WordPress and MySQL](https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk) google cloud tutorial. if I done something wrong feel free to correct me. I will very appreciate it.

## Instruction

1. First, We need to create the cluster with following command
```Shell Session
$ gcloud container clusters create <Your cluster name> --num-nodes=3
```
> **Note**: If you are using an existing Google Kubernetes Engine cluster or if you have created a cluster through Google Cloud Platform Console, you need to run the following command to retrieve cluster credentials and configure kubectl command-line tool with them:
```Shell Session
$ gcloud container clusters get-credentials <Your cluster name>
```
> If you have already created a cluster with the gcloud container clusters create command listed above, this step is not necessary.
2. First, we need to create a Volume for storing MySQL and Wordpress data so we will create a PersistentVolumeClaim for claiming a volume to use in our pod. for describe every single line of `.yaml` file can see below.
```Shell Session
$ kubectl apply -f mysql-volumeclaim.yaml
$ kubectl apply -f wordpress-volumeclaim.yaml
```
you can see about `apply` command in [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/).
if you want to check the claims get bound just run following command.
```Shell session
$ kubectl get pvc
```
output:
```Shell session
NAME                    STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-volumeclaim       Bound     pvc-e1c21c51-be74-11e8-950c-42010a94008f   200Gi      RWO            standard       3m
wordpress-volumeclaim   Bound     pvc-06fec630-be75-11e8-950c-42010a94008f   200Gi      RWO            standard       2m
```
3. Set up MySQL
    
    We will create Kubernetes Secret to store the password for the database. I don't why we need this maybe you can tell me after reading this [best practice to use Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/#best-practices). Anyway, I will create the secret named `mysql` via following command (and replace YOUR_PASSWORD with a passphrase of your choice):.
    ```Shell Session
    $ kubectl create secret generic mysql --from-literal=password=YOUR_PASSWORD
    ```
    you can view and confirm what is you just created by 
    ```Shell Session
    $ kubectl get secret mysql -o yaml
    ```
    the output will be like
    ```yaml
    ...
    data:
      password: <Your password>
    ...
    ```
    your password will encode to base64.
    
    after you create the secret. let's deploy by following command:
    ```Shell Session
    $ kubectl create -f mysql.yaml
    ```
    Check to see if the Pod is running. It might take a few minutes for the Pod to transition to Running status as attaching the persistent disk to the compute node takes a while:
    ```Shell Session
    $ kubectl get pod -l app=mysql
    ```
    the output:
    ```
    NAME                 READY     STATUS    RESTARTS   AGE
    mysql-259040-flmqh   1/1       Running   0          3m
    ```
    ### Create MySQL service
    The next step is to create a [Service](https://kubernetes.io/docs/concepts/services-networking/service/) to expose the MySQL container and make it accessible from the wordpress container you are going to create.
    You will use the Service manifest defined in [mysql-service.yaml](#mysql-service.yaml). 
    
    This manifest describes a Service that creates a [Cluster IP](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) on port 3306 for the Pods matching the label `app: mysql`. The `mysql` container deployed in the previous step has this label. In this case, you use `type:ClusterIP` for the Service, as this value makes the Service reachable only from within the cluster.
    
    The Cluster IP created routes traffic to the MySQL container from all compute nodes in the cluster and is not accessible to clients outside the cluster. Once the Service is created, the `wordpress` container can use the DNS name of the `mysql` container to communicate, even though they are not in the same compute node.

    To deploy this manifest file, run:
    ```Shell Session
    $ kubectl create -f mysql-service.yaml
    ```
    



## Describe .yaml (I don't know what its called)
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
    
      More info: https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#resourcerequirements-v1-core
    
      I cannot find what resources that I can requests.  So I assume I can only request storage for this ["kind"](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#persistentvolumeclaim-v1-core). I think the unit that we request we can use same as the unit of [the memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory).


```yaml
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
```
### mysql.yaml
This manifest describes a Deployment with a single instance MySQL Pod which will have the `MYSQL_ROOT_PASSWORD` environment variable whose value is set from the secret created. The mysql container will use the `PersistentVolumeClaim` and mount the persistent disk at `/var/lib/mysql` inside the container.
```yaml
kind: Deployment
apiVersion: v1
metadata:
  name: mysql
  labels:
    app: mysql
```
this is the deployment file that uses API version and metadata that tells the **name** of deployment and **labels** it with label name **app** and its value is **mysql**.

for further read see: https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#deployment-v1-apps
```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
```
**spec**: Specification of the desired behavior of the Deployment.
  + **replicas**: Number of desired pods. This is a pointer to distinguish between explicit zero and not specified. Defaults to 1.
  + **selector**: The selector field defines how the Deployment finds which Pods to manage. In this case, you simply select a label that is defined in the Pod template (`app: mysql`). However, more sophisticated selection rules are possible, as long as the Pod template itself satisfies the rule.
```yaml
  template:
    metadata:
      labels:
        app: mysql
```
this is the `template` of our pod every pod has its own `metadata` so we will `labels` it as `app: mysql` for convenience when we get more `replicas`.
```yaml
    spec:
      containers:
        - image: mysql:5.7
          name: mysql
```
It tells this template use the containers image called `mysql` at version `5.7` form [docker hub](https://hub.docker.com/_/mysql/) and named it as `mysql`. yeah this pod can have multiple container but MUST be at least one container.
```yaml
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
```
List of environment variables to set in the container. Cannot be updated. and as you can see `name: MYSQL_ROOT_PASSWORD` to define environment varaibles name is `MYSQL_ROOT_PASSWORD` and get its `valueFrom` the kubernete secret by key reference that named `name: mysql` and get `key: password` that we defined in [step 3]().
```yaml
          ports:
            - containerPort: 3306
              name: mysql
```
+ **ports**: List of ports to expose from the container. Exposing a port here gives the system additional information about the network connections a container uses, but is primarily informational. Not specifying a port here DOES NOT prevent that port from being exposed. Any port which is listening on the default "0.0.0.0" address inside a container will be accessible from the network. Cannot be updated.
  + **containerPort**: Number of port to expose on the pod's IP address. This must be a valid port number, 0 < x < 65536.
  + **name**: If specified, this must be an IANA_SVC_NAME and unique within the pod. Each named port in a pod must have a unique name. Name for the port that can be referred to by services.
```yaml
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
              subPath: mysql
```
  + **volumeMounts**: Pod volumes to mount into the container's filesystem. Cannot be updated.
    + **name**: This must match the Name of a Volume. (what is that? I don't understand)
    + **mountPath**: Path within the container at which the volume should be mounted. Must not contain ':'.
    + **subPath**: Path within the volume from which the container's volume should be mounted. Defaults to "" (volume's root).
    
      (I don't know why we needed this but if not   specify `subPath` we cannot deploy `mysql:5.7` for further read see below)
```yaml
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-volumeclaim
```
  + **volumes**: List of volumes that can be mounted by containers belonging to the pod. More info: https://kubernetes.io/docs/concepts/storage/volumes
    + **name**: Volume's name. Must be a DNS_LABEL and unique within the pod. More info: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names
    + **persistentVolumeClaim**: PersistentVolumeClaimVolumeSource represents a reference to a PersistentVolumeClaim in the same namespace. More info: https://kubernetes.io/docs/concepts/storage/persistent-volumes#persistentvolumeclaims
      + **claimName**: ClaimName is the name of a PersistentVolumeClaim in the same namespace as the pod using this volume. More info: https://kubernetes.io/docs/concepts/storage/persistent-volumes#persistentvolumeclaims

### mysql-service.yaml
A Kubernetes `Service` is an abstraction which defines a logical set of `Pods` and a policy by which to access them - sometimes called a micro-service. The set of `Pods` targeted by a `Service` is (usually) determined by a [Label Selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors).
```yaml
kind: Service
apiVersion: v1
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  type: ClusterIP
  ports:
    - port: 3306
  selector:
    app: mysql
```
+ **kind**: is a string value representing the REST resource this object represents. Servers may infer this from the endpoint the client submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds
+ **apiVersion**: APIVersion defines the versioned schema of this representation of an object. Servers should convert recognized schemas to the latest internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#resources
+ **metadata**: Standard object's metadata.
+ **spec**: Spec defines the behavior of a service. https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status
  + **type**: determines how the Service is exposed. Defaults to ClusterIP. Valid options are ExternalName, ClusterIP, NodePort, and LoadBalancer. "ExternalName" maps to the specified externalName. "ClusterIP" allocates a cluster-internal IP address for load-balancing to endpoints. Endpoints are determined by the selector or if that is not specified, by manual construction of an Endpoints object. If clusterIP is "None", no virtual IP is allocated and the endpoints are published as a set of endpoints rather than a stable IP. "NodePort" builds on ClusterIP and allocates a port on every node which routes to the clusterIP. "LoadBalancer" builds on NodePort and creates an external load-balancer (if supported in the current cloud) which routes to the clusterIP. More info: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services---service-types
  + **ports**: The list of ports that are exposed by this service. More info: https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
    + port: The port that will be exposed by this service.
  + **selector**: Route service traffic to pods with label keys and values matching this selector. If empty or not present, the service is assumed to have an external process managing its endpoints, which Kubernetes will not modify. Only applies to types ClusterIP, NodePort, and LoadBalancer. Ignored if type is ExternalName. More info: https://kubernetes.io/docs/concepts/services-networking/service/




# REFERANCE
https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/
https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md#resources

Deploy `mysql:5.7` issues:
https://github.com/docker-library/mysql/issues/186
https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/#deploy-mysql

# LICENSE
mostly the content of this repository is taken from [Using Persistent Disks with WordPress and MySQL](https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk) that licensed under the [Creative Commons Attribution 3.0 License](https://creativecommons.org/licenses/by/3.0/) and code samples are licensed under the [Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0).
I own nothing. if any word that I wrote it under license WTFPL (Do What the Fuck You Want To Public License).