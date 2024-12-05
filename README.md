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
    ### Customer example/use case
	### Bell Canada
	### AI backend (or FE)

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




## XRd image
## topology yaml
## srv6 config
## launch topology
