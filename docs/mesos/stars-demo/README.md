<!--- master only -->
> ![warning](../../images/warning.png) This document applies to the HEAD of the calico-mesos-deployments source tree.
>
> View the calico-mesos-deployments documentation for the latest release [here](https://github.com/projectcalico/calico-mesos-deployments/blob/0.27.0%2B2/README.md).
<!--- else
> You are viewing the calico-mesos-deployments documentation for release **release**.
<!--- end of master only -->

# Stars demo with the Mesos Docker Containerizer
The included demo uses the stars visualizer to set up a frontend and backend service, as well as a client service, all running on mesos. It then configures network policy on each service.

## Prerequisites
This demo requires a Mesos cluster with Calico-libnetwork running. Click on the
following items for information on how to set them up:
- [etcd cluster](#etcd-guide)
- [mesos master](#mesos-master)
- mesos slave
    - [docker 1.9+ configured to use your etcd cluster as its datastore](#docker-multi-network)
    - [calicoctl binary installed](#install-calicoctl)
    - [calico node & libnetwork](#calico-node)
    
To quickly launch a ready cluster, see the [vagrant mesos guide](#calico-mesos)

## Overview
This demo uses stars, a network connectivity visualizer. We will launch the following four
dummy tasks across the cluster:
- Backend 
- Frontend
- Client
- Management-UI

Client, Backend, and Frontend will each be run as a star-probe, which will attempt
to communicate with each other probe, and report their status on a self-hosted webserver.

Management-UI runs star-collect, which collects the status from each of the 
probes, and generates a viewable web page which illustrates the current state of the network.

## Getting Started
### Prep: 
On each agent, pull docker image `djosborne/star:v0.5.0` for a faster marathon launch later.

	docker pull djosborne/star:v0.5.0

On one of your agents, download this repository. You will run the rest of this guide from this agent.

	curl -O -L https://github.com/djosborne/calico-containers/archive/mesos-docker-stars.tar.gz
	tar -xvf mesos-docker-stars.tar.gz
	cd calico-containers-mesos-docker-stars/


### 1. Create a Docker network
With Calico, a Docker network represents a logical set of rules that define the 
allowed traffic in and out of containers assigned to that network.  The rules
are encapsulated in a Calico "profile".  Each Docker network is assigned its 
own Calico profile.

For this demo, we will create a network for each service, stso that we can specify a unique set of rules for each. Run the following commands on any agent to create the networks:

```
docker network create --driver calico --ipam-driver calico --subnet=192.168.0.0/16 management-ui
docker network create --driver calico --ipam-driver calico client
docker network create --driver calico --ipam-driver calico frontend
docker network create --driver calico --ipam-driver calico backend
```
TODO: Note on the subnet

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
To launch a container using the Marathon API with a JSON blob, simply include
the net parameter in the request.  Here's a sample blob of what our collector looks like.

```
{
    "id":"/calico-apps",
    "apps": [
        {
          "id": "client",
          "cmd": "star-probe --urls=http://frontend.calico-stars.marathon.mesos:9000/status,http://backend.calico-stars.marathon.mesos:9000/status",
          "cpus": 0.1,
          "mem": 64.0,
          "container": {
            "type": "DOCKER",
            "docker": {
              "image": "mesosphere/star:v0.3.0",
              "parameters": [
                { "key": "net", "value": "client" }
              ]
            }
          }
        }
    ]
}
```

Note the `parameters` field which specifies which docker network to join, as well as
the static IP address request.

To speed things up, we'll use the prefilled [stars.json](./stars.json) file. Launch it on mesos by curling it to Marathon. Be sure to set your Marathon IP as appropriate:

```
export MARATHON_IP=172.24.197.101
curl -X PUT -H "Content-Type: application/json" http://$MARATHON_IP:8080/v2/groups/calico-stars  -d @docs/mesos/stars-demo/stars.json
```

> Note: The tasks may be "Deploying" for awhile as the star docker image is pulled.

#### [Alternative] Launching a container through the UI
This method of using calico-libnetwork to launch docker containers is
accessible through the standard Marathon UI.
When launching a task, select an arbitrary(*) network
(Bridge or Host), and then provide the following additional parameter 
(under the Docker options)

```
Key = net
Value = <network name>
```

Where `<network name>` is the name of the network, for example "databases".

> (*) The selection is arbitrary because the additional net parameter overrides
> the selected network type.

### 3. Add route to Calico IP's
Before viewing the Marathon UI, you will need to ensure that you can reach the Marathon IP. It is likely that the device you
are using to view the Marathon UI is connecting through a router which is not
peering with the calico routers, and therefore does not know how to route 
to the calico-assigned IPs. There are several solutions to this, 
(and for information on them, [contact us on slack!][slack]),
but for now, we'll follow the one outlined in [Exposing Container Port to Internet](https://github.com/projectcalico/calico-containers/blob/master/docs/ExposePortsToInternet.md#expose-container-port-to-host-interface--internet).

```
iptables -A PREROUTING -t nat -i eth1 -p tcp --dport 9001 -j DNAT  --to 192.168.255.254:9001
iptables -t nat -A OUTPUT -p tcp -o lo --dport 9001 -j DNAT --to-destination 192.168.255.254:9001
```

### 4. View the UI
Now that we can route to our task, let's view the UI. Our UI was passed a label to specially
request the address `172.24.197.101:9001` for the collector UI. Let's visit that IP (this will fail!):

- http://172.24.197.101:9001

Oh no! Our connection is refused, and We can't see the UI. Let's use the `calicoctl profile <profile> rule show` 
to display the rules in the profile associated with the `management-ui` network:

	$ calicoctl profile management-ui rule show

	Inbound rules:
	   1 allow from tag management-ui
	Outbound rules:
	   1 allow

As you can see, the default rules allow all outbound traffic, but only accept inbound
traffic from endpoints also attached to the "management-ui" network.

Our dev box is trying to view the Management-UI from an IP that isn't attached to any 
known endpoints. Therefore, calico is blocking the connection.
Lets re-configure the management to allow connections from anywhere, so we can access it 
on port 80 from our dev box:

```
export ETCD_AUTHORITY=<ETCD_AUTHORITY>
calicoctl profile management-ui rule remove inbound allow from tag management-ui
calicoctl profile management-ui rule add inbound allow tcp to ports 9001
```

>Replace `<ETCD_AUTHORITY>` with the correct `ip:port` value. For example, if you installed
>your cluster using vagrant, your `ETCD_AUTHORITY` will be `172.24.197.101:2379`.

Changes to calico profiles are distributed immediately across the network. 
So we should immediately be able to view the UI: 
- http://192.168.255.254:9000

Hmm, so the web page is viewable, but its blank! Well, its grey. But where is the pretty
diagram? Another look at the profile and we realize that the managemtn-UI is not 
allowed to communicate with each probe to find out their statuses! We'll need
to add a rule to each network to allow this connection through:

```
calicoctl profile client rule add inbound allow tcp from tag management-ui to ports 9000
calicoctl profile backend rule add inbound allow tcp from tag management-ui to ports 9000
calicoctl profile frontend rule add inbound allow tcp from tag management-ui to ports 9000
```

Lets try again:
- http://192.168.255.254:9000

Hooray! The probes are viewable. Now its time to configure sensible routes between the services in our cluster. Lets add some policies to make the following statements true:

- The frontend services should respond to requests from clients:

    calicoctl profile frontend rule add inbound allow tcp from tag client to port 9001

- The backend services should respond to requests from the frontend:

	calicoctl profile backend rule add inbound allow tcp from tag frontend to port 9001

    
Lets see what our cluster looks like now:
- http://192.168.255.254:9000

That's it!

```
curl -X DELETE http://$MARATHON_IP:8080/v2/groups/calico-stars?force=true

```

[![Analytics](https://calico-ga-beacon.appspot.com/UA-52125893-3/calico-mesos-deployments/docs/CalicoWithTheDockerContainerizer.md?pixel)](https://github.com/igrigorik/ga-beacon)