# Persistent Volumes

## Local Storage

The simplest volumes are local volumes. It is not a question here of having long-time persistence (eg caching), because obviously they are intimately linked to the life cycle of the machine and can not be moved.

```
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f redis.yaml
pod/redis created
admin@ip-172-31-3-110:~/resources$ kubectl describe po redis
...
    Mounts:
      /data/redis from redis-storage (rw)
...
```

## Block

To have persistence over long time, it is necessary to use other volume drivers (ebs, rbd ...)

This often involves being on an IaaS (eg AWS, GCP), or using a Cloud-Native storage service (eg [rook](https://rook.io/)).

Because our cluster is on AWS, we have the ability to create a volume:

```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: ebs-pv
  labels:
    type: amazonEBS
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: vol-0747270d1c47a4ca6
    fsType: ext4
```

However, the following example will only work if an operator creates the volume for us upstream and gives us the volume reference (volumeID). This makes obtaining a particularly laborious volume in terms of operations.

Fortunately, on some IaaS (eg AWS, vSphere) it is possible to automate things thanks to Storage Class:

```
admin@ip-172-31-3-110:~/resources$ kubectl get sc
NAME                PROVISIONER             AGE
default (default)   kubernetes.io/aws-ebs   21h
```

Once a Storage Class is present, there is no need to worry about the creation of PersistentVolume, just make a PersistentVolumeClaim so that the volume is automatically created by the provider:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
  labels:
    env: dev
spec:
  storageClassName: default
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Here, `storageClassName` is the` StorageClass` defined on our cluster.

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f claim.yaml
persistentvolumeclaim/my-claim created
admin@ip-172-31-3-110:~/resources$ kubectl get pvc
NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-claim   Pending                                      default        5s
admin@ip-172-31-3-110:~/resources$ kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-claim   Bound    pvc-60f69899-d5cb-11e8-9d48-0218d63ab908   5Gi        RWO            default        11s
admin@ip-172-31-3-110:~/resources$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
pvc-60f69899-d5cb-11e8-9d48-0218d63ab908   5Gi        RWO            Delete           Bound    default/my-claim   default                 6s
admin@ip-172-31-3-110:~/resources$ kubectl delete -f claim.yaml
persistentvolumeclaim "my-claim" deleted
admin@ip-172-31-3-110:~/resources$ kubectl get pvc
No resources found.
admin@ip-172-31-3-110:~/resources$ kubectl get pv
No resources found.
```

The simplest option is to place the Claim in the same yaml file as the application for which you are using a Claim:

```
admin@ip-172-31-3-110:~/resources$ kubectl create ns mongo2
namespace/mongo2 created
```

```
apiVersion: v1
kind: Service
metadata:
  name: mongo2
  namespace: mongo2
  labels:
    name: mongo2
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    run: mongo
--- 
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo2
  namespace: mongo2
spec:
  serviceName: "mongo2"
  replicas: 3
  template:
    metadata:
      labels:
        run: mongo
    spec:
      containers:
      - image: mongo:3.4.1
        name: mongodb
        command:
        - mongod
        - --replSet
        - rs0
        ports:
        - containerPort: 27017
          name: peer
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "default"
      resources:
        requests:
          storage: 2Gi
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f mongo2.yaml
service/mongo2 created
statefulset.apps/mongo2 created
admin@ip-172-31-3-110:~/resources$ kubectl -n mongo2 get po -w
NAME       READY   STATUS              RESTARTS   AGE
mongo2-0   0/1     ContainerCreating   0          10s
mongo2-0   0/1   ContainerCreating   0     18s
mongo2-0   1/1   Running   0     19s
mongo2-1   0/1   Pending   0     0s
mongo2-1   0/1   Pending   0     0s
mongo2-1   0/1   Pending   0     11s
mongo2-1   0/1   ContainerCreating   0     11s
mongo2-1   0/1   ContainerCreating   0     29s
mongo2-1   1/1   Running   0     29s
mongo2-2   0/1   Pending   0     0s
mongo2-2   0/1   Pending   0     0s
mongo2-2   0/1   Pending   0     12s
mongo2-2   0/1   ContainerCreating   0     12s
mongo2-2   0/1   ContainerCreating   0     29s
mongo2-2   1/1   Running   0     29s
^Cadmin@ip-172-31-3-110:~/resources$ kubectl -n mongo2 get pvc
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-persistent-storage-mongo2-0   Bound    pvc-2c417c00-d5cc-11e8-9d48-0218d63ab908   2Gi        RWO            default        1m
mongo-persistent-storage-mongo2-1   Bound    pvc-3773dfcb-d5cc-11e8-9d48-0218d63ab908   2Gi        RWO            default        1m
mongo-persistent-storage-mongo2-2   Bound    pvc-48988d08-d5cc-11e8-9d48-0218d63ab908   2Gi        RWO            default        44s
admin@ip-172-31-3-110:~/resources$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                      STORAGECLASS   REASON   AGE
pvc-2c417c00-d5cc-11e8-9d48-0218d63ab908   2Gi        RWO            Delete           Bound    mongo2/mongo-persistent-storage-mongo2-0   default                 1m
pvc-3773dfcb-d5cc-11e8-9d48-0218d63ab908   2Gi        RWO            Delete           Bound    mongo2/mongo-persistent-storage-mongo2-1   default                 1m
pvc-48988d08-d5cc-11e8-9d48-0218d63ab908   2Gi        RWO            Delete           Bound    mongo2/mongo-persistent-storage-mongo2-2   default                 37s
```

This time our mongo has storage persist, and even in case of host failure, a replica of mongo will be able to restart on another node without losing is storage.

Be careful though, we are using AWS EBS storage, and this one is bound to the zone (eg `eu-central-1a`); in fact, our mongo instance will have to restart in this AZ (the labels are there for that).

## Conclusion

There are a large variety of storage adapted to all use-cases (eg file, block, object ...). For the block, Storage Class helps to automate storage openness (and deletion), which greatly helps to speed up the implementation of an application that needs persistence.