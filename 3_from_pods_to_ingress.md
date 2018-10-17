# From Pods to Ingress

It's time to get to the heart of the matter!

## Pods

Although the Pod is the atomic unit of Kubernetes, its deployment unit is deployment. It is therefore not possible to create an inline Pod, but it is possible to do it from a yaml file:

Let's create a `pod.yaml` file with this content:

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
spec:
  containers:
  - name: simple-container
    image: nginx
```

```
$ nano pod.yaml
Crtl + x
Y
enter
```

As we have seen, apiVersion is the API version of the resource we want. The Kind is obviously Pod. The only metadata we give him is his name (it's mandatory). Finally, comes the description of the container, at least his name, and the image he will use.

```
student1@ip-172-31-3-110:~$ kubectl apply -f pod.yaml
pod/simple-pod created
```

We use the `kubectl apply` command with the file we created.

```
student1@ip-172-31-3-110:~$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
simple-pod   1/1     Running   0          1m
```

Our Pod is deployed!

Whether Docker is the runtime or not does not matter here, all that matters is that it respects the OCI standards.

Because of this, you can find some commands that we have already used with Docker:

### Capture the logs

```
student1@ip-172-31-3-110:~$ kubectl logs simple-pod
```

### Enter the pod containers

```
student1@ip-172-31-3-110:~$ kubectl exec -ti simple-pod bash
root@simple-pod:/#
```

## Deployment

It's interesting to see how a Pod works, but we will not use it directly in Kubernetes. Instead, let's look at the deployment side:

```
student1@ip-172-31-3-110:~$ kubectl create deployment first-deployment --image=nginx
deployment.apps/first-deployment created
```

What have we done ? We have just launched a Pod:

```
student1@ip-172-31-3-110:~$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
first-deployment-79956f5c54-ghscx   1/1     Running   0          58s
simple-pod                          1/1     Running   0          11m
```

But not a Pod in reality. We have a deployment:

```
student1@ip-172-31-3-110:~$ kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
first-deployment   1         1         1            1           1m
```

And a replicaSet:

```
student1@ip-172-31-3-110:~$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
first-deployment-79956f5c54   1         1         1       1m
```

Unlike the simple Pod, here we can scale our deployment:

```
student1@ip-172-31-3-110:~$ kubectl scale deploy/first-deployment --replicas=3
deployment.extensions/first-deployment scaled
student1@ip-172-31-3-110:~$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
first-deployment-79956f5c54-ghscx   1/1     Running   0          3m
first-deployment-79956f5c54-mkpkh   1/1     Running   0          12s
first-deployment-79956f5c54-r7zsn   1/1     Running   0          12s
simple-pod                          1/1     Running   0          14m
```

Scale out, as well as scale in:

```
student1@ip-172-31-3-110:~$ kubectl scale deploy/first-deployment --replicas=1
deployment.extensions/first-deployment scaled
student1@ip-172-31-3-110:~$ kubectl get pods
NAME                                READY   STATUS        RESTARTS   AGE
first-deployment-79956f5c54-ghscx   1/1     Running       0          4m
first-deployment-79956f5c54-mkpkh   0/1     Terminating   0          46s
first-deployment-79956f5c54-r7zsn   0/1     Terminating   0          46s
simple-pod                          1/1     Running       0          15m
```

We can look at the Pod template launched by the deployment:

```
student1@ip-172-31-3-110:~$ kubectl get po/first-deployment-79956f5c54-ghscx -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: 10.42.2.4/32
  creationTimestamp: 2018-10-17T11:18:34Z
  generateName: first-deployment-79956f5c54-
  labels:
    app: first-deployment
    pod-template-hash: "3551291710"
  name: first-deployment-79956f5c54-ghscx
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: first-deployment-79956f5c54
    uid: 62934f0d-d1fe-11e8-a112-02b0490a9828
  resourceVersion: "13199"
  selfLink: /api/v1/namespaces/default/pods/first-deployment-79956f5c54-ghscx
  uid: 629525dc-d1fe-11e8-a112-02b0490a9828
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-grspr
      readOnly: true
  dnsPolicy: ClusterFirst
  nodeName: student1-worker2
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-grspr
    secret:
      defaultMode: 420
      secretName: default-token-grspr
```

A lot of information here comes from the runtime.

### Rollout

We'll see how easy it is to upgrade a deployment.

First, to follow the best practices, let's move our deployment to 3 pods:

```
student1@ip-172-31-3-110:~$ kubectl scale deploy/first-deployment --replicas=3
deployment.extensions/first-deployment scaled
```

Then we will modify the image of nginx that we use to simulate an upgrade:

```
student1@ip-172-31-3-110:~$ kubectl set image deploy/first-deployment nginx=nginx:1.15.2
deployment.extensions/first-deployment image updated
student1@ip-172-31-3-110:~$ kubectl get pod
NAME                                READY   STATUS              RESTARTS   AGE
first-deployment-5647b7559c-6tv4b   1/1     Running             0          7s
first-deployment-5647b7559c-992pb   0/1     ContainerCreating   0          1s
first-deployment-5647b7559c-k7xrw   1/1     Running             0          14s
first-deployment-79956f5c54-ghscx   1/1     Running             0          1h
first-deployment-79956f5c54-nh9lh   0/1     Terminating         0          3m
first-deployment-79956f5c54-vh92f   0/1     Terminating         0          3m
simple-pod                          1/1     Running             0          2h
```

A Pod is created with the new image, then a Pod is deleted, and so on, until all pods are at the same version.

We can see the history of our deployments with the `history` option.

We can do a very fast rollback on the previous version:

```
student1@ip-172-31-3-110:~$ kubectl rollout history deploy/first-deployment
deployment.extensions/first-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
student1@ip-172-31-3-110:~$ kubectl rollout status deploy/first-deployment
deployment "first-deployment" successfully rolled out
student1@ip-172-31-3-110:~$ kubectl get deploy/first-deployment -o yaml
...
    spec:
      containers:
      - image: nginx:1.15.2
...
student1@ip-172-31-3-110:~$ kubectl rollout undo deploy/first-deployment
deployment.extensions/first-deployment
student1@ip-172-31-3-110:~$ kubectl get deploy/first-deployment -o yaml
    spec:
      containers:
      - image: nginx
```

**Note**: From the beginning, we use the nginx image without specifying a tag (version). Docker assigns him a tag: **nginx:latest**. But using a tag image latest is usually not a good practice. We should have preferred him from the beginning nginx: 1.15.5. Indeed, if the latest tag has a practical side, it does not describe the version in which we are, and our image could have the latest tag is in 1.14.1.

## Services

As we have seen, our deployment serves as an abstraction layer for managing our Pods. But how can we access the application that is in these Pods? Each has a different IP address, which can change at any time?

```
student1@ip-172-31-3-110:~$ for i in hsfpj jnkmb xxvjv ; do kubectl describe po/first-deployment-79956f5c54-$i |grep -i ip ; done
Annotations:        cni.projectcalico.org/podIP: 10.42.2.10/32
IP:                 10.42.2.10
Annotations:        cni.projectcalico.org/podIP: 10.42.1.10/32
IP:                 10.42.1.10
Annotations:        cni.projectcalico.org/podIP: 10.42.2.9/32
IP:                 10.42.2.9
```

The abstraction layer at the network level is the service.

```
student1@ip-172-31-3-110:~$ kubectl expose --name=nginx --port=80 --target-port=80 deploy/first-deployment
service/nginx exposed
student1@ip-172-31-3-110:~$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP   4h
nginx        ClusterIP   10.43.171.201   <none>        80/TCP    14s
```

The IP address we have obtained will remain as long as we do not delete the service, even if the deployment or Pods disappear. They are not coupled.

In addition, the service did not give me an IP address, but also a DNS entry inside the cluster. Our IP address should be resolved in nginx.default.svc.cluster.local, or simply nginx inside the namespace (which we'll see later), nginx.default from other namespaces.

```
student1@ip-172-31-3-110:~$ kubectl exec -ti simple-pod bash
root@simple-pod:/# apt update
root@simple-pod:/# apt install -y dnsutils
root@simple-pod:/# nslookup nginx
Server:		10.43.0.10
Address:	10.43.0.10#53

Name:	nginx.default.svc.cluster.local
Address: 10.43.171.201
root@simple-pod:/# apt install -y curl
root@simple-pod:/# curl nginx
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

We have 3 pods in our deployment, and the service will distribute traffic to these pods in round-robin. To demonstrate this, we will make an aberration in terms of immutable infrastructure:

```
student1@ip-172-31-3-110:~$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
first-deployment-79956f5c54-hsfpj   1/1     Running   0          31m
first-deployment-79956f5c54-jnkmb   1/1     Running   0          31m
first-deployment-79956f5c54-xxvjv   1/1     Running   0          31m
simple-pod                          1/1     Running   0          2h
student1@ip-172-31-3-110:~$ kubectl exec -ti first-deployment-79956f5c54-hsfpj bash
root@first-deployment-79956f5c54-hsfpj:/# echo "Pod1" > /usr/share/nginx/html/index.html
root@first-deployment-79956f5c54-hsfpj:/# exit
student1@ip-172-31-3-110:~$ kubectl exec -ti first-deployment-79956f5c54-jnkmb bash
root@first-deployment-79956f5c54-jnkmb:/# echo "Pod2" > /usr/share/nginx/html/index.html
root@first-deployment-79956f5c54-jnkmb:/# exit
student1@ip-172-31-3-110:~$ kubectl exec -ti first-deployment-79956f5c54-xxvjv bash
root@first-deployment-79956f5c54-xxvjv:/# echo "Pod3" > /usr/share/nginx/html/index.html
root@first-deployment-79956f5c54-xxvjv:/# exit
exit
student1@ip-172-31-3-110:~$ kubectl exec -ti simple-pod bash
root@simple-pod:/# curl nginx
Pod3
root@simple-pod:/# curl nginx
Pod1
root@simple-pod:/# curl nginx
Pod2
```

There is another way to achieve this result with methods that are in the good practices of the immutable infrastructure concepts. But for the sake of convenience, and for the demonstration, it is enough.

## Ingress

Now, our application is in place and is in HA, and it has an internal load-balancer to the front cluster. But our service only allows us to have an IP address internal to the cluster. We must now expose our service outside the cluster.

Our cluster already has an Ingress controller that runs like DaemonSet on every worker.

```
student1@ip-172-31-3-110:~$ kubectl get pods -n ingress-nginx
NAME                                    READY   STATUS    RESTARTS   AGE
default-http-backend-797c5bc547-l4jxf   1/1     Running   0          5h
nginx-ingress-controller-jrgb4          1/1     Running   0          5h
nginx-ingress-controller-z99gr          1/1     Running   0          5h
student1@ip-172-31-3-110:~$ kubectl get nodes
NAME               STATUS   ROLES               AGE   VERSION
student1-master1   Ready    controlplane,etcd   4h    v1.11.3
student1-worker1   Ready    worker              4h    v1.11.3
student1-worker2   Ready    worker              4h    v1.11.3
student1@ip-172-31-3-110:~$ kubectl describe node student1-worker1 |grep -i ip
                    flannel.alpha.coreos.com/public-ip: 172.31.1.26
                    rke.cattle.io/external-ip: 18.195.213.49
                    rke.cattle.io/internal-ip: 172.31.1.26
  InternalIP:  172.31.1.26
student1@ip-172-31-3-110:~$ curl 172.31.1.26
default backend - 404
```

The Ingress controller's role is to direct traffic to the right service. If it receives a request for foo.example.com, it directs it either to the service that corresponds to its request, and in the case where it does not exist, to the default backend that produces, as we have just seen , a 404 error.

Then create an `ingress.yaml` file to copy the following content:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
```

```
nano ingress.yaml
Crtl + x
Y
enter
```

```
student1@ip-172-31-3-110:~$ kubectl apply -f ingress.yaml
ingress.extensions/my-ingress created
student1@ip-172-31-3-110:~$ kubectl get ing
NAME         HOSTS                ADDRESS                     PORTS   AGE
my-ingress   nginx.example.com   18.195.213.49,54.93.36.34   80      5s
```

When you list the Ingress, you see 2 IP addresses associated with it. These are the public addresses of the workers.

```
student1@ip-172-31-3-110:~$ curl -H 'Host: nginx.example.com' 18.195.213.49
Pod1
student1@ip-172-31-3-110:~$ curl -H 'Host: nginx.example.com' 54.93.36.34
Pod3
```

Now, when you `curl` to one of these addresses by specifying the host name you want to reach, you are redirected to the service, which in turn will redirect traffic to one of our 3 Pods.

## Labels and Selector

You think it seems to work a little bit magically. How a deployment knows the Pods it must handle, a service the Pods to which to direct the traffic. Thanks to labels and selectors:

```
student1@ip-172-31-3-110:~$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
first-deployment-79956f5c54-hsfpj   1/1     Running   0          1h
first-deployment-79956f5c54-jnkmb   1/1     Running   0          1h
first-deployment-79956f5c54-xxvjv   1/1     Running   0          1h
simple-pod                          1/1     Running   0          3h
student1@ip-172-31-3-110:~$ kubectl describe po/first-deployment-79956f5c54-hsfpj|grep -i labels
Labels:             app=first-deployment
```

Our Pod has a `app = first-deployment` label. This means that the Deployment as well as the Service must have a `app = first-deployment` selector that allows them to identify it:

```
student1@ip-172-31-3-110:~$ kubectl describe deploy/first-deployment |grep -i selector
Selector:               app=first-deployment
student1@ip-172-31-3-110:~$ kubectl describe svc/nginx |grep -i selector
Selector:          app=first-deployment
```

But we will see the labels and the selectors a little further, when we have also seen the taints and the tolerations.

## Conclusion

We have just skimmed the layers of abstraction that Kubernetes puts in place inside the cluster to make your applications highly resilient and easy to maintain. Of course, this is just an exposition of the surface, there is much more to discover!