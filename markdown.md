#  __------------=[ Elastic NSM Engineer ]=------------__

### __`[Configuring Elastic Containers]`__  
  ---

`lxc list`  
`lxc start --all`  

Connect to each containerized instance and begin configuration
```
Management Interfaces     | training    
--------------------------|-------------
ssh elastic@10.81.139.30  | elastic0    
ssh elastic@10.81.139.31  | elastic1    
ssh elastic@10.81.139.32  | elastic2    
ssh elastic@10.81.139.40  | pipeline0   
ssh elastic@10.81.139.41  | pipeline1   
ssh elastic@10.81.139.42  | pipeline2   
ssh elastic@10.81.139.50  | kibana      
ssh elastic@10.81.139.10  | repo        
ssh elastic@10.81.139.20  | sensor      
```

Edit the Hosts File:

`sudo vi /etc/hosts`
```
127.0.0.1 localhost
10.81.139.10 repo
10.81.139.20 sensor
10.81.139.30 elastic0
10.81.139.31 elastic1
10.81.139.32 elastic2
10.81.139.40 pipeline0
10.81.139.41 pipeline1
10.81.139.42 pipeline2
10.81.139.50 kibana
```

Disable IPV6:  

`sudo vim /etc/sysctl.conf` 
 ```
net.ipv6.conf.all.disable_ipv6=0
```

Edit Network Scripts

`sudo vim /etc/sysconfig/network-scripts/ifcf-eth0`

Configure the Networking


```
# [elastic]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=elastic0
NM_CONTROLLED=no
TYPE=Ethernet
DHCP_HOSTNAME=elastic0
IPADDR=10.81.139.30
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [pipeline]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=pipeline0
NM_CONTROLLED=no
TYPE=Ethernet
DHCP_HOSTNAME=pipeline0
IPADDR=10.81.139.40
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [repo]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=repo
NM_CONTROLLED=no
TYPE=Ethernet
DHCP_HOSTNAME=repo
IPADDR=10.81.139.10
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [kibana]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=kibana
NM_CONTROLLED=no
TYPE=Ethernet
DHCP_HOSTNAME=kibana
IPADDR=10.81.139.50
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [sensor]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=sensor
NM_CONTROLLED=no
TYPE=Ethernet
DHCP_HOSTNAME=sensor
IPADDR=10.81.139.20
GATEWAY=10.81.139.1
PREFIX=24
```

Restart Networking and Verify Container IP  

`sudo systemctl restart network`  
`lxc list`

Make the SSH Config 

`sudo vi ~/.ssh/config`

```
Host repo
  HostName repo
  User elastic
Host sensor
  HostName sensor
  User elastic
Host elastic0
  HostName elastic0
  User elastic
Host elastic1
  HostName elastic1
  User elastic
Host elastic2
  HostName elastic2
  User elastic
Host pipeline0
  HostName pipeline0
  User elastic
Host pipeline1
  HostName pipeline1
  User elastic
Host pipeline2
  HostName pipeline2
  User elastic
Host kibana
  HostName kibana
  User elastic
```
Generate a SSH Keypair

`ssh-keygen`

Copy SSH Keypair to Containers

`for host in sensor repo elastic{0..2} pipeline{0..2} kibana; do ssh-copy-id $host; done`

---

Configure Repo Server

`ssh repo`  
`sudo yum install nginx -y`  
`sudo unzip ~/all-class-files.zip -d /usr/share/nginx`  
`sudo mv /usr/share/nginx/all-class-files /usr/share/nginx/fileshare/`  
`sudo cp ~/emerging.rules.tar.gz /usr/share/nginx/fileshare/`  
`sudo cd /usr/share/nginx/fileshare/`    
`sudo vi /etc/nginx/conf.d/fileshare.conf`

```
server {
  listen 8000;
  location / {
    root /usr/share/nginx/fileshare;
    autoindex on;
    index index.html index.htm;
  }
}
```

`sudo firewall-cmd --add-port=8000/  tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo firewall-cmd --list-all`  
`sudo systemctl enable --now nginx`  
`sudo ss -lnt | grep 8000`  
`sudo yum install yum-utils -y`  
`cd /repo && ll`    
`sudo reposync -l --repoid=extras --download_path=/repo/local-extras`  
`sudo yum install createrepo -y`  
`sudo createrepo /repo/local-extras`  
`sudo vi /etc/nginx/conf.d/packages.conf`  

```
server {
  listen 8008;
  location / {
    root /repo;
    autoindex on;
    index index.html index.htm;
  }
}
```
`sudo firewall-cmd --add-port=8008/tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo firewall-cmd --list-all`  
`sudo systemctl restart nginx`  
`^restart^status`    

`exit`  


Verify repo accessibility by opening chrome and browsing to  
```
https://repo:8000 
https://repo:8008 
```
Configure the containers to pull from the repo  
`ssh sensor`  
`sudo yum list zeek`  
`mkdir ~/archive`  
`ll /etc/yum.repos.d`  
Copy the local repos to the archive directory  
`sudo mv /etc/yum.repos.d/* ~/archive/`  
`cd /etc/yum.repos.d/`  
`sudo curl -LO http://repo:8000/local.repo`  
`sudo vi /etc/yum.repos.d/local.repo` 
``` 
:%s/<hostname:port>/repo:8008/g
```
`sudo yum makecache fast`  
`exit`

Secure Repo and Fileshare by Creating a CA & Certs  
`ssh repo`  
`mkdir ~/certs`  
`cd ~/certs`
`openssl genrsa -des3 -out localCA.key 2048`  
`openssl req -x509 -new -nodes -key localCA.key -sha256 -days 1095 -out local.CA.crt`  
`openssl genrsa -out repo.key 2048`  
`openssl req -new -key repo.key -out repo.csr`  
`sudo vi ~/certs/repo.ext`  

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = repo
IP.1 = 10.81.139.10
```

`openssl x509 -req -in repo.csr -CA localCA.crt -CAkey localCA.key -CAcreateserial -out repo.crt -days 365 -sha256 -extfile repo.ext`  

Move the keys so Nginx has access to them  
`sudo mv repo.crt /etc/nginx/`  
`sudo mv repo.key /etc/nginx/`  

`cd /etc/nginx/conf.d/`  
`sudo curl -LO http://repo:8000/nginx/proxy.conf`  
`sudo vi /etc/nginx/nginx.conf`
```
:set nu
:39
```
Comment out the listener on port 80.      
```   
    38     server {
    39 #        listen       80;
    40 #        listen       [::]:80;
    41 #        server_name  _;
    42         root         /usr/share/nginx/html;
```

`sudo vi /etc/nginx/conf.d/packages.conf`  
```
listen 127.0.0.1:8008
```

`sudo vi /etc/nginx/conf.d/fileshare.conf`
```
listen 127.0.0.1:8000
```
Allow access through the firewall.  

`sudo firewall-cmd --add-port={80,443}/tcp --permanent`  
`sudo firewall-cmd --remove-port={8000,8008}/tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo firewall-cmd --list-all`  

Restart Nginx.  
`systemctl restart nginx`  
`^restart^status`  

