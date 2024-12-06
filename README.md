# byo-dcloud

Table of Contents

- [byo-dcloud](#byo-dcloud)
  - [SRv6 overview](#srv6-overview)
  - [Why SRv6?](#why-srv6)
  - [dCloud topo builder and launch dCloud instance](#dcloud-topo-builder-and-launch-dcloud-instance)
  - [ssh to dcloud VM](#ssh-to-dcloud-vm)
      - [Password is C1sco12345](#password-is-c1sco12345)
  - [Install Containerlab](#install-containerlab)
  - [XRd image](#xrd-image)
  - [topology yaml](#topology-yaml)
  - [Accessing routers](#accessing-routers)
  - [Appendix](#appendix)

## SRv6 overview
A brief PPT deck

## Why SRv6? 
* Customer example/use case
* Bell Canada
* AI backend (or FE)


## dCloud topo builder and launch dCloud instance

Link to [Lab Guide](Lab-Guide-for-BYO-dCloud-Lab.pdf)

Topology builder link:
https://tbv3-ui.ciscodcloud.com/

Once you've completed the topology builder/launch steps we'll pick up here

## ssh to dcloud VM

```
ssh dcloud@198.18.133.100
```
#### Password is C1sco12345

1.	Optional: change hostname and your password to something easier to type:

•	vi or nano /etc/hostname and /etc/hosts

•	Run the passwd command to change the pw

## Install Containerlab

We’ll use the open-source tool Containerlab to build/deploy our XRd network:
Containerlab homepage: https://containerlab.dev/

1. Link to Containerlab install instructions:  https://containerlab.dev/install/

2. Scroll down to Quick Setup, copy the curl/setupcommand and paste it into your ssh session:
```
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

The installation script will check for dependencies and install packages such as Docker and containerd. 

Once the script completes run “sudo clab version”. Example:

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

5. XRd requires us to tune a couple sysctl parameters. Use either vi or nano to edit /etc/sysctl.conf:

```
sudo nano /etc/sysctl.conf
```

Add the following lines to the end of the file:

```
net.bridge.bridge-nf-call-iptables=0
net.bridge.bridge-nf-call-ip6tables=0
fs.inotify.max_user_instances=65536
fs.inotify.max_user_watches=65536
kernel.pid_max=1048576
```
Then issue the following command to apply the change:

```
sudo sysctl -p
```

Example output:
```
dcloud@server:~/byo-dcloud$ sudo sysctl -p
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
fs.inotify.max_user_instances = 65536
fs.inotify.max_user_watches = 65536
kernel.pid_max = 1048576
dcloud@server:~/byo-dcloud$
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
We can ignore the cgroups and Hugepages errors. One interesting portion of the output is the number of XRd CP nodes the script estimates we can run on the VM:

```
xrd checks
-----------------------
PASS -- RAM
        Available RAM is 30.3 GiB.
        This is estimated to be sufficient for 15 XRd instance(s), although memory
        usage depends on the running configuration.
        Note that any swap that may be available is not included.
```

3. Launch the topology:

```
cd byo-dcloud
sudo clab deploy -t topology.yaml
```

Example output:
```
dcloud@server:~/byo-dcloud$ sudo clab deploy -t topology.yaml 
INFO[0000] Containerlab v0.60.0 started                 
INFO[0000] Parsing & checking topology file: topology.yaml 
INFO[0000] Creating docker network: Name="mgt", IPv4Subnet="172.20.2.0/24", IPv6Subnet="", MTU=0 
INFO[0000] Creating lab directory: /home/dcloud/byo-dcloud/clab-topo 
INFO[0000] Creating container: "xrd05"                  
INFO[0000] Creating container: "xrd03"                  
INFO[0000] Creating container: "xrd07"                  
INFO[0000] Creating container: "xrd02"                  
INFO[0000] Creating container: "xrd01"                  
INFO[0000] Creating container: "xrd06"                  
INFO[0000] Creating container: "xrd04"                  
INFO[0001] Created link: xrd01:Gi0-0-0-1 <--> xrd02:Gi0-0-0-0 
INFO[0001] Running postdeploy actions for Cisco XRd 'xrd02' node 
INFO[0001] Created link: xrd01:Gi0-0-0-2 <--> xrd05:Gi0-0-0-0 
INFO[0001] Running postdeploy actions for Cisco XRd 'xrd01' node 
INFO[0001] Created link: xrd02:Gi0-0-0-2 <--> xrd06:Gi0-0-0-1 
INFO[0002] Created link: xrd02:Gi0-0-0-1 <--> xrd03:Gi0-0-0-0 
INFO[0002] Created link: xrd05:Gi0-0-0-1 <--> xrd04:Gi0-0-0-2 
INFO[0002] Running postdeploy actions for Cisco XRd 'xrd04' node 
INFO[0002] Created link: xrd04:Gi0-0-0-1 <--> xrd07:Gi0-0-0-1 
INFO[0002] Created link: xrd03:Gi0-0-0-1 <--> xrd04:Gi0-0-0-0 
INFO[0002] Running postdeploy actions for Cisco XRd 'xrd03' node 
INFO[0003] Created link: xrd05:Gi0-0-0-2 <--> xrd06:Gi0-0-0-2 
INFO[0003] Running postdeploy actions for Cisco XRd 'xrd05' node 
INFO[0003] Running postdeploy actions for Cisco XRd 'xrd06' node 
INFO[0003] Created link: xrd06:Gi0-0-0-0 <--> xrd07:Gi0-0-0-2 
INFO[0003] Running postdeploy actions for Cisco XRd 'xrd07' node 
INFO[0003] Adding containerlab host entries to /etc/hosts file 
INFO[0003] Adding ssh config for containerlab nodes     
╭─────────────────┬─────────────────────────────────┬─────────┬────────────────╮
│       Name      │            Kind/Image           │  State  │ IPv4/6 Address │
├─────────────────┼─────────────────────────────────┼─────────┼────────────────┤
│ clab-topo-xrd01 │ cisco_xrd                       │ running │ 172.20.2.201   │
│                 │ ios-xr/xrd-control-plane:24.2.2 │         │ N/A            │
├─────────────────┼─────────────────────────────────┼─────────┼────────────────┤
│ clab-topo-xrd02 │ cisco_xrd                       │ running │ 172.20.2.202   │
│                 │ ios-xr/xrd-control-plane:24.2.2 │         │ N/A            │
├─────────────────┼─────────────────────────────────┼─────────┼────────────────┤
│ clab-topo-xrd03 │ cisco_xrd                       │ running │ 172.20.2.203   │
│                 │ ios-xr/xrd-control-plane:24.2.2 │         │ N/A            │
├─────────────────┼─────────────────────────────────┼─────────┼────────────────┤
│ clab-topo-xrd04 │ cisco_xrd                       │ running │ 172.20.2.204   │
│                 │ ios-xr/xrd-control-plane:24.2.2 │         │ N/A            │
├─────────────────┼─────────────────────────────────┼─────────┼────────────────┤
│ clab-topo-xrd05 │ cisco_xrd                       │ running │ 172.20.2.205   │
│                 │ ios-xr/xrd-control-plane:24.2.2 │         │ N/A            │
├─────────────────┼─────────────────────────────────┼─────────┼────────────────┤
│ clab-topo-xrd06 │ cisco_xrd                       │ running │ 172.20.2.206   │
│                 │ ios-xr/xrd-control-plane:24.2.2 │         │ N/A            │
├─────────────────┼─────────────────────────────────┼─────────┼────────────────┤
│ clab-topo-xrd07 │ cisco_xrd                       │ running │ 172.20.2.207   │
│                 │ ios-xr/xrd-control-plane:24.2.2 │         │ N/A            │
╰─────────────────┴─────────────────────────────────┴─────────┴────────────────╯
dcloud@server:~/byo-dcloud$ 
```

It generally takes about 2 minutes for the XRd nodes to come up and be available.
To check their uptime run docker ps:

```
docker ps
```

Example output:
```
dcloud@server:~/byo-dcloud$ docker ps
CONTAINER ID   IMAGE                             COMMAND            CREATED         STATUS         PORTS     NAMES
1c3996f5f431   ios-xr/xrd-control-plane:24.2.2   "/usr/sbin/init"   2 minutes ago   Up 2 minutes             clab-topo-xrd06
f6d3279cef2e   ios-xr/xrd-control-plane:24.2.2   "/usr/sbin/init"   2 minutes ago   Up 2 minutes             clab-topo-xrd04
edae330f6f5c   ios-xr/xrd-control-plane:24.2.2   "/usr/sbin/init"   2 minutes ago   Up 2 minutes             clab-topo-xrd02
b32c34773680   ios-xr/xrd-control-plane:24.2.2   "/usr/sbin/init"   2 minutes ago   Up 2 minutes             clab-topo-xrd01
9cb57e355217   ios-xr/xrd-control-plane:24.2.2   "/usr/sbin/init"   2 minutes ago   Up 2 minutes             clab-topo-xrd05
ae29c068b7b9   ios-xr/xrd-control-plane:24.2.2   "/usr/sbin/init"   2 minutes ago   Up 2 minutes             clab-topo-xrd07
6933f747536b   ios-xr/xrd-control-plane:24.2.2   "/usr/sbin/init"   2 minutes ago   Up 2 minutes             clab-topo-xrd03
dcloud@server:~/byo-dcloud$ 
```

## Accessing routers

1. Containerlab creates host entries in /etc/hosts for each node in the topology. We can use these to ssh into the routers:

```
ssh cisco@clab-topo-xrd01
or
ssh cisco@172.20.2.201
```
Password is cisco123

2. From xrd01 (or really any node), validate the network is working as expected:

```
show isis database
show bgp summary
show segment-routing srv6 sid
```

## Appendix

Containerlab commands:
```
sudo clab -h
sudo clab deploy -t <topology yaml file>
sudo clab destroy -t <topology yaml file>
```

