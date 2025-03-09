### Part 2 Quick

```
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

```
sudo clab version
```

```
tar -xvf xrd-control-plane-container-x86.24.4.1.tgz
```

```
docker load -i xrd-control-plane-container-x64.dockerv1.tgz
```

```
git clone https://github.com/brmcdoug/byo-dcloud.git
```

```
cd byo-dcloud
```

```
sudo clab deploy -t topology.yaml
```



