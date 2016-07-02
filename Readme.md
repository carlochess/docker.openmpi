## docker.openmpi

With the code in this repository, you can build a Docker container that provides 
the OpenMPI runtime and tools along with various supporting libaries, 
including the MPI4Py Python bindings. The container also runs an OpenSSH server
so that multiple containers can be linked together and used via `mpirun`.

```
$> container=docker run -d -p 22 carlochess/openmpi
# Check port of container
$> docker inspect $container
# And then, filled on port number, eg. 22
$> ssh -p 22 mpirun@localhost # password mpirun
```


## Start an MPI Container Cluster

While containers can in principle be started manually via `docker run`, we suggest that your use 
[Docker Compose](https://docs.docker.com/compose/), a simple command-line tool 
to define and run multi-container applications. We provde a sample `docker-compose.yml`
file in the repository:

```
version: "2"

services:
  mpi_head:
    image: carlochess/openmpi
    ports: 
     - "22"
    volumes:
     - ./app:/app
  mpi_node: 
    image: carlochess/openmpi
    volumes:
     - ./app:/app
```

The file defines an `mpi_head` and an `mpi_node`. Both containers run the same `openmpi` image. 
The only difference is, that the `mpi_head` container exposes its SHH server to 
the host system, so you can log into it to start your MPI applications.


The following command will start one `mpi_head` container and three `mpi_node` containers: 

```
$> docker-compose scale mpi_head=1 mpi_node=3
```
Once all containers are running, figure out the host port on which Docker exposes the  SSH server of the  `mpi_head` container: 

```
$> 
```

Now you know the port, you can login to the `mpi_head` conatiner. The username is `mpirun` and password `mpirun`:

 ```
$> ssh -p 23227 mpirun@localhost
```
 
## Start an MPI Container Cluster with Docker Swarm
You could also run in a cluster of computers using Docker Swarm or Swarm mode (Docker 1.12).

Consider use a nfs server with /app folder

First, create a Overlay network
```
$> docker network create --driver overlay uchuva-net 
```
Then you could use all the power of Swarm

```
$> DOCKER_HOST=IP_DOCKER_SWARM_MANAGER
$> docker-compose up -d
$> docker-compose scale mpi_head=1 mpi_node=2
---------------------------------------
XXX xxa aaa
XXX xxa aaa
XXX xxa aaa
---------------------------------------
```

...

```
$> ssh -p 23227 mpirun@remotehost #password: mpirun
$> ping XXX
```

## User Docker Machine


```
#!/bin/bash
set -e
#export DIGITAL_OCEAN_TOKEN=XXXXXXXXXXXXXXXXXXXX
#driver=digitalocean
#driverOpts=--digitalocean-access-token=$DIGITAL_OCEAN_TOKEN

driver=virtualbox
driverOpts=--virtualbox-memory=512

# create network
echo "----- Iniciamos la máquina de consul -----"
docker-machine create -d $driver $driverOpts discover

echo "----- Iniciamos consul -----"
docker $(docker-machine config discover) run -d -p "8500:8500" --name="consul" -h "consul" progrium/consul -server -bootstrap

echo "----- Iniciamos la máquina de registry -----"
docker-machine create -d $driver $driverOpts registry

echo "----- Iniciamos registry -----"
docker $(docker-machine config registry) run \
    -v /var/lib/boot2docker/:/certs \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.pem \
    -e REGISTRY_HTTP_TLS_KEY=/certs/server-key.pem \
    -d -p "5000:5000" --restart=always --name registry registry:2

echo "----- Creamos el swarm master -----"
docker-machine create -d $driver $driverOpts --swarm --swarm-master \
    --swarm-discovery="consul://$(docker-machine ip discover):8500" \
    --engine-opt="cluster-store=consul://$(docker-machine ip discover):8500" \
    --engine-opt="cluster-advertise=eth1:2376" \
    --engine-opt "insecure-registry $(docker-machine ip registry):5000" \
    swarm-master

docker $(docker-machine config swarm-master) run XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

for i in $(seq 1 2); do
    echo "----- Creando swarm node $i -----"
    docker-machine create -d $driver $driverOpts --swarm \
        --swarm-discovery="consul://$(docker-machine ip discover):8500" \
        --engine-opt="cluster-store=consul://$(docker-machine ip discover):8500" \
        --engine-opt="cluster-advertise=eth1:2376" \
        --engine-opt "insecure-registry $(docker-machine ip registry):5000" \
        swarm-node-0$i
done

echo "----- Ya puedes entrar al master -----"
eval $(docker-machine env --swarm swarm-master)

echo "----- Creamos un overlay network -----"
docker network create --driver overlay uchuva-net

echo " "
echo "---- ---------------- -----"
echo "---- Ready to hack ---"
echo "---- ---------------- -----"
```

When you finish, clean up the mess by:

```
$> docker-machine rm -y discover registry swarm-master swarm-node-0{1..2}
```

