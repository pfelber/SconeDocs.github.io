
## Starting a SCONE Application on a Swarm

```bash
 docker service create --name "ha" --detach=true -p 80:80 -p 443:443 --constraint 'node.labels.sgx != 0'  sconecuratedimages/nginx:noshielding
```
If this fails, you get a status as follows: 

```bash
> docker service ps ha
ID                  NAME                IMAGE                      NODE                DESIRED STATE       CURRENT STATE           ERROR                       PORTS
f65id6ow5n6w        ha.1                sconecuratedimages/nginx   beatrix             Ready               Ready 1 second ago                                  
jt6wj5e3lso4         \_ ha.1            sconecuratedimages/nginx   beatrix             Shutdown            Failed 3 seconds ago    "task: non-zero exit (1)"   
sspou3mcis8m         \_ ha.1            sconecuratedimages/nginx   beatrix             Shutdown            Failed 9 seconds ago    "task: non-zero exit (1)"   
p3bw780pu63b         \_ ha.1            sconecuratedimages/nginx   beatrix             Shutdown            Failed 15 seconds ago   "task: non-zero exit (1)"   
75zjsesil5k4         \_ ha.1            sconecuratedimages/nginx   beatrix             Shutdown            Failed 22 seconds ago   "task: non-zero exit (1)"   
```

Reasons for failures might be that you do have access to the sgx device. Did you indeed install the patched docker version? Did you indeed label the nodes correctly?

In case service correctly starts up, you will see a status like this: 
```bash
> docker service ps ha
docker service ps nginx_service
ID                  NAME       IMAGE                                  NODE                DESIRED STATE       CURRENT STATE            ERROR                              PORTS
x2xq1c3aede7        ha.1       sconecuratedimages/nginx:noshielding   alice               Running             Running 8 minutes ago         
```

## Stopping the service

Stop the service via:

```bash
docker service rm ha
```


## Scalable Fault-Tolerant Service

We start a nginx service this time with the scontain.com website. We start two replicas, each
of these need to run on  a SGXv1 machines:

```bash
> docker service create --name "sconeweb" --detach=true  --publish 80:80 --publish 443:443 --constraint 'node.labels.sgx == 1'  --replicas=2 sconecuratedimages/sconetainer:noshielding
```

If the image is not available on one of the nodes, you might see the following status:

```bash
> docker service ps sconeweb
ID                  NAME                IMAGE                                        NODE                DESIRED STATE       CURRENT STATE          ERROR                              PORTS
o79714pw2fpn        sconeweb.1          sconecuratedimages/sconetainer:noshielding   alice               Running             Running 4 hours ago                                       
t0byepte0fzj         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   beatrix             Shutdown            Rejected 4 hours ago   "No such image: sconecuratedim…"   
mg4xdq868syq         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   beatrix             Shutdown            Rejected 4 hours ago   "No such image: sconecuratedim…"   
ry1pqen9jgan         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   beatrix             Shutdown            Rejected 4 hours ago   "No such image: sconecuratedim…"   
q05ti7gkxc7r         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   beatrix             Shutdown            Rejected 4 hours ago   "No such image: sconecuratedim…"   
zxj74inh2zdf        sconeweb.2          sconecuratedimages/sconetainer:noshielding   alice               Running             Running 4 hours ago                                       
```

For the next steps, make sure that all nodes have access to image *sconecuratedimages/sconetainer:noshielding* 
by executing on all nodes:

```bash
docker pull sconecuratedimages/sconetainer:noshielding
```

## Draining a node

To be able to drain all containers from a node, we need to figure out the node's id. We can do this manually by executing the following command:

```bash
> docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
91a1vvex4dgozfrzy1y136gmg *   alice               Ready               Active              Leader
jhrayos9ylu02egwvkxpqtbwb     beatrix             Ready               Active              
```

You can now take node *alice* out of service by executing:
```bash
> docker node update --availability drain 91a1vvex4dgozfrzy1y136gmg
```

The status of service might now be as follows:

```bash
> docker service ps sconeweb
ID                  NAME                IMAGE                                        NODE                DESIRED STATE       CURRENT STATE            ERROR                              PORTS
qaekdszhm8ua        sconeweb.1          sconecuratedimages/sconetainer:noshielding   beatrix             Running             Running 5 seconds ago                                       
o79714pw2fpn         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   alice               Shutdown            Shutdown 7 seconds ago                                      
t0byepte0fzj         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   beatrix             Shutdown            Rejected 4 hours ago     "No such image: sconecuratedim…"   
mg4xdq868syq         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   beatrix             Shutdown            Rejected 4 hours ago     "No such image: sconecuratedim…"   
ry1pqen9jgan         \_ sconeweb.1      sconecuratedimages/sconetainer:noshielding   beatrix             Shutdown            Rejected 4 hours ago     "No such image: sconecuratedim…"   
kheb76bceupz        sconeweb.2          sconecuratedimages/sconetainer:noshielding   beatrix             Running             Running 6 seconds ago                                       
zxj74inh2zdf         \_ sconeweb.2      sconecuratedimages/sconetainer:noshielding   alice               Shutdown            Shutdown 7 seconds ago       
```

To put the node back in service, execute:

```bash
docker node update --availability active 91a1vvex4dgozfrzy1y136gmg
```


## Running your own registry

In the above example, we have seen that we had to pull images explicitly from docker hub on each node. This is of course not satisfactory solution. Instead, we should run our own local registry with a copy of the image. You can do this as follows: 

```bash
docker service create --name registry --detach=true  --publish 5000:5000 --publish 443:443 --replicas=1 registry
```

Now the our local registry is running, we need to pull the images that we want to store in this registry first. Then, we tag the pulled image with the name of our local registry:

```bash
docker pull sconecuratedimages/sconetainer:noshielding
docker tag sconecuratedimages/sconetainer:noshielding localhost:5000/sconetainer
```

We then push the tagged image to our local registry:
```bash
docker push localhost:5000/sconetainer
```

We can now create a service using the image stored in the local registry:

```bash
docker service create --name "ha" --detach=true  --publish 9080:80 --publish 9443:9443 --constraint 'node.labels.sgx != 0'  --replicas=2 localhost:5000/sconetainer
```

## Updating the image of a service

Say, there is a new version of the sconetainer image available. We can update this image in our local registry
as follows:

```bash
docker pull sconecuratedimages/sconetainer:noshielding
docker tag sconecuratedimages/sconetainer:noshielding localhost:5000/sconetainer
docker push localhost:5000/sconetainer
```

We can now update the service as follows:

```bash
docker service update --image  localhost:5000/sconetainer ha
```

Author: Christof Fetzer,  2017
