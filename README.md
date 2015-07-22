# Chameleon Cloud Tutorial - Docker Machine, Compose, and Swarm

:warning: :warning: :warning:  
**Tutorial Under Development**

**Because of incompatibilities this tutorial doesn't currently use Chameleon resources. Instead you can use another cloud provider or local VM's. See [this](#setup-swarm-cluster) section for details.**

This tutorial will cover using Docker Machine, Compose and Swarm. In the first tutorial we setup containers on 2 different hosts and linked them together to run a simple webpage. In this tutorial we will set up a similar page that lets you post messages and lists those previously posted. It uses 3 containers and we'll arrange them with Compose.

Machine allows us to create Docker hosts and control them without interacting with the host machines directly. This way you don't have to SSH to machines running the Docker daemon to run containers.

Compose simplifies the process of arranging and linking containers together. Compose lets us specify the links and runtime configurations of containers in a single config file, rather than having several lengthy commands to execute in the right sequence.

Swarm is used to group multiple Docker hosts together so that containers or groups of containers can scale across machines.

Machine, Compose, and Swarm can be used together to simply and powerfully orchestrate a service. This is what we'll doing in this tutorial.

## Prerequisites

It's expected that you have gone through [Docker Tutorial 1](http://cloudandbigdatalab.github.io/docs/Chameleon%20Cloud%20Tutorial%20-%20Docker%20Fundamentals.pdf) or are already familiar with its content. No more prior knowledge is required past the first tutorial.

## Steps Outline

\# | Description | Time (mins)
---|-------------|------------
TBD

## Setup Swarm Cluster

**For this tutorial we're using a personal laptop and the Rackspace cloud.**

We're using a personal laptop because it's sensible to control remote machines from a local machine rather than SSH'ing to a remote machine to control remote machines. However, you can do all your work from a Chameleon instance. Every step is the same past installation, except you'll need to add `sudo` to your commands on the default Chameleon CentOS image.

We're using Rackspace because Machine doesn't currently support passing in reservation ids during host creation and therefore does not work with Chameleon. Support for Chameleon will likely happen in the future. See this [issue](https://github.com/docker/machine/issues/1461) on their GitHub.

### Installation

For this demo we used a personal MacBook running OS X but installation instructions are available for basically any OS. Here are the instructions for [Machine](https://docs.docker.com/machine/#installation) and [Compose](https://docs.docker.com/compose/install/). **If you're getting "Permission Denied" using curl, run `sudo -i` to become root, run the commands, then `exit`.**

### Swarm Token

Create a machine. We're naming it `docker-main`.

```shell
docker-machine create \
  -d rackspace \
  --rackspace-username $USERNAME \
  --rackspace-api-key $API_KEY \
  --rackspace-region $REGION \
  docker-main
```

Load the machine env variables.

```shell
eval "$(docker-machine env docker-main)"
```

Generate a token. Save the token for later. You can set it as env variable if you'd like.

```shell
docker run swarm create
```

### Swarm Master

Create the Swarm master.

```shell
docker-machine create \
  -d rackspace \
  --rackspace-username $USERNAME \
  --rackspace-api-key $API_KEY \
  --rackspace-region $REGION \
  --swarm \
  --swarm-master \
  --swarm-discovery token://$TOKEN \
  docker-swarm-master
```

### Swarm Nodes

We're using a shell loop to create 2 swarm nodes. You can create as many as you want. (Creating 2 on Rackspace took about 3 minutes.)

```shell
for((i=0;i<2;i++)); do \
  docker-machine create \
    -d rackspace \
    --rackspace-username $USERNAME \
    --rackspace-api-key $API_KEY \
    --rackspace-region $REGION \
    --swarm \
    --swarm-discovery token://$TOKEN \
    docker-swarm-node-$i; \
done
```

### Connect to Swarm Master

```shell
eval "$(docker-machine env --swarm docker-swarm-master)"
```

You can see info about your swarm with `docker info`. The output should be similar to:

```shell
Containers: 4
Images: 3
Role: primary
Strategy: spread
Filters: affinity, health, constraint, port, dependency
Nodes: 3
 docker-swarm-master: 23.253.242.223:2376
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.014 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-37-generic, operatingsystem=Ubuntu 14.04.1 LTS, provider=rackspace, storagedriver=aufs
 docker-swarm-node-0: 104.130.127.222:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.014 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-37-generic, operatingsystem=Ubuntu 14.04.1 LTS, provider=rackspace, storagedriver=aufs
 docker-swarm-node-1: 104.130.139.18:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.014 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-37-generic, operatingsystem=Ubuntu 14.04.1 LTS, provider=rackspace, storagedriver=aufs
CPUs: 3
Total Memory: 3.041 GiB
```

## Compose

Now we're going to launch our composition. Download the [docker-compose.yml](https://github.com/cloudandbigdatalab/chameleon-cloud-tutorial-docker-2/blob/master/docker-compose.yml) file from our repo. This lays out the 3 container composition.

Container Name | Apps | Description
----------|------|------------
server | Nginx | handles http requests
page | uWSGI and Django | uWSGI connects Nginx to Django, Django generates the html
db | Postgres | database for page, Django connects to Postgres

The Dockerfiles and code for the containers are in their respective folders. Note for the Postgres container we're using the unmodified image off Docker Hub so their isn't a folder for it.

## Run the Composition

`-p tutorial-2` specifies our project name. Otherwise the name of the directory would be used.

```shell
docker-compose -p tutorial-2 up -d
```

## Scale the Composition
