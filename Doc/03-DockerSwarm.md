# 03 - Docker Swarm

A bunch of containers running on single hardware is a nice thing. But what if more resiliency is required ? 

It is time for some container orchestration tool - lets start with Swarm.
And this time - to keep things simple - I will use [Play with Docker](https://labs.play-with-docker.com/) page


## Initialize Swarm 

First I had to add three instances. Now I had 3 nodes - the numbers are just to distinguish nodes.
I decided to make node1 **the manager** and I initialized cluster on it with:
```
docker swarm init --advertise-addr eth0
```
![initialize cluster](./images/L03-001-swarm-init.jpg)

Next step - connect worker nodes.

The structure of command is like this:
```
docker swarm join-token manager
```
But Swarm was kind enough to provide me with command necessary to connect remaining two host to the cluster. So in my case it was 
```
docker swarm join --token SWMTKN-1-20xhdg7e9q9d671vwa71otqfc43vtl7eq3yekvxp7lxqcd16yx-625ezr2nrvqu2x2bkvg2lire7 192.168.0.8:2377
```
![join swarm](./images/L03-002-join-swarm.jpg)

I run the command on both node2 and node3

And BTW - if you are like me and use Firefox, you provably struggle to copy / paste command from one [Play with Docker](https://labs.play-with-docker.com/) instance to another. Ctrl-c and Ctrl-V does not work and you do not want to type so long tokens. There is a solution ! :smile: Just use **ctrl-Insert** for copy and **shift-insert** to paste.



Now I can list all the nodes with 
```
docker node ls
```
![swarm nodes](./images/L03-003-swarm-cluster.jpg)

The asterisk (*) next to the ID of the node represents the node that handled that specific command (*docker node ls* in this case).


## First service

Now when I have a cluster I can deploy first service. The idea is pretty straightforward - simple nginx 1.12 that will run in detached mode. "The *--mount* flag is useful to have NGINX print out the hostname of the node it's running on. " - as stated in the lab manual :wink:
The interesting part is the *--publish* command that uses the Swarm build in routing mesh. Port 80 is exposed on every node. Requests coming to port 80 will be routed to nodes where nginx container is running.

```
docker service create --detach=true --name nginx1 --publish 80:80  --mount source=/etc/hostname,target=/usr/share/nginx/html/index.html,type=bind,ro nginx:1.12
```
![first service](images/L03-004-first-service.jpg)


To inspect the service I can use
```
docker service ls
```
![service is running](images/L03-005-service-running.jpg)


To have even more info about specific service I can run 
```
docker service ps nginx1
```
![task on node](images/L03-006-service-on-node.jpg)


To test if sevice in did is running (and can answer on port 80) I can run

```
curl localhost:80
```
![curl response](images/L03-007-curl-response.jpg)



## Scale your service

Till now the effect of using swarm was not much better than use of a single docker container - a single nginx.

Time to change it. Let's update the service to 5 replicas !

```
docker service update --replicas=5 --detach=true nginx1
```
![more instances](images/L03-008-more-instances.jpg)

Now when I run 

```
docker service ps nginx1
```
I can see that nginx tasks are spread across existing nodes
![service log](images/L03-009-service-on-nodes.jpg)

If I run
```
curl localhost:80
``` 
few times in a row, I can see that the answer comes from different nodes
![routing mesh](images/L03-010-load-ballancing.jpg)


The same I can check with 
```
docker service logs nginx1
```
In did requests were served by different containers. E.g. nginx1.5 means that container 5 (located on node 2) answered request
![log](images/L03-011-service-log.jpg)



## Apply rolling updates

How to update service ? 

Easily with 
```
docker service update --image nginx:1.13 --detach=true nginx1
```
![start the update](images/L03-012-start-update.jpg)

I theory if one is quick enough it is possible to "see" rolling update process. I was to slow and saw all containers updates
![rolling update](images/L03-013-rolling-update.jpg)

Oh, well - I can always run another update
```
docker service update --image nginx:1.20.0 --detach=true nginx1
```
This time I was able to witness the update on some nodes
![next update](images/L03-014-another-update.jpg)


## Reconcile problems with containers

But what happen if something bad happen to one of the nodes ? In this lab I have only one master node, so I cannot afford to lose it, but with the worker node - thats a different story
After all - Swarm is a declarative system. It should cope with this kind of problem and ensure that number of service instances remain constant.

Lets try it.

To have a fresh view - I start with new service. I have to change port on which my service is available -routing mesh is able to publish only one service on each port

```
docker service create --detach=true --name nginx2 --replicas=5 --publish 81:80  --mount source=/etc/hostname,target=/usr/share/nginx/html/index.html,type=bind,ro nginx:1.12
```
![new service](images/L03-015-new-service.jpg)


Now on node1 i run
```
watch -n 1 docker service ps nginx2
```
to see what happen if I remove one node
![watch service](images/L03-016-watch.jpg)


Now on node3 I execute
```
docker swarm leave
```
![swarm leave](images/L03-017-leave-swarm.jpg)

Now Swarm runs new containers on existing nodes to keep service running in five containers. This can be checked with 
```
docker service ps nginx2
```

![reorganization](images/L03-018-reorganization.jpg)

