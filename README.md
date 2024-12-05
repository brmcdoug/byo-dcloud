# byo-dcloud

Table of Contents

- [byo-dcloud](#byo-dcloud)
  - [SRv6 overview](#srv6-overview)
  - [Why SRv6?](#why-srv6)
  - [dCloud topo builder](#dcloud-topo-builder)
  - [launch dcloud instance](#launch-dcloud-instance)
  - [Install Containerlab](#install-containerlab)
  - [XRd image](#xrd-image)
  - [topology yaml](#topology-yaml)
  - [srv6 config](#srv6-config)
  - [launch topology](#launch-topology)

## SRv6 overview
A brief PPT deck

## Why SRv6? 
* Customer example/use case
* Bell Canada
* AI backend (or FE)

Link to [Lab Guide](Lab-Guide-for-BYO-dCloud-Lab.pdf)

## dCloud topo builder
https://tbv3-ui.ciscodcloud.com/


## launch dcloud instance
## Install Containerlab

Containerlab home page
https://containerlab.dev/

Installation link
https://containerlab.dev/install/

1. Scroll down to Quick Setup, copy the curl/setupcommand and paste it into your ssh session:
```
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

The installation script will check for dependencies and install packages such as Docker and containerd. Once the script completes run “sudo clab version”. Example:

```
dcloud@server:~$ sudo clab version
  ____ ___  _   _ _____  _    ___ _   _ _____ ____  _       _     
 / ___/ _ \| \ | |_   _|/ \  |_ _| \ | | ____|  _ \| | __ _| |__  
| |  | | | |  \| | | | / _ \  | ||  \| |  _| | |_) | |/ _` | '_ \ 
| |__| |_| | |\  | | |/ ___ \ | || |\  | |___|  _ <| | (_| | |_) |
 \____\___/|_| \_| |_/_/   \_\___|_| \_|_____|_| \_\_|\__,_|_.__/ 

    version: 0.60.0
     commit: 53c2ce42
       date: 2024-12-05T18:11:54Z
     source: https://github.com/srl-labs/containerlab
 rel. notes: https://containerlab.dev/rn/0.60/
dcloud@server:~$
```
2. Optional: Add your user to the docker group so you don't have to use sudo to run containerlab commands.
   https://docs.docker.com/engine/install/linux-postinstall/

```
sudo usermod -aG docker $USER
```

exit and ssh/login again to refresh your group memberships. Then test docker commands without sudo:

```
docker ps
```
Example output:
```
dcloud@server:~$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
dcloud@server:~$
```

## XRd image

1. Acquire XRd image...for the sake of time, we'll use an XRd image posted to Box. From your ssh session, run:

```
wget -O xrd-control-plane-container-x86.24.2.2.tgz -L https://cisco.box.com/shared/static/8ca2aa6cei6np3w04aan4ga1k7d58506
```

This takes a few minutes. Back to the PPT.

2. Untar the image:

```
tar -xvf xrd-control-plane-container-x86.24.2.2.tgz
```
example output:
```
dcloud@server:~$ tar -xvf xrd-control-plane-container-x86.24.2.2.tgz 
./
./IOS-XR-SW-XRd.crt
./cisco_x509_verify_release.py3
./cisco_x509_verify_release.py3.README
./cisco_x509_verify_release.py3.signature
./xrd-control-plane-container-x64.dockerv1.tgz
./xrd-control-plane-container-x64.dockerv1.tgz.signature
dcloud@server:~$
```

3. Docker load the image:

```
docker load -i xrd-control-plane-container-x64.dockerv1.tgz 
```

Example output:
```
dcloud@server:~$ docker load -i xrd-control-plane-container-x64.dockerv1.tgz 
7eda0ae02719: Loading layer [==================================================>]  1.315GB/1.315GB
Loaded image: ios-xr/xrd-control-plane:24.2.2
dcloud@server:~$ 
```

4. Verify the image is available:

```
docker images
```

Example output:
```
dcloud@server:~$ docker images
REPOSITORY                 TAG       IMAGE ID       CREATED        SIZE
ios-xr/xrd-control-plane   24.2.2    a931f15c0a0d   2 months ago   1.28GB
dcloud@server:~$ 
```

5. Increase the kernel pid parameter:

```
sudo sysctl -w kernel.pid_max=1048576
```

Setting the parameter via CLI is only temporary. To make the change permanent, add the following line to /etc/sysctl.conf:

```
kernel.pid_max=1048576
```
Then issue the following command to apply the change:

```
sudo sysctl -p
```

## topology yaml
Containerlab uses a yaml file to define the topology, and it is enormously flexible.
My today's purposes we'll use the 7-node topology here: [topology.yaml](topology.yaml)

In the yaml file, we define the topology name, mgt network, nodes, images, path to config file, and links in the network.

1. git clone this repo to your VM:

```
git clone https://github.com/brmcdoug/byo-dcloud.git
```

Feel free to review the configs in the [xrd-config](xrd-config) folder

2. Before we launch the topology, let's run the host-check script to verify our host will support the topology:

```
cd byo-dcloud/util
./host-check
```

## srv6 config
## launch topology
