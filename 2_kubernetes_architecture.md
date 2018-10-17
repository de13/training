# Kubernetes architecture

Il est important de pouvoir identifier les différents composants d'architecture, afin de pouvoir identifier les éventuels problèmes qui peuvent survenir.

Sur votre poste de travail, vous avez `kubectl`, la commande qui vous permet d'interagir avec L'API Kubernetes, configuré (la configuration se trouve dans `~/.kube/config`.

Identifions d'abord les composants :

```
student1@ip-172-31-3-110:~$ kubectl get componentstatus
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
```

Le fait que nous ayons une réponse montrer que l'API est fonctionnelle. Que nous retourne cette commande ? L'état des composant master : scheduler, controller-manager et etcd. Nous pouvons constater ici qu'il sont tous healthy et fonctionnels.

Le control plane est donc opérationel. Passons au data plane :

```
student1@ip-172-31-3-110:~$ kubectl get nodes
NAME               STATUS   ROLES               AGE   VERSION
student1-master1   Ready    controlplane,etcd   8m    v1.11.3
student1-worker1   Ready    worker              7m    v1.11.3
student1-worker2   Ready    worker              7m    v1.11.3
```

Nous avons deux workers ready, sur lesquels le kubelet est installé en v1.11.3.

A ce stade nous pouvons en savoir un peu plus sur les nodes, leur capacité, ce qu'ils contiennent avec :

```
student1@ip-172-31-3-110:~$ kubectl get nodes student1-worker1 -o yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"06:40:b4:bf:aa:09"}'
    flannel.alpha.coreos.com/backend-type: vxlan
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    flannel.alpha.coreos.com/public-ip: 172.31.1.26
    node.alpha.kubernetes.io/ttl: "0"
    rke.cattle.io/external-ip: 18.195.213.49
    rke.cattle.io/internal-ip: 172.31.1.26
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: 2018-10-17T09:08:01Z
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    cattle.io/creator: norman
    kubernetes.io/hostname: student1-worker1
    node-role.kubernetes.io/worker: "true"
  name: student1-worker1
  resourceVersion: "10917"
  selfLink: /api/v1/nodes/student1-worker1
  uid: 25fb648a-d1ec-11e8-a112-02b0490a9828
spec:
  podCIDR: 10.42.1.0/24
status:
  addresses:
  - address: 172.31.1.26
    type: InternalIP
  - address: student1-worker1
    type: Hostname
  allocatable:
    cpu: "2"
    ephemeral-storage: "14927638094"
    hugepages-2Mi: "0"
    memory: 3942692Ki
    pods: "110"
...
  images:
  - names:
    - rancher/hyperkube@sha256:fec54399a611d533e27e97085936cd08d8696af44936f365c1757f0a57ca123e
    - rancher/hyperkube:v1.11.3-rancher1
    sizeBytes: 957542224
  - names:
    - rancher/nginx-ingress-controller@sha256:90841873aaa8079f51d4ed2be668ba48c403b2b1d484bad1061155af4443d872
    - rancher/nginx-ingress-controller:0.16.2-rancher1
    sizeBytes: 364115902
...
  nodeInfo:
    architecture: amd64
    bootID: 82a5dba3-8cc2-4363-82af-e1c8dcc53c91
    containerRuntimeVersion: docker://17.3.2
    kernelVersion: 4.4.0-1052-aws
    kubeProxyVersion: v1.11.3
    kubeletVersion: v1.11.3
    machineID: 8c859f2bde9e41155bca40d2d85aebb5
    operatingSystem: linux
    osImage: Ubuntu 16.04.4 LTS
    systemUUID: EC260DD6-CB50-D801-7118-E6ADB059E5D7
```