<!--- master only -->
> ![warning](../images/warning.png) This document applies to the HEAD of the calico-docker source tree.
>
> View the calico-docker documentation for the latest release [here](https://github.com/projectcalico/calico-docker/blob/v0.13.0/README.md).
<!--- else
> You are viewing the calico-docker documentation for release **release**.
<!--- end of master only -->

# Running the Calico tutorials on Azure

## 1. Getting started with Azure
These instructions describe how to set up two CoreOS hosts on Azure.  For more general background, see
[the CoreOS on Azure documentation][coreos-azure].

Download and install nodejs, then install the Azure CLI.
On Ubuntu:
```
sudo apt-get install nodejs-legacy
sudo apt-get install npm
sudo npm install -g azure-cli
```
For more information, see Microsoft's [Install Azure CLI instructions][azure-instructions].

Log into your account:
```
azure login
```

### 1.1 Create cloud service for CoreOS cluster
Create a cloud service to connect your CoreOS cluster VMs to:
```
azure service create <cloud-service-name>
```

## 2. Create VM instances

```
azure vm create --custom-data=cloud-config.yaml --ssh=22 --ssh-cert=./myCert.pem --no-ssh-password --vm-name=node-1 --connect=coreos-cluster --location="West US" 2b171e93f07c4903bcad35bda10acf22__CoreOS-Stable-522.6.0 core
```

TODO: 
THIS IS THE CLASSIC MODE! Do we want this??? Do we want a version with Resource Manager instead???
HOWTO:
https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-coreos-how-to/


[coreos-azure]: https://coreos.com/os/docs/latest/booting-on-azure.html
[azure-instructions]: https://azure.microsoft.com/en-us/documentation/articles/xplat-cli-install/
