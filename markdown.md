

# Elastic NSM Engineer

Begin by listing the available containers  
`lxc list`  
`lxc start --all`

Connect to each containerized instance and begin network configuration.

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo lxc exec -t $host -- /bin/bash -c "vi /etc/sysconfig/network-scripts/ifcfg-eth0 && systemctl restart network"; done`

The following network configurations should be added

```
# [elastic0]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=elastic0
NM_CONTROLLED=no
TYPE=Ethernet
IPADDR=10.81.139.30
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [elastic1]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=elastic1
NM_CONTROLLED=no
TYPE=Ethernet
IPADDR=10.81.139.31
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [elastic2]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=elastic2
NM_CONTROLLED=no
TYPE=Ethernet
IPADDR=10.81.139.32
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [pipeline0]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=pipeline0
NM_CONTROLLED=no
TYPE=Ethernet
IPADDR=10.81.139.40
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [pipeline1]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=pipeline1
NM_CONTROLLED=no
TYPE=Ethernet
IPADDR=10.81.139.41
GATEWAY=10.81.139.1
PREFIX=24
```

```
# [pipeline2]
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=pipeline2
NM_CONTROLLED=no
TYPE=Ethernet
IPADDR=10.81.139.42
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
IPADDR=10.81.139.20
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
IPADDR=10.81.139.10
GATEWAY=10.81.139.1
PREFIX=24
```

Verify Container IP  

`lxc list`

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

Copy the edited hosts file to each container excluding repo

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp /etc/hosts elastic@$host:~/hosts && ssh -t elastic@$host 'sudo mv ~/hosts /etc/hosts && sudo systemctl restart network'; done`   

Create the SSH config  

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

Copy SSH Keypair to All Containers Except the Local Repository

`for host in sensor elastic{0..2} pipeline{0..2} kibana; do ssh-copy-id $host; done`

---

Create and Configure the Local Repository Server

---

`ssh repo`  

Install the nginx service  
`sudo yum install nginx -y`  

Unzip the local archive and save it to the nginx fileshare  
`sudo unzip ~/all-class-files.zip -d /usr/share/nginx`  
`sudo mv /usr/share/nginx/all-class-files /usr/share/nginx/fileshare/`

Rename the archive to fileshare  
`sudo cp ~/emerging.rules.tar.gz /usr/share/nginx/fileshare/`  

Edit the fileshare config file  
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

Allow the fileshare through the firewall  
`sudo firewall-cmd --add-port=8000/  tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo firewall-cmd --list-all`  

Enable and verify the nginx service  
`sudo systemctl enable --now nginx`  
`sudo ss -lnt`  

Install yum-utils to create a local repo  
`sudo yum install yum-utils -y`  
`cd /repo && ll`    
`sudo reposync -l --repoid=extras --download_path=/repo/local-extras`  
`sudo yum install createrepo -y`  
`sudo createrepo /repo/local-extras` 

Edit the nginx packages config  
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

Allow the packages index through the firewall  
`sudo firewall-cmd --add-port=8008/tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo firewall-cmd --list-all`  
`sudo systemctl restart nginx`  
`^restart^status`    
`exit`  


Verify repo accessibility by opening chrome and browsing to  

```
http://repo:8000 
http://repo:8008 
```

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

Comment out the listener on port 80. 

  
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

`exit` 

---

Update the Nginx proxy configuration

---

`ssh repo`  

`sudo vi /etc/nginx/conf.d/proxy.conf`
```
     28   location /packages/ 
     29     proxy_pass http://127.0.0.1:8008/;
```

Restart Nginx.  
`systemctl restart nginx`  
`^restart^status`  
`ss -lnt`  
`exit`


---

Push the created certificate from the local repository to all the containers

---

`ssh repo`  

For each host, copy the local cert from the repo home directory to all the containers and update the ca-trust

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp ~/certs/localCA.crt elastic@$host:~/localCA.crt && ssh -t elastic@$host 'sudo mv ~/localCA.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust'; done`  

Finally, copy the CRT from the repo into the host workstation

`exit`  
`sudo scp elastic@repo:/home/elastic/certs/localCA.crt ~/localCA.crt`  

---

Configuring the Sensor to Utilize the Local Repository  

---

`ssh sensor`  
`sudo yum list zeek`  
`mkdir ~/archive`  
`ll /etc/yum.repos.d`  

Move the local repos to the previously created archive directory  

`sudo mv /etc/yum.repos.d/* ~/archive/`  
`cd /etc/yum.repos.d/`  

Create the local.repo file
`sudo vi /etc/yum.repos.d/local.repo` 

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
Reload the yum cache

`sudo yum makecache fast`  


`exit`

---

Configure the rest of the containers to pull from the local repository  

---

Connect to the sensor to begin pushing the files to the other containers  

`ssh sensor`

Copy the updated local.repo to the archive directory.  

`cp /etc/yum.repos.d/local.repo ~/archive/`

For each host, move the files from the yum.repos.d directory to an archive, and then copy the local.repo file from the sensor's /etc/yum.repos.d/ to all the containers excluding the repo, and then update the yumcache to read the new files. make sure your local.repo file is updated in the sensors /etc/yum.repos.d directory if you want to use this command:

`for host in elastic{0..2} pipeline{0..2} kibana; do ssh -t elastic@$host 'sudo mkdir ~/archive && sudo mv /etc/yum.repos.d/* ~/archive/' && sudo scp /etc/yum.repos.d/local.repo elastic@$host:~/local.repo && ssh -t elastic@$host 'sudo mv ~/local.repo /etc/yum.repos.d/local.repo && sudo yum makecache fast' ; done`

`exit`

---

Configuring the Sensor Monitor Interface

---

Install ethtool and its dependencies  
`sudo yum install ethtool -y`  

Show interface informtion and begin configuring eth1 as the monitor interface  
`ip a`  
`sudo ethtool -k eth1`  

Pull the interface bash script from the repo  
`cd ~/`  
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
  
`echo -e '# [monitor]\nDEVICE=eth1\nBOOTPROTO=none\nONBOOT=yes\nNM_CONTROLLED=no\nTYPE=Ethernet\n' | sudo tee /etc/sysconfig/network-scripts/ifcfg-eth1 >/dev/null`  

Verify the file was created correctly  
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

Installing and Configuring Stenographer

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
`cd ~/`  
`sudo yum install suricata -y`  

Edit the configuration files for suricata  
`cd /etc/suricata`  

```
sed -i '56s/.*/default-log-dir: \/data\/suricata/; 60s/.*/  enabled: no/; 76s/.*/      enabled: no/; 404s/.*/      enabled: no/; 557s/.*/      enabled: no/; 580s/.*/  - interface: eth1/; 582s/.*/    threads: 3/; 981s/.*/run-as:/; 982s/.*/  user:suricata/; 983s/.*/  group:suricata/; 1434s/.*/  set-cpu-affinity: yes/; 1452s/.*/        cpu: [ "0-2" ]/; 1459s/.*/          medium: [ 1 ]/; 1460s/.*/          high: [ 2 ]/; 1461s/.*/          default: "high"/; 1500s/.*/    enabled: no/; 1516s/.*/    enabled: no/; 1521s/.*/    enabled: no/; 1527s/.*/    enabled: no/; 1536s/.*/    enabled: no/' /etc/suricata/suricata.yaml
```

Verify the following line numbers have changed  
`sudo vi /etc/suricata/suricata.yaml`  

```
:set nu

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
`sed -i '8s/.*/OPTIONS="--af-packet=eth1 --user suricata --group suricata "/' /etc/sysconfig/suricata`

Verify the file was correctly edited  
`sudo vi /etc/sysconfig/suricata`  
```
:8      OPTIONS="--af-packet=eth1 --user suricata --group suricata "
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

Installing and Configuring Zeek  

---

Install the Zeek package & Dependencies from the local repo  
`ssh sensor`  
`cd ~/`  
`sudo yum install zeek -y`  
`sudo yum install zeek-plugin-af_packet -y`  
`sudo yum install zeek-plugin-kafka -y`  

Edit the configuration files for Zeek  
`cd /etc/zeek`  
`sudo vi /etc/zeek/zeekctl.cfg`  
```
:set nu

:67     LogDir = /data/zeek  
:68     lb_custom.InterfacePrefix=af_packet::   #newline

``` 
`lscpu -e`  
`sudo vi /etc/zeek/node.cfg`  

```
:set nu

:8  #
:9  #
:10 #
:11 #

:16-31 [Remove #]

:23   pin_cpus=1                #newline
:32   interface=eth1
:33   lb_method=custom          #newline
:34   lb_procs=2                #newline
:35   pin_cpus=2,3              #newline
:36   env_vars=fanout_id=77     #newline

```

Continue creating and editing config files for zeek  
`sudo mkdir /usr/share/zeek/site/scripts`  
`cd /usr/share/zeek/site/scripts`  

Pull down extra configuration files from the local repo  
`url_base="https://repo/fileshare/zeek/"; files=("afpacket.zeek" "extension.zeek" "extract-files.zeek" "fsf.zeek" "json.zeek" "kafka.zeek"); for file in "${files[@]}"; do sudo curl -LO "${url_base}${file}"; done`  

Edit the local.zeek file  
`sudo vi /usr/share/zeek/site/local.zeek` 

```
:set nu

:104 @load ./scripts/afpacket.zeek    #newline
:105 @load ./scripts/extension.zeek   #newline
:107 redef ignore_checksums = T;      #newline

```

Continue creating and editing config files for zeek  
`sudo mkdir /data/zeek`  
`chown -R zeek:zeek /data/zeek`  
`chown -R zeek:zeek /etc/zeek`  
`chown -R zeek:zeek /usr/share/zeek`  
`chown -R zeek:zeek /usr/bin/zeek`  
`chown -R zeek:zeek /usr/bin/capstats`  
`chown -R zeek:zeek /var/spool/zeek`  

Set capabilities of the zeek user  
`sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/zeek`  
`sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/capstats`  
`sudo getcap /usr/bin/zeek`  
`sudo getcap /usr/bin/capstats`  

Deploy zeek as a zeek user  
`sudo -u zeek zeekctl deploy`  
`sudo -u zeek zeekctl status`

Verify zeek data collection  
`ll /data/zeek/current/`  
`exit`  

---

Installing and Configuring FSF

---

`ssh sensor`  

Install the fsf package from the local repo   
`sudo yum install fsf -y`

Edit the configuration files for fsf  
`sudo vi /opt/fsf/fsf-server/conf/config.py`  

```
:set nu

:9      'LOG_PATH' : '/data/fsf'
:10     'YARA_PATH' : '/var/lib/yara-rules/rules.yara
:11     'PID_PATH' : '/run/fsf/fsf.pid`
:12     'EXPORT_PATH' : '/data/fsf/archive'
:15     ['rockout]
:18     { 'IP_ADDRESS' : "localhost", 'PORT' : 5800 }

```

Create the directories for fsf to access  
`sudo mkdir -p /data/fsf/archive`

Change the permissions for the directories  
`sudo chown -R fsf: /data/fsf`

Edit the fsf client config  
`sudo vi /opt/fsf/fsf-client/conf/config.py`  

```
:set nu

:9      ['localhost',]
```

Enable and start the fsf service  
`sudo systemctl enable fsf --now`  
`sudo systemctl status fsf`  

Verify fsf service is running as intended  
`/opt/fsf/fsf-client/fsf_client.py --full ~/interface.sh`  
`ll /data/fsf/`


`exit`

---

Editing Zeek to work with FSF  

---

`ssh sensor`  

Edit the zeek config file to the additional fsf scripts  
`sudo vi /usr/share/zeek/site/local.zeek`  

```
:set nu

:106      @load ./scripts/extract-files.zeek
:107      @load ./scripts/fsf.zeek
:108      @load ./scripts/json.zeek
```

Stop zeek workers and redeploy  
`sudo -u zeek zeekctl stop`  
`sudo -u zeek zeekctl start`   
`sudo -u zeek zeekctl status`  

Verify the filescanning is loaded by zeek  
`curl google.com`  
`cat /data/zeek/current/files.log`
`exit`

---

Installing and Configuring Zookeeper as a Cluster  

---

Begin by accessing the pipeline0 container  
`ssh pipeline0`  

Install kafka and zookeeper    
`sudo yum install kafka zookeeper -y`  

Create the data directories and set ownership for zookeeper  
`sudo mkdir -p /data/zookeeper`  

Create a unique id pipeline0  
`sudo -s`    
`sudo echo '1' >> /data/zookeeper/myid`    
`exit`  
`cat /data/zookeeper/myid`

Create a unique id on pipeline1  
`sudo -s`  
`sudo echo '2' >> /data/zookeeper/myid`  
`exit`  
`cat /data/zookeeper/myid`  

Create a unique id pipeline2  
`sudo -s`  
`sudo echo '3' >> /data/zookeeper/myid`  
`exit`  
`cat /data/zookeeper/myid`

Set ownership for zookeeper  
`sudo chown -R zookeeper: /data/zookeeper` 

Edit the configuration files for zookeeper  
`sudo vi /etc/zookeeper/zoo.cfg`  

```
# The unique id of this broker should be different for each kafka node. Good practice is to match the kafka broker id to the zookeeper server id.
broker.id=0

# the port in wich kafka should use to communicate with other kafka clients
port=9092
# the hostname or IP address in which the server listens on
listeners=PLAINTEXT://pipeline0:9092

# hostname that will be advertised to producers and consumers
advertised.listeners=PLAINTEXT://pipeline0:9092

# number of threads used to send network responses
num.network.threads=3

# number of threads used to make I/O requests
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# where kafka should write its data to
log.dirs=/data/kafka

# how many partitions and replicas should be generated for topics that are created by other software
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
default.replication.factor = 3
min.insync.replicas = 2

# how many threads should be used for shutdown and start up
num.recovery.threads.per.data.dir=3

# how long should we retain logs in kafka
log.retention.hours=12
log.retention.bytes=90000000000

# max size of a single log file
log.segment.bytes=1073741824

# frequency in miliseconds to check if a log needs to be deleted
log.retention.check.interval.ms=300000
log.cleaner.enable=false

# will not allow a node to be elected leader if it is not in sync with other nodes. Prevents possible missing messages
unclean.leader.election.enable=false

# automatically create topics from external software
auto.create.topics.enable=false


# how to connect kafka to zookeeper
zookeeper.connect=pipeline0:2181,pipeline1:2181,pipeline2:2181
zookeeper.connection.timeout.ms=30000
```

Edit the firewall configuration to allow zookeeper  
`sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent`  
`sudo firewall-cmd --reload` 

Enable and start zookeeper on each node and verify cluster config  
`sudo systemctl enable zookeeper --now` #pipeline0  
`sudo systemctl enable zookeeper --now` #pipeline1  
`sudo systemctl enable zookeeper --now` #pipeline2  

Verify cluster config   
`exit`

`for host in pipeline{0..2}; do (echo "stats" | nc $host 2181 -q 2); done | grep -e 10.81.139.1 -e Clients -e Mode`

---

Installing and Configuring Kafka as a Cluster  

---
Begin by accessing the pipeline containers  
`ssh pipeline0`  
`ssh pipeline1`  
`ssh pipeline2`  

Create the data directories and set ownership for kafka    
`sudo mkdir -p /data/kafka`  
`sudo chown -R kafka: /data/kafka` 

Create a backup of the server properties file  
`sudo cp /etc/kafka/server{.properties,.properties.bk}`

Edit the server properties file   
`sudo vi /etc/kafka/server.properties`  

```
# The unique id of this broker should be different for each kafka node. Good practice is to match the kafka broker id to the zookeeper server id.
broker.id=0

# the port in wich kafka should use to communicate with other kafka clients
port=9092
# the hostname or IP address in which the server listens on
listeners=PLAINTEXT://pipeline0:9092

# hostname that will be advertised to producers and consumers
advertised.listeners=PLAINTEXT://pipeline0:9092

# number of threads used to send network responses
num.network.threads=3

# number of threads used to make I/O requests
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# where kafka should write its data to
log.dirs=/data/kafka

# how many partitions and replicas should be generated for topics that are created by other software
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
default.replication.factor = 3
min.insync.replicas = 2

# how many threads should be used for shutdown and start up
num.recovery.threads.per.data.dir=3

# how long should we retain logs in kafka
log.retention.hours=12
log.retention.bytes=90000000000

# max size of a single log file
log.segment.bytes=1073741824

# frequency in miliseconds to check if a log needs to be deleted
log.retention.check.interval.ms=300000
log.cleaner.enable=false

# will not allow a node to be elected leader if it is not in sync with other nodes. Prevents possible missing messages
unclean.leader.election.enable=false

# automatically create topics from external software
auto.create.topics.enable=false


# how to connect kafka to zookeeper
zookeeper.connect=pipeline0:2181,pipeline1:2181,pipeline2:2181
zookeeper.connection.timeout.ms=30000
```

Make a few changes to each file  

```

:3  broker.id=0 #(pipeline0)
:3  broker.id=1 #(pipeline1)
:3  broker.id=2 #(pipeline2)

:13 pipeline0   #(pipeline0)
:13 pipeline1   #(pipeline1)
:13 pipeline2   #(pipeline2)

:19 pipeline0   #(pipeline0)
:19 pipeline1   #(pipeline1)
:19 pipeline2   #(pipeline2)


```
Allow kafka through the firewall  
`sudo firewall-cmd --add-port=9092/tcp --permanent`  
`sudo firewall-cmd --reload`  

Enable and start kafka  
`sudo systemctl enable kafka --now`

Verify kafka is operating as intended on pipeline0  
`ssh pipeline0`  

`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic test --partitions 3 --replication-factor 3`

`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic test`

Delete the test topic and verify topic was deleted  

`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --delete --topic test`

`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --list`

Create the zeek raw topic  
`sudo /usr/share/kafka/bin/kafka-topics.sh --create --zookeeper pipeline0:2181 --replication-factor 3 --partitions 3 --topic zeek-raw`  

`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic zeek-raw`  

`exit`

---

Verify Kafka and Zeek are working together

---

`ssh sensor`  

Edit the kafka script that will be loaded by zeek  
`sudo vi /usr/share/zeek/site/scripts/kafka.zeek`

```
:set nu

:7 "pipeline0:9092,pipeline1:9092,pipeline2:9092"
```

`sudo vi /usr/share/zeek/site/local.zeek`  

```
:set nu

:109      @load ./scripts/kafka.zeek
```

`sudo -u zeek zeekctl deploy`  
`sudo -u zeek zeekctl status` 

`exit`  

Verify streaming from the pipeline  
`sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic zeek-raw`  

Generate traffic on the sensor and verify on pipeline0  
`curl google.com`  

---

Installing and Configuring Filebeat

---

`ssh sensor`  

Begin by installing the filebeat packages and dependencies  
`sudo yum install filebeat -y`  

Create a backup of the filebeat configuration  
`sudo mv /etc/filebeat/filebeat{.yml,.yml.bk}`  

Curl the filebeat configuration from the local repository  
`cd /etc/filebeat`  
`sudo curl -LO https://repo/fileshare/filebeat/filebeat.yml`  

Edit the filebeat configuration file  
`sudo vi filebeat.yml`  

```
:set nu

:34       hosts: ["pipeline0:9092,"pipeline1:9092","pipeline2:9092"]
```

Create the extra kafka topics to accept fsf-raw and suricata-raw  
`ssh pipeline0`  

`sudo /usr/share/kafka/bin/kafka-topics.sh --create --zookeeper pipeline0:2181 --replication-factor 3 --partitions 3 --topic fsf-raw`  

`sudo /usr/share/kafka/bin/kafka-topics.sh --create --zookeeper pipeline0:2181 --replication-factor 3 --partitions 3 --topic suricata-raw`

List the current kafka topics  
`sudo /usr/share/kafka/bin/kafka-topics.sh --list --zookeeper pipeline0:2181`  

Enable and start filebeat on the sensor  
`ssh sensor`  
`systemctl enable filebeat --now`  

Verify messages are added to suricata-raw and fsf-raw on pipeline0  
`ssh pipeline0`  

`sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic suricata-raw --from-beginning`

`sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic fsf-raw --from-beginning`

---

Installing and Configuring Elasticsearch as Single Node

---

`ssh elastic0`   

Begin by installing elasticsearch  
`sudo yum install elasticsearch -y`  

Create a backup of the elasticsearch configuration file  
`sudo mv /etc/elasticsearch/elasticsearch{.yml,.yml.bk}`  

Curl the elasticsearch configuration file from the local repository  
`cd ~`  
`sudo curl -LO https://repo/fileshare/elasticsearch/elasticsearch.yml`  

Edit the downloaded configuration file  
`sudo vi elasticsearch.yml`  
```
:set nu

node.name:  sensor1
path.data: /data/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
http.port:9200
network.host: _local:ipv4_
discovery.type single-node

```

Move the edited config to the elastic directory and change the perms
`sudo mv ~/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml`  

`sudo chmod 640 /etc/elasticsearch/elasticsearch.yml`  

Create a memory override file to cease memoryswapping by elastic  
`sudo mkdir /usr/lib/systemd/system/elasticsearch.service.d`  

`sudo chmod 755 /usr/lib/systemd/system/elasticsearch.service.d`

Create and edit the memory override file  
`sudo vi /usr/lib/systemd/system/elasticsearch.service.d/override.conf`

```
[Service]
LimitMEMLOCK=infinity
```

Change the permissions on the override config file  
`sudo chmod 644 /usr/lib/systemd/system/elasticsearch.service.d/override.conf`

`sudo systemctl daemon-reload`  

Create and edit the configuration file for the elastic heap size  
`sudo vi /etc/elasticsearch/jvm.options.d/jvm_override.conf`  

```
-Xms2g
-Xmx2g
```

Create and modify the data directory  
`sudo mkdir -p /data/elasticsearch`  
`sudo chown -R elasticsearch: /data/elasticsearch`  
`sudo chmod 755 /data/elasticsearch`  

Edit the firewall configuration to allow elasticsearch  
`sudo firewall-cmd --add-port={9200,9300}/tcp --permanent`  
`sudo firewall-cmd --reload`  

Enable and start elasticsearch  
`sudo systemctl enable elasticsearch --now`  

---

Installing and Configuring Elasticsearch as a Cluster

---

`ssh elastic0`  
`ssh elastic1`  
`ssh elastic2`  

Begin by installing elasticsearch  
`sudo yum install elasticsearch -y`  

Create a backup of the elasticsearch configuration file  
`sudo mv /etc/elasticsearch/elasticsearch{.yml,.yml.bk}`  

Curl the elasticsearch configuration file from the local repository  
`cd ~`  
`sudo curl -LO https://repo/fileshare/elasticsearch/elasticsearch.yml`  

Edit the downloaded configuration file  
`sudo vi elasticsearch.yml`  

```
cluster.name:  nsm-cluster
node.name:  es-node-0
path.data: /data/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: _site:ipv4_
http.port: 9200
discovery.seed_hosts: ["elastic0","elastic1","elastic2"]
cluster.initial_master_nodes: ["es-node-0","es-node-1","es-node-2"]
```
```
cluster.name:  nsm-cluster
node.name:  es-node-1
path.data: /data/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: _site:ipv4_
http.port: 9200
discovery.seed_hosts: ["elastic0","elastic1","elastic2"]
cluster.initial_master_nodes: ["es-node-0","es-node-1","es-node-2"]
```
```
cluster.name:  nsm-cluster
node.name:  es-node-2
path.data: /data/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: _site:ipv4_
http.port: 9200
discovery.seed_hosts: ["elastic0","elastic1","elastic2"]
cluster.initial_master_nodes: ["es-node-0","es-node-1","es-node-2"]
```

Move the edited config to the elastic directory and change the perms  
`sudo mv ~/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml`  
`sudo chown -R elasticsearch: /etc/elasticsearch/`  
`sudo chmod 640 /etc/elasticsearch/elasticsearch.yml`  

Create a memory override file to cease memoryswapping by elastic  
`sudo mkdir /usr/lib/systemd/system/elasticsearch.service.d`  

`sudo chmod 755 /usr/lib/systemd/system/elasticsearch.service.d`

Create and edit the memory override file  
`sudo vi /usr/lib/systemd/system/elasticsearch.service.d/override.conf`

```
[Service]
LimitMEMLOCK=infinity
```

Change the permissions on the override config file  
`sudo chmod 644 /usr/lib/systemd/system/elasticsearch.service.d/override.conf`

`sudo systemctl daemon-reload`  

Create and edit the configuration file for the elastic heap size  
`sudo vi /etc/elasticsearch/jvm.options.d/jvm_override.conf`  

```
-Xms2g
-Xmx2g
```

Create and modify the data directory  
`sudo mkdir -p /data/elasticsearch`  
`sudo chown -R elasticsearch: /data/elasticsearch`  
`sudo chmod 755 /data/elasticsearch`  

Edit the firewall configuration to allow elasticsearch  
`sudo firewall-cmd --add-port={9200,9300}/tcp --permanent`  
`sudo firewall-cmd --reload`  

Enable and start elasticsearch  
`sudo systemctl enable elasticsearch --now`  

Verify the cluster is operating as intended  
`curl elastic0:9200/_cat/nodes`  

---

Installing and Configuring Kibana

---

`ssh kibana`  

Begin by installing kibana  
`sudo yum install kibana -y`  

Create a backup of the configuration file  
`sudo mv /etc/kibana/kibana{.yml,.yml.bk}`  

Edit the configuration file for kibana  
`sudo vi /etc/kibana/kibana.yml`  
```
server.port: 5601
server.host: localhost
server.name: kibana
elasticsearch.hosts: ["http://elastic0:9200","http://elastic1:9200","http://elastic:9200"]
```

Install nginx for running a proxy on kibana  
`sudo yum install nginx -y`  

Edit the nginx config file to add kibana  
`sudo vi /etc/nginx/conf.d/kibana.conf`  
```
server {
  listen 80;
  server_name kibana;
  proxy_max_temp_file_size 0;

  location / {
    proxy_pass http://127.0.0.1:5601/;

    proxy_redirect off;
    proxy_buffering off;

    proxy_http_version 1.1;
    proxy_set_header Connection "Keep-Alive";
    proxy_set_header Proxy-Connection "Keep-Alive";

  }

}
```

Edit the nginx config and unload the default server  
`sudo vi /etc/nginx/nginx.conf`  
```
:39     #
:40     #
:41     #
```

Edit the firewall config to allow the newly created server through  
`sudo firewall-cmd --add-port=80/tcp --permanent`  
`sudo firewall-cmd --reload`  

Enable and start nginx  
`sudo systemctl enable nginx --now`  
`sudo systemctl status nginx`  
`sudo systemctl enable kibana --now`  
`sudo systemctl status kibana` 

Curl the ecs file from the local repository  
`sudo curl -LO https://repo/fileshare/kibana/ecskibana.tar.gz`  

Uncompress the archive  
`tar -zxvf ecskibana.tar.gz`  

Install jq  
`sudo yum install jq -y`  

Import the index templates  
`sudo ./import-index-templates.sh http://elastic0:9200`  




Verify kibana is operating as intended by browsing to kibana  
`exit`  

`http://kibana`

---

Installing and Configuring Logstash

---