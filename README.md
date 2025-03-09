# byo-dcloud

Table of Contents

- [byo-dcloud](#byo-dcloud)
  - [dCloud Topology Builder and Launch dCloud Instance](#dcloud-topology-builder-and-launch-dcloud-instance)
  - [ssh to the dCloud VM](#ssh-to-the-dcloud-vm)
  - [Install Containerlab](#install-containerlab)
  - [XRd Docker Image](#xrd-docker-image)
  - [Containerlab Topology Definition](#containerlab-topology-definition)
  - [ssh to XRd Routers](#ssh-to-xrd-routers)
  - [Run Some Pings](#run-some-pings)
    - [SRv6 L3VPN Reachability](#srv6-l3vpn-reachability)
    - [SRv6 TE Policies](#srv6-te-policies)
  - [Additional Resources](#additional-resources)
  - [Appendix](#appendix)


## dCloud Topology Builder and Launch dCloud Instance

Link to [Lab Guide](Lab-Guide-for-BYO-dCloud-Lab.pdf)

Topology builder link:
https://tbv3-ui.ciscodcloud.com/

Once you've completed the topology builder/launch steps we'll pick up here

## ssh to the dCloud VM

```
ssh dcloud@198.18.133.100
```
*Password is C1sco12345*

1.	Optional - change hostname and your password to something easier to type:

•	vi or nano /etc/hostname and /etc/hosts

•	Run the passwd command to change the pw

## Install Containerlab

We’ll use the open-source tool Containerlab to build/deploy our XRd network

Containerlab homepage: https://containerlab.dev/

1. Link to Containerlab install instructions:  https://containerlab.dev/install/

2. Scroll down to Quick Setup, copy the curl/setup command and paste it into your ssh session:
```
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

The installation script will check for dependencies and install packages such as Docker and containerd in addition to containerlab. 

Once the script completes run 
```
sudo clab version
```
Example output:
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

## XRd Docker Image

1. Acquire an XRd image...normally you'd download the image from CCO (and then upload it to your VM), but for the sake of time, we'll use an XRd image posted to Box. From your ssh session, run:

```
wget -O xrd-control-plane-container-x86.24.2.2.tgz -L https://cisco.box.com/shared/static/8ca2aa6cei6np3w04aan4ga1k7d58506
```

This may take a few minutes

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

## Containerlab Topology Definition
Containerlab uses a yaml file to define the topology, and it is enormously flexible.
For today's purposes we'll use the 7-node topology here: [topology.yaml](topology.yaml)

Note: the topology also includes a pair of Alpine linux containers (*carrots01 and carrots02*) that are users of the network

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
We can ignore the cgroups and Hugepages errors. 

One interesting portion of the output is the number of XRd control plane nodes the script estimates we can run on the VM:

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

## ssh to XRd Routers

1. Containerlab creates host entries in /etc/hosts for each node in the topology. We can use these to ssh into the routers:

```
ssh cisco@clab-topo-xrd01
or
ssh cisco@172.20.2.201
```
*Password is cisco123*

2. From xrd01 (or really any node), validate the network is working as expected:

```
show isis database
show bgp summary
show segment-routing srv6 sid
```

## Run Some Pings
The carrots linux containers need some ip config, then we'll run pings over the SRv6 network

1. Run this set of commands on the VM to give the containers ip addresses and routes:
   
```
docker exec -it clab-topo-carrots01 ip addr add 10.101.3.1/24 dev eth1
docker exec -it clab-topo-carrots01 ip route add 10.107.2.0/24 via 10.101.3.2
docker exec -it clab-topo-carrots01 ip route add 40.0.0.0/24 via 10.101.3.2
docker exec -it clab-topo-carrots01 ip route add 50.0.0.0/24 via 10.101.3.2

docker exec -it clab-topo-carrots02 ip addr add 10.107.2.1/24 dev eth1
docker exec -it clab-topo-carrots02 ip route add 10.101.3.0/24 via 10.107.2.2
```

2. Start a separate ssh session to the VM, and docker exec a tcpdump on xrd01:
```
docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-0
```

3. Back on the first ssh session, start a ping from carrots01 to carrots02:
```
docker exec -it clab-topo-carrots01 ping 10.107.2.1 -i .3 -c 10
```

### SRv6 L3VPN Reachability

The tcpdump should show the ICMP traffic captured over the SRv6 network; something like:
```
dcloud@server:~$ docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on Gi0-0-0-1, link-type EN10MB (Ethernet), capture size 262144 bytes
04:09:55.381957 IS-IS, p2p IIH, src-id 0000.0000.0001, length 1497
04:09:56.179006 IP6 fc00:0:1::1 > fc00:0:7:e005::: IP 10.101.3.1 > 10.107.2.1: ICMP echo request, id 58, seq 0, length 64
04:09:56.479012 IP6 fc00:0:1::1 > fc00:0:7:e005::: IP 10.101.3.1 > 10.107.2.1: ICMP echo request, id 58, seq 1, length 64
04:09:56.779201 IP6 fc00:0:1::1 > fc00:0:7:e005::: IP 10.101.3.1 > 10.107.2.1: ICMP echo request, id 58, seq 2, length 64
04:09:57.079676 IP6 fc00:0:1::1 > fc00:0:7:e005::: IP 10.101.3.1 > 10.107.2.1: ICMP echo request, id 58, seq 3, length 64
```

If you don't see any output, try switching the tcpdump to -ni Gi0-0-0-1
```
docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-1
```

Outbound traffic and return traffic may take separate paths through the network
```
dcloud@server:~$ docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on Gi0-0-0-0, link-type EN10MB (Ethernet), capture size 262144 bytes
04:11:25.293742 IS-IS, p2p IIH, src-id 0000.0000.0002, length 1497
04:11:26.459121 IP6 fc00:0:7::1 > fc00:0:1:e005::: IP 10.107.2.1 > 10.101.3.1: ICMP echo reply, id 64, seq 0, length 64
04:11:26.759005 IP6 fc00:0:7::1 > fc00:0:1:e005::: IP 10.107.2.1 > 10.101.3.1: ICMP echo reply, id 64, seq 1, length 64
04:11:27.059023 IP6 fc00:0:7::1 > fc00:0:1:e005::: IP 10.107.2.1 > 10.101.3.1: ICMP echo reply, id 64, seq 2, length 64
04:11:27.359603 IP6 fc00:0:7::1 > fc00:0:1:e005::: IP 10.107.2.1 > 10.101.3.1: ICMP echo reply, id 64, seq 3, length 64
```

### SRv6 TE Policies
xrd01 has a pair of SRv6 TE policies for carrots routes 40.0.0.0/24 and 50.0.0.0/24

Run pings from carrots01 to the 40.0.0.1 and 50.0.0.1 addresses and see the SRv6 uSID encapsualtions in your tcpdump:
```
docker exec -it clab-topo-carrots01 ping 40.0.0.1 -i .3 -c 10
docker exec -it clab-topo-carrots01 ping 50.0.0.1 -i .3 -c 10
```

tcpdump:
```
docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-0
```
```
docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-1
```

Example output:
```
dcloud@server:~$ docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on Gi0-0-0-0, link-type EN10MB (Ethernet), capture size 262144 bytes
04:17:13.139129 IS-IS, p2p IIH, src-id 0000.0000.0001, length 1497
04:17:13.300334 IS-IS, p2p IIH, src-id 0000.0000.0002, length 1497
04:17:13.824614 IP6 fc00:0:1::1 > fc00:0:2:3:7:e005::: IP 10.101.3.1 > 40.0.0.1: ICMP echo request, id 106, seq 0, length 64
04:17:14.124553 IP6 fc00:0:1::1 > fc00:0:2:3:7:e005::: IP 10.101.3.1 > 40.0.0.1: ICMP echo request, id 106, seq 1, length 64
04:17:14.424780 IP6 fc00:0:1::1 > fc00:0:2:3:7:e005::: IP 10.101.3.1 > 40.0.0.1: ICMP echo request, id 106, seq 2, length 64
```

And:
```
dcloud@server:~$ docker exec -it clab-topo-xrd01 tcpdump -ni Gi0-0-0-1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on Gi0-0-0-1, link-type EN10MB (Ethernet), capture size 262144 bytes
04:17:36.472404 IS-IS, p2p IIH, src-id 0000.0000.0005, length 1497
04:17:37.704762 IP6 fc00:0:1::1 > fc00:0:5:7:e005::: IP 10.101.3.1 > 50.0.0.1: ICMP echo request, id 124, seq 0, length 64
04:17:37.708602 IP6 fc00:0:7::1 > fc00:0:1:e005::: IP 50.0.0.1 > 10.101.3.1: ICMP echo reply, id 124, seq 0, length 64
04:17:38.005303 IP6 fc00:0:1::1 > fc00:0:5:7:e005::: IP 10.101.3.1 > 50.0.0.1: ICMP echo request, id 124, seq 1, length 64
04:17:38.009543 IP6 fc00:0:7::1 > fc00:0:1:e005::: IP 50.0.0.1 > 10.101.3.1: ICMP echo reply, id 124, seq 1, length 64
04:17:38.305255 IP6 fc00:0:1::1 > fc00:0:5:7:e005::: IP 10.101.3.1 > 50.0.0.1: ICMP echo request, id 124, seq 2, length 64
```


## Additional Resources

Lots of additional command output examples here:

https://github.com/jalapeno/SRv6_dCloud_Lab/blob/main/lab_3/validation-cmd-output.md

Lots of additional SRv6 Labs examples here:

https://github.com/segmentrouting/srv6-labs


## Appendix

Containerlab commands:
```
sudo clab -h
sudo clab deploy -t <topology yaml file>
sudo clab destroy -t <topology yaml file>
```

