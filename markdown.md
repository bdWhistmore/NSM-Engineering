# Elastic NSM Engineer

### Configuring LX Containers

---

`lxc list`  
`lxc start --all`

Connect to each containerized instance and begin configuration.

```
Container Name            IP Address    
---------------------------------------
ssh elastic@elastic0      10.81.139.30
ssh elastic@elastic1      10.81.139.31
ssh elastic@elastic2      10.81.139.32
ssh elastic@pipeline0     10.81.139.40
ssh elastic@pipeline1     10.81.139.41
ssh elastic@pipeline2     10.81.139.42
ssh elastic@kibana        10.81.139.50
ssh elastic@repo          10.81.139.10
ssh elastic@sensor        10.81.139.20     
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

---

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
`sudo ss -lnt`  
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

Move the local repos to the previously created archive directory  

`sudo mv /etc/yum.repos.d/* ~/archive/`  
`cd /etc/yum.repos.d/`  
`sudo curl -LO http://repo:8000/local.repo`  
`sudo vi /etc/yum.repos.d/local.repo` 
``` 
:%s/<hostname:port>/repo:8008/g
```
`sudo yum makecache fast`  
`exit`

---

Secure Repo and Fileshare by Creating a CA & Certs 

--- 
`ssh repo`  

Create a directory for the Certificates to be stored

`mkdir ~/certs`  
`cd ~/certs`

Generate a local CA Key with 2048 bits, using des3 encryption

`openssl genrsa -des3 -out localCA.key 2048`  

Generate a self-signed x509 certificate for the local CA that will be valid for 1095 days

`openssl req -x509 -new -nodes -key localCA.key -sha256 -days 1095 -out local.CA.crt`  

Generate a key for the repo

`openssl genrsa -out repo.key 2048`  

Generate a certificate signing request (CSR) for the repository

`openssl req -new -key repo.key -out repo.csr`  

Create a file named repo.ext in the ~/certs directory and add the following contents

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


Use the repo.ext file to sign the CSR and create a certificate for the repository 

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
    38     server {}
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
Allow access through the firewall  

`sudo firewall-cmd --add-port={80,443}/tcp --permanent`  
`sudo firewall-cmd --remove-port={8000,8008}/tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo firewall-cmd --list-all`  

Restart Nginx.  
`systemctl restart nginx`  
`^restart^status`  
`ss -lnt`  
`exit`

---
Configure the containers to pull from the local repository  

---

`ssh sensor`
`vi /etc/yum.repos.d/local.repo`  

Replace all instances of repo:8008 with https://repo/packages  

```
:%s/http:\/\/repo:8008\//https:\/\/repo\/packages\//g
```

The updated code should read:

```
[local-base]
name=local-base
baseurl=https://repo/packages/local-base/
enabled=1
gpgcheck=0

[local-rocknsm-2.5]
name=local-rocknsm-2.5
baseurl=https://repo/packages/local-rocknsm-2.5/
enabled=1
gpgcheck=0

[local-elasticsearch-7.x]
name=local-elasticsearch-7.x
baseurl=https://repo/packages/local-elastic-7.x/
enabled=1
gpgcheck=0

[local-epel]
name=local-epel
baseurl=https://repo/packages/local-epel/
enabled=1
gpgcheck=0

[local-extras]
name=local-extras
baseurl=https://repo/packages/local-extras/
enabled=1
gpgcheck=0

[local-updates]
name=local-updates
baseurl=https://repo/packages/local-updates/
enabled=1
gpgcheck=0
```
Copy the updated local.repo to the archive directory.

`cp /etc/yum.repos.d/local.repo ~/archive/`

For each host, backup the repos in the archive directory

`for host in elastic{0..2} pipeline{0..2} kibana; do scp -r ~/archive elastic@$host:/home/elastic ; done`

For each host, change the root password to circumvent permission issues

`for host in elastic{0..2} pipeline{0..2} kibana; do sudo passwd root ; done`

For each host, empty the repos.d folder by removing the default repos stored in /etc/yum.repos.d/

`for host in elastic{0..2} pipeline{0..2} kibana; do sudo rm -f /etc/yum.repos.d/* ; done`

For each host, scp the local.repo file from the sensor to the empty remote yum.repos.d directory

`for host in elastic{0..2} pipeline{0..2} kibana; do sudo scp /etc/yum.repos.d/local.repo root@$host:/etc/yum.repos.d/local.repo ; done`  

`exit`

Update the Nginx proxy configuration

`ssh repo`  

`sudo vi /etc/nginx/conf.d/proxy.conf`
```
     28   location /packages/ 
     29     proxy_pass http://127.0.0.1:8008/;
```
`sudo systemctl restart nginx`  

For each host, copy and move the Local CA Cert to the Trust Anchors Directory

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do scp ~/certs/localCA.crt root@$host:/etc/pki/ca-trust/source/anchors/localCA.crt ; done`

For each host, update the CA Trust with this command or manually

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do ssh elastic@$host 'sudo update-ca-trust' ; done`

For each host, remake the yum cache with this command or manually

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do ssh elastic@$host 'sudo yum makecache fast' ; done`

`exit`

Finally, copy the CRT from the repo into the host workstation

`sudo scp elastic@repo:/home/elastic/certs/localCA.crt ~/localCA.crt`

---

Configuring the Sensor Monitor Interface

---

Install ethtool and its dependencies  
`sudo yum install ethtool -y`  

Show interface informtion and begin configuring eth1 as the monitor interface  
`ip a`  
`sudo ethtool -k eth1`  

Pull the interface bash script from the repo
`sudo curl -LO https://repo/fileshare/interface.sh`  

```
#!/bin/bash

for var in $@
do
  echo "turning off offloading on $var"
  ethtool -K $var tso off gro off lro off gso off rx off tx off sg off rxvlan off txvlan off
  ethtool -N $var rx-flow-hash udp4 sdfn
  ethtool -N $var rx-flow-hash udp6 sdfn
  ethtool -C $var adaptive-rx off
  ethtool -C $var rx-usecs 1000
  ethtool -G $var rx 4096
done
exit 0
```
Make the file executable  
`sudo chmod +x interface.sh`  

Run the script.  
`sudo ./interface.sh eth1`  

Verify the changes were made  
`sudo ethtool -k eth1`

Run the script on startup  
`sudo vi /sbin/ifup-local`  

Update the file to add in the ethtool persistence

```
#!/bin/bash
if [[ "$1" == "eth1" ]]
then
for i in rx tx sg tso ufo gso gro lro rxvlan txvlan
do
/usr/sbin/ethtool -K $1 $i off
done
/usr/sbin/ethtool -N $1 rx-flow-hash udp4 sdfn
/usr/sbin/ethtool -N $1 rx-flow-hash udp6 sdfn
/usr/sbin/ethtool -n $1 rx-flow-hash udp6
/usr/sbin/ethtool -n $1 rx-flow-hash udp4
/usr/sbin/ethtool -C $1 rx-usecs 10
/usr/sbin/ethtool -C $1 adaptive-rx off
/usr/sbin/ethtool -G $1 rx 4096

/usr/sbin/ip link set dev $1 promisc on

fi
```

Make the ethtool persistence file executable  
`sudo chmod +x /sbin/ifup-local`  

Add a reference to the created script to run on startup  
`sudo vi /etc/sysconfig/network-scripts/ifup`  

```
if [ -x /sbin/ifup-local ]; then
/sbin/ifup-local #{DEVICE}
fi
```

Create and edit the eth1 interface script  
`sudo vi /etc/sysconfig/network-scripts/ifcfg-eth1`  
```
# [monitor]
DEVICE=eth1
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
TYPE=Ethernet
```

Reboot the networking interfaces and verify changes  
`sudo systemctl restart network`  
`sudo ethtool -k eth1`  
`ip a`  

Test the capture interface  
`sudo tcpdump -nn -i eth1`  
`sudo tcpdump -nn -i eth1 '!port 22'`
`exit`  

---

Installing & Configuring Stenographer

---

Install the stenographer package from the local repo
`ssh sensor`  
`sudo yum install stenographer -y`  
`sudo yum install which`  

Edit the configuration file for stenographer  
`cd /etc/stenographer`  
`sudo vi config`  

```
{
  "Threads": [
    { "PacketsDirectory": "/data/stenographer/packets"
    , "IndexDirectory": "/data/stenographer/index"
    , "MaxDirectoryFiles": 30000
    , "DiskFreePercentage": 30
    }
  ]
  , "StenotypePath": "/usr/bin/stenotype"
  , "Interface": "eth1"
  , "Port": 1234
  , "Host": "127.0.0.1"
  , "Flags": []
  , "CertPath": "/etc/stenographer/certs"
}

```

Create the data directories for data storage
`sudo mkdir -p /data/stenographer/{index,packets}`  

Change the ownership of the stenographer data directory to allow writing
`sudo chown -R stenographer:stenographer /data/stenographer `

Execute the stenokeys script to generate a key-pair
`sudo stenokeys.sh stenographer stenographer`  

Enable and start stenographer
`sudo systemctl enable stenographer --now`  
`sudo systemctl status stenographer`  

Verify stenographer is operating as intended  
`ping 8.8.8.8`  
`sudo stenoread 'host 8.8.8.8' -nn`
`ll /data/stenographer/{index,packets}`    

`exit`  

---

Installing and Configuring Suricata  

---

Install the suricata package from the local repo  
`ssh sensor`  
`sudo yum install suricata -y`  

Edit the configuration files for suricata  
`sudo -s`  
`cd /etc/suricata`  
`sudo vi suricata.yaml`  
```
:set nu
```
```
:56     default-log-dir: /data/suricata
:60     enabled: no
:76     enabled: no
:404    enabled: no
:557    enabled: no
:580    - interface: eth1
:582    threads: 3
:981    run-as:
:982    user:suricata
:983    group:suricata
:1434   set-cpu-affinity: yes
:1452   cpu: [ "0-2" ]
:1459   medium: [ 1 ]
:1460   high: [ 2 ]
:1461   default: "high"
:1500   enabled: no
:1516   enabled: no
:1521   enabled: no
:1527   enabled: no
:1536   enabled: no
```

Edit the sysconfig for suricata  
`sudo vi /etc/sysconfig/suricata`  
```
:8      OPTIONS="--af-packet=eth1 --user suricata --group suricata"
```

Point suricata to pull rulesets from the repo  
`sudo suricata-update add-source emergingthreats https://repo/fileshare/emerging.threats.tar.gz`

Update suricata to refresh the loaded ruleset  
`sudo suricata-update`  

Make the suricata data directories and assign permissions
`sudo mkdir -p /data/suricata`
`sudo chown -R suricata:suricata /data/suricata`  

Enable and start suricata
`sudo systemctl enable suricata --now`  
`sudo systemctl status suricata`

Verify suricata is operating as intended
`curl google.com`  
`cat /data/suricata/eve.json`

`exit`

---






