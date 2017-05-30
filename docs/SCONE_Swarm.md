# SCONE SWARM

## Installation

Our [SCONE Host Setup](SCONE_HOSTINSTALLER_README.md) script will typically set up a docker swarm. You can skip this section, if you have already a swarm running. 

To manually set up a Docker swarm, log into designated manage node and execute:

```bash
export MANAGER_IP=`hostname --ip-address`
docker swarm init --advertise-addr $MANAGER_IP
```

We determine the address of the swarm manager and the token to join the swarm as follows

```bash
export manager_addr=`docker info --format '{{(index .Swarm.RemoteManagers 0).Addr}}'`
export token=`docker swarm join-token -q worker`
echo export token=$token
echo export manager_addr=$manager_addr
```

On each node that should join the swarm, ensure that you installed our patched Docker engine (which makes the isgx device available to all containers).

Add a new node to the swarm by setting *$token*  and *$manager_addr* to the values determined above and
execute on the new node:

```bash
docker swarm join --token $token $manager_addr
```

## Setting Node Labels

Docker Swarm supports the labeling of nodes. In our case, we define a label *"sgx"* that permits us to determine if a node supports sgx and what version of sgx. 

We define the following node labels:

* sgx == "0" iff the node does not support sgx

* sgx == "1" iff the node supports sgx version 1

* sgx == "2" iff the node supports sgx version 2 (which is not yet available)


To assign a label, we need to figure out the node ID. An easy way to do so, is via the following command:

```bash
> docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
91a1vvex4dgozfrzy1y136gmg *   alice               Ready               Active              Leader
ocwyg135npgpsrpe2ncuw542x     beatrix             Ready               Active              

```
One can now manually update a label as follows:

```bash
> docker node update --label-add sgx="1" 91a1vvex4dgozfrzy1y136gmg
91a1vvex4dgozfrzy1y136gmg
```
For large clusters, we can automate the labeling
as described in the next subsection.

## Assign labels to all nodes

Docker swarm keeps listing nodes nodes that are down - even if the host running the node has in meantime rejoined as a new node.
 
To purge all down nodes, execute the following:

```bash
docker node rm $(docker node ls -q)
```

Assuming that all nodes in your swarm support sgx, we assign SGX label "1" to all (remaining) nodes:

```bash
export nodes=$(docker node ls -q)
for node in $nodes ; do
	docker node update --label-add sgx="1" $node
done
```

We can check that all sgx labels are "1" by executing the following:

```bash
docker inspect --format '{{(index  .Spec.Labels "sgx")}}' $nodes
```

Author: Christof Fetzer, 2017
