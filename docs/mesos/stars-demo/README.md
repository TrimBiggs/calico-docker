<!--- master only -->
> ![warning](../../images/warning.png) This document applies to the HEAD of the calico-mesos-deployments source tree.
>
> View the calico-mesos-deployments documentation for the latest release [here](https://github.com/projectcalico/calico-mesos-deployments/blob/0.27.0%2B2/README.md).
<!--- else
> You are viewing the calico-mesos-deployments documentation for release **release**.
<!--- end of master only -->

# Stars demo with the Mesos Docker Containerizer
This demo uses the stars network visualizer to set up a frontend and backend service, as well as a client service and UI, all running on Mesos. It then configures network policy on each service.

The goal of this demo is to provide a meaningful visualization of how Calico
manages security between services in a Mesos cluster.

For a deeper look at how to configure Calico with the Docker containerizer,
check out [Calico's Docker Containerizer guide](../UsageGuideDockerContainerizer).

## Prerequisites
This demo requires a Mesos cluster with Calico-libnetwork running,
along with a few additional components.

To simplify the setup, we have created a Vagrant file to quickly
deploy a master and two agents. Follow the [Vagrant Mesos Guide](./Vagrant.md)
to get started.

Your cluster should contain the following components.

- Mesos Master Instance - `172.24.197.101`
	- Etcd - `172.24.197.101:2379`
	- Marathon - `172.24.197.101:8080`
	- Marathon Load Balancer (running in Docker container)
- Two Mesos Agent Instances - `172.24.197.102`, `172.24.197.103`
    - Calicoctl
    - `calico/node` and `calico/node-libnetwork` running in Docker 1.9+

## Overview
This demo uses Stars, a network connectivity visualizer. We will launch the following four
dummy tasks across the cluster using the Docker containerizer:
- Backend
- Frontend
- Client
- Management-UI

Client, Backend, and Frontend will each be run as a star-probe, which will attempt
to communicate with each other probe, and report their status on a self-hosted webserver.

Management-UI runs star-collect, which collects the status from each of the
probes and generates a viewable web page illustrating the current state of
the network.  We will use the Marathon load balancer to access the Stars UI
using port mapping from the host to the Management UI container.

## Getting Started
### Prep: 
On each agent, pull the Docker image `djosborne/star:v0.5.0` to speed up the
Marathon install once the tasks start.

	docker pull djosborne/star:v0.5.0

On one of your agents, download the [stars.json](./stars.json) from this directory.

### 1. Create a Docker network
With Calico, a Docker network represents a logical set of rules that define the
allowed traffic in and out of containers assigned to that network.  The rules
are encapsulated in a Calico "profile".  Each Docker network is assigned its
own Calico profile.

For this demo, we will create a network for each service so that we can specify a unique set of rules for each. Run the following commands on any agent to create the networks:

```
docker network create --driver calico --ipam-driver calico --subnet=192.168.0.0/16 management-ui
docker network create --driver calico --ipam-driver calico client
docker network create --driver calico --ipam-driver calico frontend
docker network create --driver calico --ipam-driver calico backend
```

>The subnet is passed in here to ensure that the IP address of the `management-ui`
>can be statically configured.

Check that our networks were created by running the following command on any agent:

	$ docker network ls

	NETWORK ID          NAME                DRIVER
	5b20a79c129e        bridge              bridge
	60290468013e        none                null
	726dcd49f16c        host                host
	58346b0b626a        management-ui       calico
	9c419a7a6474        backend             calico
	9cbe2b294d34        client              calico
	ff613162c710        frontend            calico

### 2. Launch the demo
With your networks created, it is trivial to launch a Docker container
through Mesos using the standard Marathon UI and API.

#### Using Marathon's REST API to Launch Calico Tasks
You can launch a new task by passing a JSON blob to the Marathon REST API.

##### Example JSON
Here's a sample blob of what the Management UI task looks like as JSON.

```
{
  "id":"/calico-apps",
  "apps": [
      {
        "id": "management-ui",
        "cmd": "star-probe --urls=http://frontend.calico-stars.marathon.mesos:9000/status,http://backend.calico-stars.marathon.mesos:9000/status",        
        "cpus": 0.1,
        "mem": 64.0,
        "container": {
          "type": "DOCKER"
          "docker": {
            "portMappings":[{"containerPort": 9001, "servicePort": 10000}],
            "image": "mesosphere/star:v0.3.0",
            "parameters": [
              { "key": "net", "management-ui" },
              { "key": "ip", "value": "192.168.255.254" }
            ]
          }
        }
      }
  ]
}
```

Note the `parameters` field which specifies:

- The Docker network to join
- A specific IP address from the Calico Pool to set as the Management UI IP

Also note the `portMappings` field, which maps the Management UI's port 9001 to
the port 10000 of the host containing the Marathon load balancer.  For this
demo, this means that `http://172.24.197.101:100000` maps to `http://192.168.255.254:9001`.

##### Start a Task
To speed things up, we'll use the prefilled [stars.json](./stars.json) file
that you downloaded earlier on your agent. This file contains four tasks to
create containers for the management-ui, client, frontend, and backend services.

First you'll need to set the `MARATHON_IP` to be the IP address of the machine that is running Marathon:

	export MARATHON_IP=172.24.197.101

Then, using the Mesos agent that contains the `stars.json` file,
launch a new Marathon task with the following curl command
(make sure that you are using the correct path to the `stars.json` file):

	curl -X PUT -H "Content-Type: application/json" http://$MARATHON_IP:8080/v2/groups/calico-stars  -d @stars.json

You can view the Marathon dashboard by visiting `http://<MARATHON_IP>:8080` in your browser.

### 3. View the Management UI
Now that we have configured our Marathon tasks, let's view the Stars UI.

#### View the Stars UI from a Browser
As mentioned above, the Stars Management UI container JSON passes a port mapping
to the Marathon load balancer when creating the container.  Since the load balancer
is running on Mesos Master, you can access the UI's port 9001 by visiting port
10000 on the Master.

Before we configure Calico policy for the UI, let's ***try*** to access
the webpage on Master from a machine that can reach the Master IP:

	http://172.24.197.101:10000

Our connection is refused since the default behavior of a Calico profile
is to only allow inbound traffic from nodes with the same profile
(or from nodes in the same network, in this case).

#### Allow Traffic to `management-ui` Network
Let's view the rules for the `management-ui` network's profile by running the
`calicoctl profile <profile> rule show` command:

	$ export ETCD_AUTHORITY=172.24.197.101:2379
	$ calicoctl profile management-ui rule show

	Inbound rules:
	   1 allow from tag management-ui
	Outbound rules:
	   1 allow

As you can see, the default rules allow all outbound traffic, but only accept inbound
traffic from endpoints also attached to the `management-ui` network.

Lets re-configure the profile to allow connections to port 9001 from anywhere,
so we can access it in the browser:

```
calicoctl profile management-ui rule remove inbound allow from tag management-ui
calicoctl profile management-ui rule add inbound allow tcp to ports 9001
```

At this point, the web page is viewable, but there is no data or information about
the cluster connections.  This is because the client, frontend, and backend
networks are also blocking incoming traffic!

#### Allow Traffic to `client`, `frontend`, and `backend` Networks
Let's add a rule to each network to allow the `management-ui` network to
probe port 9000 of the three other networks:

```
calicoctl profile client rule add inbound allow tcp from tag management-ui to ports 9000
calicoctl profile backend rule add inbound allow tcp from tag management-ui to ports 9000
calicoctl profile frontend rule add inbound allow tcp from tag management-ui to ports 9000
```

Lets try the webpage again:

	http://172.24.197.101:10000

The nodes are viewable! Now its time to configure sensible network policy
between the services in our cluster.

#### Configure Additional Policy
Lets add some policies to make the following statements true:

**The frontend services should respond to requests from clients:**

	calicoctl profile frontend rule add inbound allow tcp from tag client to port 9001

**The backend services should respond to requests from the frontend:**

	calicoctl profile backend rule add inbound allow tcp from tag frontend to port 9001

Lets see what our cluster looks like now:

	http://172.24.197.101:10000

Hooray! You've configured policy with Calico to allow certain networks to access
other networks in your cluster!

[![Analytics](https://calico-ga-beacon.appspot.com/UA-52125893-3/calico-mesos-deployments/docs/CalicoWithTheDockerContainerizer.md?pixel)](https://github.com/igrigorik/ga-beacon)