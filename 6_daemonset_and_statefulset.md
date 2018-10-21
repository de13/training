# DaemonSet and StatefulSet

In this lab we will see two variants of deployments, DaemonSet and StatefulSet.

## DaemonSet (ds)

The most typical use case of DaemonSet, is the "agent" (eg `logstash` or` fluentd`) that we want to place on each host:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: agent
  name: agent
spec:
  selector:
    matchLabels:
      app: agent
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
      - image: registry.hub.docker.com/library/nginx:1.15.5
        name: nginx
      tolerations:
      - key: "gpu"
        operator: "Equal"
        value: "yes"
        effect: "NoSchedule"
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f daemon.yaml
daemonset.apps/agent created
admin@ip-172-31-3-110:~/resources$ kubectl get ds
NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
agent   2         2         2       2            2           <none>          38s
admin@ip-172-31-3-110:~/resources$ kubectl get po -o wide |grep agent
agent-lcc64            1/1     Running   0          44s   10.42.1.29   worker1   <none>
agent-q9qd2            1/1     Running   0          44s   10.42.2.17   worker2   <none>
```

**Warning**: As the previous example shows, labels, affinities and taints also affect DaemonSet. Since we had placed a taint on one of us workers in the previous example, our agent needs a tolerance to run on all nodes.

In the same way, you might want an agent to run only on some nodes, but not all. For this you can use the selectors, or the affinities:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: agent-lux
  name: agent-lux
spec:
  selector:
    matchLabels:
      app: agent-lux
  template:
    metadata:
      labels:
        app: agent-lux
    spec:
      nodeSelector:
        dc: luxembourg
      containers:
      - image: registry.hub.docker.com/library/nginx:1.15.5
        name: nginx
      tolerations:
      - key: "gpu"
        operator: "Equal"
        value: "yes"
        effect: "NoSchedule"
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f agent.yaml
daemonset.apps/agent-lux created
admin@ip-172-31-3-110:~/resources$ kubectl get ds/agent-lux
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
agent-lux   1         1         1       1            1           dc=luxembourg   14s
admin@ip-172-31-3-110:~/resources$ kubectl get pod -o wide |grep agent-lux
agent-lux-dmqx7        1/1     Running   0          33s   10.42.1.30   worker1   <none>
admin@ip-172-31-3-110:~/resources$ kubectl get node/worker1 --show-labels
NAME      STATUS   ROLES    AGE   VERSION   LABELS
worker1   Ready    worker   2d    v1.11.3   arch=x86,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,cattle.io/creator=norman,dc=luxembourg,kubernetes.io/hostname=worker1,node-role.kubernetes.io/worker=true,proc=gpu
```

## StatefulSet (sts)

In a completely different way, and as its name implies, the StatefulSet is a type of deployment specific to stateful applications. The StatefulSet guarantees us two things:

- A boot order of the Pods which is always the same
- The persistence of hostnames

We will create an instance of MongoDB. First of all, we need a headless service. What is a headless service? This is a service without IP address, which only serves to give a name resolution to our Pods.

And in order to see a little more clearly, why not also put into practice a concept that we saw a little earlier, the namespaces?

```
admin@ip-172-31-3-110:~/resources$ kubectl create ns mongo
namespace/mongo created
```

```
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    run: mongo
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f mongo-svc.yaml
service/mongo created
admin@ip-172-31-3-110:~/resources$ kubectl get svc -n mongo
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
mongo   ClusterIP   None         <none>        27017/TCP   12s
```

We now have to create the three replicas of Mongo and configure them by hand:

```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
  namespace: mongo
spec:
  serviceName: "mongo"
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
```

```
admin@ip-172-31-3-110:~/resources$ kubectl apply -f mongo.yaml
statefulset.apps/mongo created
admin@ip-172-31-3-110:~/resources$ kubectl get sts -n mongo
NAME    DESIRED   CURRENT   AGE
mongo   3         3         7m
dmin@ip-172-31-3-110:~/resources$ kubectl get pods -n mongo
NAME      READY   STATUS    RESTARTS   AGE
mongo-0   1/1     Running   0          7m
mongo-1   1/1     Running   0          7m
mongo-2   1/1     Running   0          6m
admin@ip-172-31-3-110:~/resources$ kubectl -n mongo exec -ti mongo-0 bash
root@mongo-0:/# mongo
> > rs.initiate( {
...    _id : "rs0",
...    members: [
...       { _id: 0, host: "mongo-0.mongo:27017" },
...       { _id: 1, host: "mongo-1.mongo:27017" },
...       { _id: 2, host: "mongo-2.mongo:27017" }
...    ]
... })
{ "ok" : 1 }
rs0:OTHER> rs.status()
{
	"set" : "rs0",
	"date" : ISODate("2018-10-21T19:36:49.397Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1540150604, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1540150604, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1540150604, 1),
			"t" : NumberLong(1)
		}
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "mongo-0.mongo:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 57,
			"optime" : {
				"ts" : Timestamp(1540150604, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-10-21T19:36:44Z"),
			"infoMessage" : "could not find member to sync from",
			"electionTime" : Timestamp(1540150603, 1),
			"electionDate" : ISODate("2018-10-21T19:36:43Z"),
			"configVersion" : 1,
			"self" : true
		},
		{
			"_id" : 1,
			"name" : "mongo-1.mongo:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 16,
			"optime" : {
				"ts" : Timestamp(1540150604, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1540150604, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-10-21T19:36:44Z"),
			"optimeDurableDate" : ISODate("2018-10-21T19:36:44Z"),
			"lastHeartbeat" : ISODate("2018-10-21T19:36:47.864Z"),
			"lastHeartbeatRecv" : ISODate("2018-10-21T19:36:45.313Z"),
			"pingMs" : NumberLong(0),
			"syncingTo" : "mongo-0.mongo:27017",
			"configVersion" : 1
		},
		{
			"_id" : 2,
			"name" : "mongo-2.mongo:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 16,
			"optime" : {
				"ts" : Timestamp(1540150604, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1540150604, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-10-21T19:36:44Z"),
			"optimeDurableDate" : ISODate("2018-10-21T19:36:44Z"),
			"lastHeartbeat" : ISODate("2018-10-21T19:36:47.861Z"),
			"lastHeartbeatRecv" : ISODate("2018-10-21T19:36:45.335Z"),
			"pingMs" : NumberLong(0),
			"syncingTo" : "mongo-0.mongo:27017",
			"configVersion" : 1
		}
	],
	"ok" : 1
}
rs0:PRIMARY>
```

Fortunately, there are other, much more automated ways to create and maintain complex workloads under Kubernetes. These are the Operators. Get more information here: [https://coreos.com/operators/](https://coreos.com/operators/)

## Conclusion

It is now possible to maintain any workload on Kubernetes, from simple stateless to complex stateful. Due to the unique nature of Kubernetes, stateful workloads are even first class citizens of Kubernetes.  
