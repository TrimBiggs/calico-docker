<!--- master only -->
> ![warning](../images/warning.png) This document applies to the HEAD of the calico-mesos-deployments source tree.
>
> View the calico-mesos-deployments documentation for the latest release [here](https://github.com/projectcalico/calico-mesos-deployments/blob/0.26.0%2B1/README.md).
<!--- else
> You are viewing the calico-mesos-deployments documentation for release **release**.
<!--- end of master only -->

# Calico with Marathon and the Mesos Docker Containerizer

The following instructions outline the steps required to enable and manage 
Calico networked containers launched from Marathon using the Mesos Docker
containerizer.  This guide covers:
-  Setting up Docker multi-host networking
-  Installing and configuring Calico
-  Creating a Docker network and managing network policy
-  Launching a container

## Prerequisites

To utilize the Docker libnetwork plugin feature of Docker, it is necessary to 
run with a Docker version 1.9 or higher, and to configure Docker to use a
cluster store.  

You can easily configure a mesos cluster by running through
the [Calico Mesos Vagrant guide](./Vagrant.md). If you do this, you can skip
directly to [Launching Containers](#launching-containers).

TODO: Check above link works

#### Required Machines 

The cluster must contain the following machines:

- Mesos Master
- Mesos Agent(s)
	- Docker 1.9+ (see below)

#### Required Services

You will also need to run the following services:

TODO: Fix these links

- [Marthon](./)
- [Etcd](./)

It is easiest to just run these services on the Mesos Master machine.
Each machine in the cluster must have access to these services.

#### Set up Docker daemon to use etcd as a cluster store

In order to run the Docker Containerizer, you will need to do the following
on each agent:

-  Stop your Docker service
	-  e.g. `systemctl stop docker` or `stop docker`
-  If you don’t already have at least Docker 1.9 then upgrade/install Docker
   1.9 or higher.
-  Modify the docker daemon parameters to point at the etcd cluster, such as:
	- Modify the Default Docker parameters file to include
	  `DOCKER_OPTS="--cluster-store=etcd://<IP address>:4001"`
    - Run docker daemon with `--cluster-store=etcd://<IP address>:4001`
-  Start your docker service.

Be sure replace `<IP address>` with IP address of your etcd cluster.

## Install and set up Calico

#### Install Calico on each agent

To install Calico on each agent, run the following commands on each agent:

```
wget https://github.com/projectcalico/calico-containers/releases/download/v0.17.0/calicoctl
chmod a+x calicoctl 
sudo ETCD_AUTHORITY=<IP Address>:4001 ./calicoctl node --libnetwork
```

Be sure to set the ETCD_AUTHORITY to the correct `IP:Port` for your etcd cluster.

## Creating a Docker network and managing network policy

With Calico, a Docker network represents a logical set of rules that define the 
allowed traffic in and out of containers assigned to that network.  The rules
are encapsulated in a Calico "profile".  Each Docker network is assigned its 
own Calico profile.

#### Create a Docker network

To create a Docker network using Calico, run the `docker network create`
command specifying "calico" as both the network and IPAM driver.

For example, suppose we want to provide network policy for a set of database
containers.  We can create a network called `databases` with the the following
command:

```
docker network create --driver calico --ipam-driver calico databases 
```

#### View the policy associated with the network

You can use the `calicoctl profile <profile> rule show` to display the
rules in the profile associated with the `databases` network.

The network name can be supplied as the profile name and the `calicoctl` tool
will look up the profile associated with that network.

```
$ ETCD_AUTHORITY=<IP address>:4001
$ calicoctl profile databases rule show
Inbound rules:
   1 allow from tag databases
Outbound rules:
   1 allow
```

As you can see, the default rules allow all outbound traffic and accept inbound
traffic only from containers attached the "databases" network.

> Note that when managing profiles created by the Calico network driver, the
> profile tag and network name can be regarded as the same thing.

#### Configuring the network policy

Calico has a rich set of policy rules that can be leveraged.  Rules can be
created to allow and disallow packets based on a variety of parameters such
as source and destination CIDR, port and tag.

The `calicoctl profile <profile> rule add` and `calicoctl profile <profile> rule remove`
commands can be used to add and remove rules in the profile.
  
As an example, suppose the databases network represents a group of MySQL
databases which should allow TCP traffic to port 3306 from "application" 
containers.

To achieve that, create a second network called "applications" which the
application containers will be attached to.  Then, modify the network policy of
the databases to allow the appropriate inbound traffic from the applications.

```
$ docker network create --driver calico --ipam-driver calico applications
$ calicoctl profile databases rule add inbound allow tcp from tag applications to ports 3306
```

The second command adds a new rule to the databases network policy that allows
inbound TCP traffic from the applications.

You can view the updated network policy of the databases to show the newly
added rule:

```
$ calicoctl profile databases rule show
Inbound rules:
   1 allow from tag databases
   2 allow tcp from tag applications to ports 3306
Outbound rules:
   1 allow
```

For more details on the syntax of the rules, run `calicoctl profile --help` to
display the valid profile commands.

## Launching Containers

With your networks configured, it is trivial to launch a Docker container 
through Mesos using the standard Marathon UI and API.

#### Launching a container through the UI

Through the Marathon UI at `MARATHON_IP:8080`, launch a Docker task as normal.
Select an arbitrary(*) network (Bridge or Host), and then provide the
following additional parameter (under the Docker options):

```
Key = net
Value = <network name>
```

Where `<network name>` is the name of the network, for example "databases".

> (*) The selection is arbitrary because the additional net parameter overrides
> the selected network type.

#### Launching a container with a JSON blob

To launch a container using the Marathon API with a JSON blob, simply include
the net parameter in the request.  For example:

```
{
    "id":"/calico-apps",
    "apps": [
        {
            "id": "unified-docker-task",
            "cmd": "ip addr && sleep 300",
            "cpus": 0.1,
            "mem": 64.0,
            "container": {
                "type": "DOCKER",
                "docker": {
                    "image": "busybox",
                    "parameters": [
                        {"key": "net", "value": "databases"}
                    ]
                }
            }
        }
    ]
}
```

## Additional Options (Optional)


#### Configure your IP pools

By default, Calico creates an IP pool with CIDR 192.168.0.0/16.  The Calico 
IPAM module will assign IP addresses to your containers from that range of IP 
addresses.

If you want to allocate from different CIDRs you can use the 
`calicoctl pool remove` and `calicoctl pool add` commands to modify the 
available pools.  For example, to change the pool from 192.168.0.0/16 to be 
172.168.0.0/16 run the following to remove the original pool and create a new
one: 

```
calicoctl pool remove 192.168.0.0/16
calicoctl pool add 172.168.0.0/16
```

You can view the configured pools using the `calicoctl pool show` command.  e.g.

```
$ ./calicoctl pool show
+----------------+---------+
|   IPv4 CIDR    | Options |
+----------------+---------+
| 172.168.0.0/16 |         |
+----------------+---------+
+--------------------------+---------+
|        IPv6 CIDR         | Options |
+--------------------------+---------+
| fd80:24e2:f998:72d6::/64 |         |
+--------------------------+---------+
```

TODO: Include Load balancer information here.


[![Analytics](https://calico-ga-beacon.appspot.com/UA-52125893-3/calico-mesos-deployments/docs/CalicoWithTheDockerContainerizer.md?pixel)](https://github.com/igrigorik/ga-beacon)
