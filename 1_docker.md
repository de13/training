# Docker

## Run your first app

It's time to launch your first application

```
$ docker run nginx
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
f17d81b4b692: Pull complete
d5c237920c39: Pull complete
a381f92f36de: Pull complete
Digest: sha256:1e195a43d44ae6245fab8d9ee523e652c65791d48542313d7b83585a4824dcca
Status: Downloaded newer image for nginx:latest
^C
```

### What has happened?

- We used Docker to launch our first application (nginx).
- Docker did not find the image locally
- Docker has made a sweater of the image since the Registry
- 3 layers have been downloaded and decompressed
- Docker then executed the runtime of the image in the foreground, displaying the logs in stdout

Did you notice the image tag?

## Let's use some options

### Background

Having the container in foreground is not very useful, let's start by turning it in background with the `-d` option:

```
$ docker run -d nginx
e0dece7bbdf80d426641f78f138a19fda39124d0559f060bc063d2c9a10e4c1d
```

We have in return the hash of the container, which allows us to identify it.

### List the containers

To list the containers of our machine, we use `docker ps`:

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
e0dece7bbdf8        nginx               "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp              friendly_yonath
```

We find our container nginx whose ID starts with the first 12 characters of our hash.

Docker has given him a random name `friendly_yonath` and we can see that he is running.

The containers are not always UP, so we need a command to list also those who are not running, and for that we use `docker ps -a`:

```
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                   PORTS               NAMES
e0dece7bbdf8        nginx               "nginx -g 'daemon of…"   4 minutes ago       Up 4 minutes             80/tcp              friendly_yonath
b505f5206818        nginx               "nginx -g 'daemon of…"   13 hours ago        Exited (0) 6 minutes ago                       tender_leakey
```

We find the container that we had launched in foreground, which is in status `Exited`.

### List the different resources of Docker

The list of containers on the machine is not the only list suscesptible to interest us. For example, we can also see the images:

```
$ docker image list
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
nginx                                   latest              dbfc48660aeb        6 minutes ago        109MB
```

This is the image that serves our container. As you can see, we use the same image for both containers, thanks to CoW, which allows containers that run the same image to have a really tiny footprint on the machine. This is important for optimizing execution time, saving disk space, and having a really large application density.

We can also list the volumes, even if for the moment we do not have any:

```
$ docker volume list
DRIVER              VOLUME NAME
```

A volume is what allows us to store data persistently in Docker.

### Delete resources

Creating resources is very easy, but you must also be able to delete them with the same simplicity. To remove a container, a simple `docker rm` is enough:

```
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                   PORTS               NAMES
e0dece7bbdf8        nginx               "nginx -g 'daemon of…"   14 minutes ago      Up 14 minutes            80/tcp              friendly_yonath
b505f5206818        nginx               "nginx -g 'daemon of…"   14 hours ago        Exited (0) 16 minutes ago                       tender_leakey
$ docker rm b505f5206818
b505f5206818
```

The command has removed the exited container, unsurprisingly. However what would happen if we tried to delete the running container?

```
$ docker rm e0dece7bbdf8
Error response from daemon: You cannot remove a running container e0dece7bbdf80d426641f78f138a19fda39124d0559f060bc063d2c9a10e4c1d. Stop the container before attempting removal or force remove
```

Docker refuses to delete it unless it's stopped, or to force it to be removed.

```
$ docker stop e0dece7bbdf8
e0dece7bbdf8
docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
e0dece7bbdf8        nginx               "nginx -g 'daemon of…"   17 minutes ago      Exited (0) 8 seconds ago                       friendly_yonath
```

Once the container is stopped, it is in exited state, and we know that we can delete it in this state.

```
$ docker start e0dece7bbdf8
e0dece7bbdf8
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
e0dece7bbdf8        nginx               "nginx -g 'daemon of…"   18 minutes ago      Up 2 seconds        80/tcp              friendly_yonath
$ docker rm -f e0dece7bbdf8
e0dece7bbdf8
```

So we restart the container to test the other method, then we force it to be removed.

The image, meanwhile, remained intact:

```
$ docker image list
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
nginx                                   latest              dbfc48660aeb        18 minutes ago        109MB
```

However, we can also choose to delete it with `docker rmi` (rm image):

```
$ docker rmi dbfc48660aeb
Untagged: nginx:latest
Untagged: nginx@sha256:1e195a43d44ae6245fab8d9ee523e652c65791d48542313d7b83585a4824dcca
Deleted: sha256:dbfc48660aeb7ef0ebd74b4a7e0822520aba5416556ee43acb9a6350372e516f
Deleted: sha256:1a34717cf175feab802f74f0edd1c41a811165f6e6af5cddf9b33f9211acde10
Deleted: sha256:df31c4d2dc314417ca1507e7e6ac4e732683a67b5aec725ede170ea7c2ecc99e
Deleted: sha256:237472299760d6726d376385edd9e79c310fe91d794bc9870d038417d448c2d5
```

The image and its different layers is no longer present on the disc. If we need it again, Docker will have to pull it from the registry. This is an action that we can also do manually, to reduce the latency of the next boot of our container:

```
$ docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
f17d81b4b692: Pull complete
d5c237920c39: Pull complete
a381f92f36de: Pull complete
Digest: sha256:b73f527d86e3461fd652f62cf47e7b375196063bbbd503e853af5be16597cb2e
Status: Downloaded newer image for nginx:latest
$ docker image list
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
nginx                                   latest              dbfc48660aeb        23 hours ago        109MB
```

### Expose container resources

Until now we have just been content to launch the application without interacting with it. Since Nginx is a front-end, we need to reach it directly on the network. For this, the most classic method is the port binding:

```
$ docker run -d -p 80:80 nginx
b3771a9e93704ba57605160480d0505f8128f25baadea5ae3780c238ae589004
$ curl localhost
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

We used port 80 of our container to use it on port 80 of our machine.

Do you remember different namespaces? Here is the network namespace that is in play. We could just as well delete this namespace for a container:

```
$ docker run -d --network host nginx
5d0940c3385c5916836e2cf75e2f0c94ddc32637639b7c064034af6b1d85b3de
$ curl localhost
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

This time, we did not use the port binding, but our container no longer has its network isolated from that of the machine, and its port 80 is now directly accessible.

## Interact with the container

Containers are part of a concept of immutable infrastructure, which means that there is very little need to interact with it: it is created, and used as such. At least, in the production phase, because for test and debugging needs, it is totally different.

### Capture the container logs

It is obviously primodial to be able to collect logs:

```
$ docker logs 5d0940c3385c
127.0.0.1 - - [17/Oct/2018:05:45:08 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
```

These are the logs of our nginx container, on which we previously curled.

`docker logs` works like tail, and you can also have a stream of logs via the` -f` option

```
$ docker logs -f 5d0940c3385c
127.0.0.1 - - [17/Oct/2018:05:45:08 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
```

You will be able to see each new log that arrives on the container.

### Enter in the container

To be in interactive mode, we use the `-i` option, most often coupled with the` -t` option which allows us to have a tty attached to the container:

```
docker exec -ti 5d0940c3385c bash
root@docker:/# 
```

Here the hostname is that of our machine, because we have removed the network isolation. Let's go on a new container:

```
$ docker rm -f 5d0940c3385c
5d0940c3385c
$ docker run -d -p 80:80 nginx
f65de9904e77f4d2dd8e62b2bf8d8884b64d3b4db95a03ec9bdb2c9daa490515
$ docker exec -ti f65de9904 bash
root@f65de9904e77:/#
```

Here the hostname is that of our container. As you can see, we use the `docker exec` command and we end up with the command we want to execute in the container, here bash. We could settle for a `ls`:

```
$ docker exec -ti f65de9904 ls
bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr
```

From now on, we have our hands on our container. What happens if we change `index.html`?

```
root@f65de9904e77:~# rm /usr/share/nginx/html/index.html
root@f65de9904e77:~# echo "Hello world!" > /usr/share/nginx/html/index.html
root@f65de9904e77:~# exit
$ curl localhost
Hello world!
```

Note that we deleted the `index.html` file to recreate it. Why ? Because the purpose of the container is to be as light as possible, and for that it does not usually have a text editor, which would weigh down a few megabytes.

Now that we have changed the container, let's launch a new instance of nginx, on port 8080 this time, port 80 being occupied by our first container:

```
$ docker run -d -p 8080:80 nginx
cdaa7d6413b41aab54beb05408343c9c000ce9267013ace6e895952fdeb85fdf
$ curl localhost:8080
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

The image remains unchanged. Do you remember AuFS and different layers? The last is a layer in R / W, and it is specific to each container. If I delete my first container, I lose my changes. In the same way, the modifications that I make on a container does not affect the other containers: it is the immutable infrastructure.

Another example to demonstrate this:

```
root@ cdaa7d6413b4:/# vi
bash: vi: command not found
$ docker exec -ti cdaa7d6413 bash
root@cdaa7d6413b4:/# apt update
root@cdaa7d6413b4:/# apt install -y vim
root@cdaa7d6413b4:/# vi --version
VIM - Vi IMproved 8.0 (2016 Sep 12, compiled Sep 30 2017 18:21:38)
root@cdaa7d6413b4:/# exit
$ docker exec -ti f65de9904e77 bash
root@f65de9904e77:/# vi
bash: vi: command not found
```

### Get information about the container

There may be a number of useful information to get on the container: its network, its volumes, the version of the application it runs, and so on. This information can be obtained with docker inspect:

```
$ docker inspect f65de9904e77
[
    {
        "Id": "f65de9904e77f4d2dd8e62b2bf8d8884b64d3b4db95a03ec9bdb2c9daa490515",
        "Created": "2018-10-17T05:58:58.351112437Z",
        "Path": "nginx",
        "Args": [
            "-g",
            "daemon off;"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 961,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2018-10-17T05:58:58.72728505Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:dbfc48660aeb7ef0ebd74b4a7e0822520aba5416556ee43acb9a6350372e516f",
```

The list of this information is long, and you are welcome to take a look at it when you wish to debug a container.

## Go further than a simple container

### Persistence of data

Containers may be required to store data. For this, the volumes are used:

```
$ docker run -d -v /data nginx
fd1aca6634d59dee58c42f041d79b3d7565305418c8634e05c98fc1eb3ac92d7
$ docker exec -ti fd1aca6634d5 bash
root@fd1aca6634d5:/# cd /data
root@fd1aca6634d5:/data# ls
root@fd1aca6634d5:/data# touch hello
root@fd1aca6634d5:/data# exit
$ docker inspect fd1aca6634d5
"Mounts": [
            {
                "Type": "volume",
                "Name": "7c6ce2ab968593bc711a85c4e6236bfd798d6650ce92151c84f5cdd187fbd774",
                "Source": "/var/lib/docker/volumes/7c6ce2ab968593bc711a85c4e6236bfd798d6650ce92151c84f5cdd187fbd774/_data",
                "Destination": "/data",
$ sudo -i
$ ls /var/lib/docker/volumes/7c6ce2ab968593bc711a85c4e6236bfd798d6650ce92151c84f5cdd187fbd774/_data
hello
```

We created an empty volume called data and mounted to the root of the filesystem of the container. This volume is a local volume, and the data it contains is saved on disk.

```
$ docker rm -f fd1aca6634d5
fd1aca6634d5
$ docker volume list
DRIVER              VOLUME NAME
local               7c6ce2ab968593bc711a85c4e6236bfd798d6650ce92151c84f5cdd187fbd774
```

Deleting the container does not delete the volume and can be used again by a container.

```
$ docker run -d --mount source=7c6ce2ab968593bc711a85c4e6236bfd798d6650ce92151c84f5cdd187fbd774,target=/newdata nginx
fd5ee55c7714e1f267927dbf30c873dc8764484494deb3b50885ce160069d0f0
$ docker exec -ti fd5ee55c771 bash
root@fd5ee55c7714:/# cd newdata
root@fd5ee55c7714:/newdata# ls
hello
root@fd5ee55c7714:/newdata# exit
```

It is also possible to mount existing directories of the local filesystem:

```
$ pwd
/home/admin
$ echo "Hello: world" > config.txt
$ docker run -d -v /home/admin/config.txt:/data/data.conf nginx
beadd99f1896ca7a4758941c18c32cda0554cf91c31c7a4e9e7318edbc305527
$ docker exec -ti beadd99f189 bash
root@beadd99f1896:/# cat /data/data.conf
Hello: world
```

It is possible to have several containers using the same volume:

```
$ docker run -d -v /data --name backend nginx
142d1ec7b671131b6777ca2761a6fb2b01d7f9d0740de3c38189d742310b8b5f
$ docker run -d --volumes-from backend --name frontend nginx
bec24d8e458df8afbf905eed781799d325b896ad63b59d47bffae6d5932c6314
docker exec -ti bec24d8e458d bash
root@bec24d8e458d:/# cd data
root@bec24d8e458d:/data# echo "hello from frontend" > file.txt
root@bec24d8e458d:/data# exit
exit
$ docker exec -ti 142d1ec7b6 bash
root@142d1ec7b671:/# cat data/file.txt
hello from frontend
```

Be careful, here our two containers have write access on the same volume.

```
$ docker run -d --volumes-from backend:ro --name newfrontend nginx
929da2db8099e2f9f643ee7d2a6019c4e506a0f2723c3a7ca85ef0c9fb896a4d
$ docker exec -ti 929da2db809 bash
root@929da2db8099:/# cat data/file.txt
hello from backend
root@929da2db8099:/# echo "hello from newfrontend" > /data/file2.txt
bash: /data/file2.txt: Read-only file system
```

### Little trick to do the housework

The `docker ps -aq` command returns the ID list of all containers. To delete them at once:

```
$ docker ps -aq|xargs docker rm -f
929da2db8099
bec24d8e458d
142d1ec7b671
beadd99f1896
abb473c05d70
fd5ee55c7714
cdaa7d6413b4
f65de9904e77
```

In the same way, `docker volume ls -q` only returns the volume ID:

```
$ docker volume ls -q|xargs docker volume rm
7c6ce2ab968593bc711a85c4e6236bfd798d6650ce92151c84f5cdd187fbd774
e2934ffe837ffe45676cb4ab2af3c58fc1e0a8c35a4964469359ffc70bdefa59
```

Attention, not to do in production obviously!

### Communication between containers

Say you have a Postgresql DB in a container, and a php container that wants to access that DB. It would be unwise in this case to expose port 5432 of the Postgresql container on the machine because only the php container needs to access it. In this case, just link the two containers:

```
$ docker search postgres
NAME                                     DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
postgres                                 The PostgreSQL object-relational database sy…   5618                [OK]
$ docker run -d -v /var/lib/postgresql/data --name db postgres
fad50a2872bd7b470dd50e8302593965e2b011182bb8cf7785a335692fe4c9c4
$ docker run -d -p 80:80 --link db:db nginx
310c7c70ffd5d34a1b2661f785c1030276b4515a2e3dbca3edf625816fbd0b50
$ docker exec -ti 310c7c70ffd bash
root@310c7c70ffd5:/# apt update
root@310c7c70ffd5:/# apt install -y inetutils-ping
root@310c7c70ffd5:/# ping db
PING db (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: icmp_seq=0 ttl=64 time=0.097 ms
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.098 ms
^C
root@310c7c70ffd5:/# apt install -y telnet
root@310c7c70ffd5:/# telnet db 5432
Trying 172.17.0.2...
Connected to db.
Escape character is '^]'.
^CConnection closed by foreign host.
```

## Conclusion

There are so many Docker options that we do not have time to explore here.

Docker is a fabulous tool for containerization, and it allows to quickly move from development to production with the same image.

However, Docker remains very local to the machine on which it is run. This is one of its limits.

In addition, deploying hundreds of containers by hand, with different names to easily identify them, with different ports on the machine, with links between the containers could quickly become very complicated.

Especially since it is not easy to make a rolling upgrade on deployed containers.

With the massive adoption of containers, adding a layer of abstraction to orchestrate them quickly became a necessity.

**Note**: Docker runtime features are now part of the OCI (Open Container Initiative), and are standardized. The Docker runtime, and more specifically containerd, is only one runtime among others, such as cri-o, rkt, gVisor, Kata...