# Namespaces, configMaps and Secrets

## Namespaces

Namespaces are used to logically isolate one application (or application group) from another.

```
student1@ip-172-31-3-110:~$ kubectl get ns
NAME            STATUS   AGE
cattle-system   Active   5h
default         Active   5h
ingress-nginx   Active   5h
kube-public     Active   5h
kube-system     Active   5h
```

We can see that before starting, a certain number of namespace are already present, including 3 that you will find on all Kubernetes clusters:

- default: which is the default namespace if you do not specify any namespace for your resource
- kube-public: which is a common namespace whose resources can be accessed by anyone
- kube-system: which is reserved for the applications needed by the cluster

Let's create our first namespace:

```
student1@ip-172-31-3-110:~$ kubectl create ns baz
namespace/baz created
student1@ip-172-31-3-110:~$ kubectl create deployment nginx --image=nginx:1.14.0 --namespace baz
deployment.apps/nginx created
student1@ip-172-31-3-110:~$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
first-deployment-79956f5c54-hsfpj   1/1     Running   0          1h
first-deployment-79956f5c54-jnkmb   1/1     Running   0          1h
first-deployment-79956f5c54-xxvjv   1/1     Running   0          1h
simple-pod                          1/1     Running   0          3h
student1@ip-172-31-3-110:~$ kubectl get pods -n baz
NAME                    READY   STATUS    RESTARTS   AGE
nginx-957f5f978-xkh85   1/1     Running   0          8s
```

When we do not specify a namespace with `get pod` we list the Pods of the Default namespace. We do not see Pods from other namespaces. When we specify the namespace, we only see the Pods belonging to this namespace.

We will see in another Lab that namespaces are not just about that, and that they bring many other features.

## ConfigMaps

The 12 Factor Apps, the immutable infrastructure, two concepts that work perfectly well with each other.

We have not said enough, but your containers are made to "travel" from one environment to another ** without modification **. Thus, an application created and tested in a DEV environment then QUA, will be found in production, reducing the risk of bug or human error.

However, to do this, it would be necessary that my application can know the connection string of my DB which is not the same in DEV as in PROD, as well as the credentials!

This is where The 12 Factor Apps comes into play: an application must pull this information from its environment, and must not embed this information in its code (which besides being error prone, is an obvious security flaw).

In Kubernetes, we have two resources for these environment variables: ConfigMaps, where we can put the endpoint of our DB, and Secrets, for more sensitive information, such as our credentials.

```
student1@ip-172-31-3-110:~$ kubectl create configmap db-endpoint --from-literal=endpoint=db.example.com:5432 -n baz
configmap/db-endpoint created
student1@ip-172-31-3-110:~$ kubectl describe configmap/db-endpoint -n baz
Name:         db-endpoint
Namespace:    baz
Labels:       <none>
Annotations:  <none>

Data
====
endpoint:
----
db.example.com:5432
Events:  <none>
```

ConfigMaps are a game of key=value. It is **local** to the namespace.

We have two ways to exploit it, either as an environment variable or as a file in a mount.

### Environment variable

Créons `pod1.yaml`:

```
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: baz
spec:
  containers:
    - name: nginx
      image: nginx
      env:
        # Define the environment variable
        - name: ENDPOINT
          valueFrom:
            configMapKeyRef:
              name: db-endpoint
              key: endpoint
```

```
student1@ip-172-31-3-110:~$ kubectl apply -f pod1.yaml
pod/pod1 created
student1@ip-172-31-3-110:~$ kubectl -n baz exec -ti pod1 bash
root@pod1:/# echo $ENDPOINT
db.example.com:5432
```

### Mountpoint

```
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: baz
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - name: endpoint
        mountPath: /tmp
  volumes:
    - name: endpoint
      configMap:
        name: db-endpoint
```

```
student1@ip-172-31-3-110:~$ kubectl apply -f pod2.yaml
pod/pod2 created
student1@ip-172-31-3-110:~$ kubectl -n baz exec -ti pod2 bash
root@pod2:/# ls /tmp/
endpoint
root@pod2:/# cat /tmp/endpoint
db.example.com:5432
```

It is now easy to imagine how the same ConfigMap (eg db-endpoint) may contain different information in a DEV or PROD environment, and how the container will simply consume this information in code change.

## Secrets

Some information is sensitive, such as certificates, tokens, or credentials. In order to protect them, we use Secrets instead of ConfigMaps. However, Secrets are only base64 encoded information, and we see it coming Aha! when are we going to tell you: "How is this safer?"

By default, it is not, since it is easy to decode base64. However, the secrets have been designed to use different backends, and when it comes time to really secure them, you may consider using a software such as Hashicorp Vault.

Let's create our Secrets:

```
student1@ip-172-31-3-110:~$ echo -n 'user1234' > user.txt
student1@ip-172-31-3-110:~$ echo -n 'pass1234' > password.txt
student1@ip-172-31-3-110:~$ kubectl create secret generic db-user --from-file=user.txt --from-file=password.txt -n baz
secret/db-user created
student1@ip-172-31-3-110:~$ kubectl get secret -n baz
NAME                  TYPE                                  DATA   AGE
db-user               Opaque                                2      40s
default-token-w7t8n   kubernetes.io/service-account-token   3      4h
student1@ip-172-31-3-110:~$ kubectl get secret db-user -n baz -o yaml
apiVersion: v1
data:
  password.txt: cGFzczEyMzQ=
  user.txt: dXNlcjEyMzQ=
kind: Secret
metadata:
  creationTimestamp: 2018-10-17T19:12:37Z
  name: db-user
  namespace: baz
  resourceVersion: "57324"
  selfLink: /api/v1/namespaces/baz/secrets/db-user
  uid: 9c735205-d240-11e8-a112-02b0490a9828
type: Opaque
student1@ip-172-31-3-110:~$ echo -n 'cGFzczEyMzQ='|base64 -d
pass1234
student1@ip-172-31-3-110:~$ echo -n 'dXNlcjEyMzQ='|base64 -d
user1234
```

Like ConfigMaps, Secrets can be consumed in two ways:

### Environment variable

```
apiVersion: v1
kind: Pod
metadata:
  name: pod3
  namespace: baz
spec:
  containers:
    - name: nginx
      image: nginx
      env:
        # Define the environment variable
        - name: USER
          valueFrom:
            secretKeyRef:
              name: db-user
              key: user.txt
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-user
              key: password.txt
```

```
student1@ip-172-31-3-110:~$ kubectl apply -f pod3.yaml
pod/pod3 created
student1@ip-172-31-3-110:~$ kubectl -n baz exec -ti pod3 bash
root@pod3:/# echo $USER
user1234
root@pod3:/# echo $PASSWORD
pass1234
```


### Mountpoint

```
apiVersion: v1
kind: Pod
metadata:
  name: pod4
  namespace: baz
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - name: secret
        mountPath: /var/secret
  volumes:
    - name: secret
      secret:
        secretName: db-user
```

```
student1@ip-172-31-3-110:~$ kubectl apply -f pod4.yaml
pod/pod4 created
student1@ip-172-31-3-110:~$ kubectl -n baz exec -ti pod4 bash
root@pod4:/# ls /var/secret/
password.txt  user.txt
root@pod4:/# cat /var/secret/user.txt
user1234
root@pod4:/# cat /var/secret/password.txt
pass1234
```

## Conclusion

ConfigMaps and secrets keep the information in the environment, avoiding the risk of having differences between applications that are deployed in different environments.

Moreover, since this information is stored in memory, it is updated instantly when a change is made.

Namespaces meanwhile are a good starting point for isolating different application stacks, and again, avoiding human errors.