### terminal session
```
ssh dcloud@198.18.133.100
sudo adduser cisco
sudo adduser cisco sudo
ssh cisco@198.18.133.100
sudo hostnamectl set-hostname topology-host
exit
ssh cisco@198.18.133.100
mkdir images
git clone https://github.com/brmcdoug/byo-dcloud.git
```

#### separate terminal session
```
sftp cisco@198.18.133.100
cd images
put nexus9300v64.10.5.2.F.qcow2
```

#### original terminal session
Install docker
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
exit
ssh cisco@198.18.133.100
docker info
docker pull iejalapeno/alpine:latest
docker images

#### install containerlab and vrnetlab

bash -c "$(curl -sL https://get.containerlab.dev)"
sudo clab version
sudo clab inspect -a
git clone https://github.com/hellt/vrnetlab.git

ls vrnetlab/
cp images/nexus9300v64.10.5.2.F.qcow2 vrnetlab/n9kv/
cd vrnetlab/n9kv
more README.md 

#### install 'make'
sudo apt install make 
mv nexus9300v64.10.5.2.F.qcow2 n9kv-10.5.2.F.qcow2
make docker-image
docker images
cd ../../byo-dcloud/

#### launch topology

git pull
sudo clab deploy -t topology-nx.yaml
```

#### wait ~10min
We're looking for 'Healthy' Status 
```
docker ps
docker logs clab-demo-r00
```
