# Probes

For this lab, [de13 / spaceboy] (https://hub.docker.com/r/de13/spaceboy/) can test the readiness and liveness of a Pod. The Pod is ready 30 seconds after it's started by default, and stays healthy for 2 minutes.

```
apiVersion: v1
kind: Service
metadata:
  name: liveness
  labels:
    run: liveness
spec:
  ports:
  - protocol: TCP
    port: 8080
    nodePort: 30080
  type: NodePort
  selector:
    run: liveness
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: liveness
  name: liveness
spec:
  containers:
  - name: liveness
    image: de13/spaceboy:v2
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
```

We can do a curl on healthz, but Kubernetes does it for us.

```
admin@ip-172-31-3-110:~/resources$ curl -i 172.31.1.129:30080/healthz
HTTP/1.1 200 OK
Date: Mon, 22 Oct 2018 08:55:05 GMT
Content-Type: text/html; charset=utf-8
Transfer-Encoding: chunked

<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <title>Liveliness</title>
</head>
kubectl exec -i check curl liveness:8080/ready
```

As soon as the Pod is no longer healthy, as a health check has been set up, the Pod is automatically restarted.

```
admin@ip-172-31-3-110:~/resources$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
liveness                 1/1     Running   2          7m
admin@ip-172-31-3-110:~/resources$ kubectl get po liveness -w
NAME       READY   STATUS    RESTARTS   AGE
liveness   1/1     Running   3          7m33s
liveness   1/1   Running   4     9m54s
```

We can also look at the readiness:

```
admin@ip-172-31-3-110:~/resources$ curl -i 172.31.1.129:30080/ready
HTTP/1.1 503 OK
Date: Mon, 22 Oct 2018 08:54:57 GMT
Content-Length: 180
Content-Type: text/html; charset=utf-8

<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <title>Readiness</title>
  </head>
  <body>
    <p>Almost here...</p>
  </body>
</html>
admin@ip-172-31-3-110:~/resources$ curl -i 172.31.1.129:30080/ready
HTTP/1.1 200 OK
Date: Mon, 22 Oct 2018 08:56:37 GMT
Content-Length: 172
Content-Type: text/html; charset=utf-8

<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <title>Readiness</title>
  </head>
  <body>
    <p>Ready!</p>
  </body>
</html>
```

During the first 30 seconds, the Pod returns a 503 error, then a 200 error. The traffic is routed from the service to the Pod only after this initial delay has been completed.

## Conclusion

Whatever application you create, always consider having health checks so that in case of a problem, the environment can automatically respond to it, rather than generating additional alarms and operations.

Liveness is very useful for applications that take a long time to start, do not hesitate to abuse it!