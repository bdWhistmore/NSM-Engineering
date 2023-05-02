#  __------------=[ Elastic NSM Engineer ]=------------__

### __`[Configuring Elastic Containers]`__  
  ---

`lxc list`  
`lxc start --all`  

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

Copy SSH Key to Containers

`for host in sensor repo elastic{0..2} pipeline{0..2} kibana; do ssh-copy-id $host; done`

Configure Repo Server

`sudo yum install nginx -y`  
`sudo unzip ~/allclassfiles.zip -d /usr/share/nginx`  
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
`sudo ss -lnt`  
`exit`  


Verify repo accessibility by opening chrome and browsing to  
```
https://repo:8000  
```