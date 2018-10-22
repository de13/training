# Requests and Limits

Multi-tenants environments probably require that we take more care to know what we consume that legacy environments.

Indeed, sharing resources is not easy, and the consumption of these must be carefully monitored.

Whether to avoid noisy neighbors, or simply to prevent an application from losing control and consuming all available resources, it is good practice to define what it needs in normal use, and how far she can go in a burst period.

## CPU

[de13/spaceoddity](https://hub.docker.com/r/de13/spaceoddity) is an application specially developed to demonstrate these features. At first, we will launch it to make a CPU burst for 5 minutes:

```
apiVersion: v1
kind: Pod
metadata:
  name: cpu-limit
  labels:
    lab: cpu-limit
spec:
  containers:
  - name: spaceoddity
    image: de13/spaceoddity:v0.1
    command: ["./goapp"]
    args: ["-cpu", "-s", "300"]
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f space1.yaml
pod/cpu-limit created
admin@ip-172-31-3-110:~/resources$ kubectl top po -l lab=cpu-limit
NAME        CPU(cores)   MEMORY(bytes)
cpu-limit   1927m        5Mi
admin@ip-172-31-3-110:~/resources$ kubectl delete -f space1.yaml
pod "cpu-limit" deleted
```

Oops! The CPU limit has not been implemented, so our Pod has consumed all the computing time of the two CPUs that are available to us ... And here is a human error that could have a strong impact in production !

We will place our limit and our request at 300m, which means 0.3 CPUs:

```
apiVersion: v1
kind: Pod
metadata:
  name: cpu-limit2
  labels:
    lab: cpu-limit
spec:
  containers:
  - name: spaceoddity
    image: de13/spaceoddity:v0.1
    command: ["./goapp"]
    args: ["-cpu", "-s", "300"]
    resources:
      requests:
        cpu: "300m"
      limits:
        cpu: "300m"
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f space2.yaml
pod/cpu-limit2 created
admin@ip-172-31-3-110:~/resources$ kubectl top po -l lab=cpu-limit
NAME         CPU(cores)   MEMORY(bytes)
cpu-limit2   300m         5Mi
admin@ip-172-31-3-110:~/resources$ kubectl delete -f space2.yaml
pod "cpu-limit2" deleted

```

As in the previous case, our application consumes the entire CPU time, except that this time it is limited to 300m, so that this burst (invonlontaire) has no real impact on the rest of the production.

## Memory

Let's deploy an application with a memory leak. We force it to 128Mi maximum, for regular use at 64Mi.

Note the last line: `restartPolicy: OnFailure`. If our application does not have enough memory to work, we ask that it be restarted:

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-limit1
  labels:
    lab: memory-limit
spec:
  containers:
  - name: spaceoddity
    image: de13/spaceoddity:v0.1
    command: ["./goapp"]
    args: ["-mem", "-s", "600"]
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"
  restartPolicy: OnFailure
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f space3.yaml
pod/memory-limit1 created
admin@ip-172-31-3-110:~/resources$ kubectl get po -l lab=memory-limit
NAME            READY   STATUS      RESTARTS   AGE
memory-limit1   1/1     Running   0          25s
admin@ip-172-31-3-110:~/resources$ kubectl get po -l lab=memory-limit
NAME            READY   STATUS      RESTARTS   AGE
memory-limit1   0/1     OOMKilled   2          35s
admin@ip-172-31-3-110:~/resources$ kubectl get po -l lab=memory-limit
NAME            READY   STATUS    RESTARTS   AGE
memory-limit1   1/1     Running   2          44s
admin@ip-172-31-3-110:~/resources$ kubectl get po -l lab=memory-limit
NAME            READY   STATUS      RESTARTS   AGE
memory-limit1   0/1     OOMKilled   2          58s
admin@ip-172-31-3-110:~/resources$ kubectl delete -f space3.yaml
pod "memory-limit1" deleted
```

Indeed, our application is periodically restarted after an OOM.

## Conclusion

It is important to always maintain control over your deployment so that you do not run into unpleasant surprises. The requests and the limits are two tools that Kubernetes puts at our disposal to do this.