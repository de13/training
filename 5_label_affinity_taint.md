# Label, Affinity, Taint

In this lab, we will see the different ways to constrain a Pod to run on a specific cluster node.

In general, you do not have to worry about where a Pod is running, the Kubernetes Scheduler has an algorithm sophisticated enough to make Pods in the same stack homogeneously distributed on different nodes, and never on missing nodes of CPU or memory.

But it is possible that you want to force a Pod to turn on a specific node (a machine equipped with a GPU for example), or to force two Pods to be collocated in the same geographical area, because they are brought to communicate a lot.

## Kubernetes scheduler

Small demonstartion of how the scheduler distributes your Pod: first create a file `foo.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: foo
  name: bar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: foo
  template:
    metadata:
      labels:
        app: foo
    spec:
      containers:
      - image: registry.hub.docker.com/library/nginx:1.15.5
        name: nginx
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f foo.yaml
deployment.apps/bar created
admin@ip-172-31-3-110:~/resources$ k get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP          NODE      NOMINATED NODE
bar-5bcf4dd466-4bvb7   1/1     Running   0          7s    10.42.1.7   worker1   <none>
bar-5bcf4dd466-vbvgl   1/1     Running   0          7s    10.42.2.2   worker2   <none>
```

Our two pods are on two different nodes.

## nodeSelector

It's time to force our Pods. First, let's assign random labels to our nodes:

```
admin@ip-172-31-3-110:~/resources$ kubectl label no/worker1 arch=x86
node/worker1 labeled
admin@ip-172-31-3-110:~/resources$ kubectl label no/worker1 dc=luxembourg
node/worker1 labeled
admin@ip-172-31-3-110:~/resources$ kubectl label no/worker2 dc=london
node/worker2 labeled
admin@ip-172-31-3-110:~/resources$ kubectl get no -l arch=x86
NAME      STATUS   ROLES    AGE   VERSION
worker1   Ready    worker   7m    v1.11.3
admin@ip-172-31-3-110:~/resources$ kubectl get no -l dc=london
NAME      STATUS   ROLES    AGE   VERSION
worker2   Ready    worker   8m    v1.11.3
```

Next, let's create a Pod that should run in Luxembourg:

```
apiVersion: v1
kind: Pod
metadata:
  name: luxembourg
  labels:
    app: local
spec:
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
  nodeSelector:
    dc: luxembourg
```

```
admin@ip-172-31-3-110:~/resources$ kubectl get po/luxembourg -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP          NODE      NOMINATED NODE
luxembourg   1/1     Running   0          16s   10.42.1.9   worker1   <none>
admin@ip-172-31-3-110:~/resources$ kubectl get no -l dc=luxembourg
NAME      STATUS   ROLES    AGE   VERSION
worker1   Ready    worker   12m   v1.11.3
```

The Pod was placed on the node we wanted.

What happens in the event that there is no resource corresponding to our request?

```
apiVersion: v1
kind: Pod
metadata:
  name: paris
  labels:
    app: local
spec:
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
  nodeSelector:
    dc: paris
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f paris.yaml
pod/paris created
admin@ip-172-31-3-110:~/resources$ kubectl get po/paris -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE
paris   0/1     Pending   0          9s    <none>   <none>   <none>
```

The Pod remains `pending` and can not be schedule. It is therefore advisable to be careful using the labels, because under certain conditions (the loss of a dc for example), our workloads may not be executed.

Many of the constraints related to the topology can however be solved thanks to two built-in labels:

- failure-domain.beta.kubernetes.io/zone
- failure-domain.beta.kubernetes.io/region

If we add these labels to our nodes (eg `failure-domain.beta.kubernetes.io/zone=luxembourg`, `failure-domain.beta.kubernetes.io/zone=london`), the Scheduler will make sure that the Pods of a same deployment (subject to having a number of replicas greater than 2 in this case), be spread over different geographical areas, in order to ensure resilience.

## Node Affinity

We saw that `nodeSelector` was interesting to constrain the placement of Pods, however it has its limits. Affinities and anti-affinities bring much more flexibility.

Delete our previous Pod:

```
admin@ip-172-31-3-110:~/resources$ kubectl delete -f paris.yaml
pod "paris" deleted
```

Then modify the yaml file:

```
apiVersion: v1
kind: Pod
metadata:
  name: paris
  labels:
    app: local
spec:
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
            - key: dc
              operator: In
              values:
              - paris
```

Here we express a preference, no longer a strong constraint.

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f paris.yaml
pod/paris created
admin@ip-172-31-3-110:~/resources$ kubectl get po/paris -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE      NOMINATED NODE
paris   1/1     Running   0          12s   10.42.2.3   worker2   <none>
```

This time, our Pod is placed on a node, even if it does not correspond to the chosen label (failing to find the said label). As a result, even in the absence of the desired resource, our workload is able to run.

`In` is not the only operator we can use, there are also` NotIn`, `Exists`,` DoesNotExist`, `Gt`,` Lt`.

To have an anti-affinity rule, just use `NotIn` instead of` In`. This ensures that your workload will never run on the `` requiredDuringSchedulingIgnoredDuringExecution`` Paris DC, or that it will not run unless you have the `preferredDuringSchedulingIgnoredDuringExecution` option.

## Pod Affinity

Affinities allow us to go even further, allowing affinities between Pods.

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity1
  labels:
    affinity: join
spec:
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
```

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity2
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: affinity
            operator: In
            values:
            - join
        topologyKey: kubernetes.io/hostname
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f aff1.yaml
pod/pod-affinity1 created
admin@ip-172-31-3-110:~/resources$ kubectl apply -f aff2.yaml
pod/pod-affinity2 created
admin@ip-172-31-3-110:~/resources$ kubectl get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP          NODE      NOMINATED NODE
pod-affinity1          1/1     Running   0          2m    10.42.2.4   worker2   <none>
pod-affinity2          1/1     Running   0          24s   10.42.2.5   worker2   <none>
```

And by putting an anti-affinity rule, for example so that two Pods that have a high consumption of I / O are not on the same machine:

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-anti-affinity1
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: affinity
            operator: In
            values:
            - join
        topologyKey: kubernetes.io/hostname
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f anti.yaml
pod/pod-anti-affinity1 created
admin@ip-172-31-3-110:~/resources$ kubectl get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE
pod-affinity1          1/1     Running   0          8m    10.42.2.4    worker2   <none>
pod-affinity2          1/1     Running   0          6m    10.42.2.5    worker2   <none>
pod-anti-affinity1     1/1     Running   0          7s    10.42.1.10   worker1   <none>
```

## Taints and Tolerations

Affinities do great service, but imagine the following case:

You have a cluster of 20 workers, two of which are equipped with GPUs. You put a label on each of them so that your data-science workloads go on those machines and benefit from the GPU for their calculations.

However, after a while, there is no more room on these machines, because other workloads, which do not need a GPU, have also been scheduled on these nodes.

An obvious solution, even if not very effective and prone to error, would be to put a label on all other machines, and you would have `gpu=yes` and` gpu=no`. In fact, you would have to put a selector on ALL your deployments, to tell them if they should go on the machines equipped with GPU or not...

Fortunately, there is in Kubernetes a much more effective means of achieving the same results, which requires much less effort, and which is not subject to error: the taints.

We will first put a taint on our node:

```
admin@ip-172-31-3-110:~$ k taint node worker1 gpu=yes:NoSchedule
node/worker1 tainted
admin@ip-172-31-3-110:~$ k describe no/worker1
...
Taints:             gpu=yes:NoSchedule
...
```

Now, let's create a simple Pod that does not need a GPU:

```
apiVersion: v1
kind: Pod
metadata:
  name: nogpu
  labels:
    app: nogpu
spec:
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f nogpu.yaml
pod/nogpu created
admin@ip-172-31-3-110:~/resources$ k get po/nogpu -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE
nogpu   1/1     Running   0          49s   10.42.2.11   worker2   <none>
```

One can be 100% certain that this Pod will never go on worker1. By the way, let's create a second Pod to prove it:

```
apiVersion: v1
kind: Pod
metadata:
  name: gpuworker1
  labels:
    app: gpuworker1
spec:
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
  nodeSelector:
    proc: gpu
```

```
admin@ip-172-31-3-110:~/resources$ k label node worker1 proc=gpu
node/worker1 labeled
admin@ip-172-31-3-110:~/resources$ kubectl apply -f gpuworker1.yaml
pod/gpuworker1 created
admin@ip-172-31-3-110:~/resources$ kubectl get pod/gpuworker1 -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE
gpuworker1   0/1     Pending   0          24s   <none>   <none>   <none>
```

The Pod can not be run on worker1, because it has a taint `gpu=yes: NoSchedule`.

To be able to run a Pod on this node, all need a toleration, of type:

```
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "yes"
  effect: "NoSchedule"
```

** Warning **: the Toleration will allow the Pod to run on the node bearing the taint, but it does not guarantee an affinity with it. This is why it is often necessary to couple the taint and an affinity:

```
apiVersion: v1
kind: Pod
metadata:
  name: withgpu
  labels:
    app: withgpu
spec:
  containers:
  - image: registry.hub.docker.com/library/nginx:1.15.5
    name: nginx
  nodeSelector:
    proc: gpu
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "yes"
    effect: "NoSchedule"
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f withgpu.yaml
pod/withgpu created
admin@ip-172-31-3-110:~/resources$ kubectl get pod/withgpu -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE
withgpu   1/1     Running   0          11s   10.42.1.19   worker1   <none>
```

Just like the previous Pod, we choose to place this Pod on the worker1, but in addition to the previous Pod, we allow, thanks to a tolerance, this Pod to be executed on it.

In the case of a small cluster where you can not fully dedicate the hardware, you may prefer to use `PreferNoSchedule` than `NoSchedule`.

## Conclusion

Affinities and taints are of great service when a cluster grows in size and constraints appear. But we must make good use of it, keeping in mind that they are there to reduce the operational burden and not increase it.