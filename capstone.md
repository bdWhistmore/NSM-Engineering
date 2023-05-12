```
 _______ _                 _          ______     _    ______      ______                                       
(_______) |           _   (_)        |  ___ \   | |  |  ___ \    / _____)                _                     
 _____  | | ____  ___| |_  _  ____   | |   | |   \ \ | | _ | |  | /      ____ ____   ___| |_  ___  ____   ____ 
|  ___) | |/ _  |/___)  _)| |/ ___)  | |   | |    \ \| || || |  | |     / _  |  _ \ /___)  _)/ _ \|  _ \ / _  )
| |_____| ( ( | |___ | |__| ( (___   | |   | |_____) ) || || |  | \____( ( | | | | |___ | |_| |_| | | | ( (/ / 
|_______)_|\_||_(___/ \___)_|\____)  |_|   |_(______/|_||_||_|   \______)_||_| ||_/(___/ \___)___/|_| |_|\____)
                                                                             |_|                               
```

`lxc start --all`

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo lxc exec -t $host -- /bin/bash -c "vi /etc/sysconfig/network-scripts/ifcfg-eth0 && systemctl restart network"; done`


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

`lxc list`

`for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp /etc/hosts elastic@$host:~/hosts && ssh -t elastic@$host 'sudo mv ~/hosts /etc/hosts && sudo systemctl restart network'; done`   

`for host in sensor elastic{0..2} pipeline{0..2} kibana; do ssh-copy-id $host; done`

---

Push the created certificate from the local repository to all the containers

---

`ssh repo`  


`for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp ~/certs/localCA.crt elastic@$host:~/localCA.crt && ssh -t elastic@$host 'sudo mv ~/localCA.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust'; done`  


`exit`  
  

---

Configuring the Sensor to Utilize the Local Repository  

---

`ssh sensor`  
 
`mkdir ~/archive && sudo mv /etc/yum.repos.d/* ~/archive/ && sudo vi /etc/yum.repos.d/local.repo && sudo yum makecache fast` 

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
 

---

Configure the rest of the containers to pull from the local repository  

---


`cp /etc/yum.repos.d/local.repo ~/archive/ && for host in elastic{0..2} pipeline{0..2} kibana; do ssh -t elastic@$host 'sudo mkdir ~/archive && sudo mv /etc/yum.repos.d/* ~/archive/' && sudo scp /etc/yum.repos.d/local.repo elastic@$host:~/local.repo && ssh -t elastic@$host 'sudo mv ~/local.repo /etc/yum.repos.d/local.repo && sudo yum makecache fast' ; done`


---

Configuring the Sensor Monitor Interface

---

Install ethtool and its dependencies  
`sudo yum install ethtool -y && cd ~/ && sudo curl -LO https://repo/fileshare/interface.sh && sudo chmod +x interface.sh && sudo ./interface.sh eth1 && sudo vi /sbin/ifup-local`    
  
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

`sudo chmod +x /sbin/ifup-local && sudo vi /etc/sysconfig/network-scripts/ifup`  

```
if [ -x /sbin/ifup-local ]; then
/sbin/ifup-local #{DEVICE}
fi
```

Create the interface and verify changes  
`echo -e '# [monitor]\nDEVICE=eth1\nBOOTPROTO=none\nONBOOT=yes\nNM_CONTROLLED=no\nTYPE=Ethernet' | sudo tee /etc/sysconfig/network-scripts/ifcfg-eth1 >/dev/null && sudo systemctl restart network && ip a`  

---

Installing and Configuring Stenographer

---

`sudo yum install stenographer -y && cd /etc/stenographer && sudo vi config && sudo mkdir -p /data/stenographer/{index,packets} && sudo chown -R stenographer:stenographer /data/stenographer && sudo stenokeys.sh stenographer stenographer && sudo systemctl enable stenographer --now && sudo systemctl status stenographer`  
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


Verify stenographer is operating as intended    
`ping 8.8.8.8 -c 5 && clear && echo "waiting 60 seconds.." && sleep 60 && sudo stenoread 'host 8.8.8.8' -nn && ll /data/stenographer/{index,packets}`    

---

Installing and Configuring Suricata  

---

`sudo -s`


`sudo yum install suricata -y && cd /etc/suricata && sed -i '56s/.*/default-log-dir: \/data\/suricata/; 60s/.*/  enabled: no/; 76s/.*/      enabled: no/; 404s/.*/      enabled: no/; 557s/.*/      enabled: no/; 580s/.*/  - interface: eth1/; 582s/.*/    threads: 3/; 981s/.*/run-as:/; 982s/.*/  user:suricata/; 983s/.*/  group:suricata/; 1434s/.*/  set-cpu-affinity: yes/; 1452s/.*/        cpu: [ "0-2" ]/; 1459s/.*/          medium: [ 1 ]/; 1460s/.*/          high: [ 2 ]/; 1461s/.*/          default: "high"/; 1500s/.*/    enabled: no/; 1516s/.*/    enabled: no/; 1521s/.*/    enabled: no/; 1527s/.*/    enabled: no/; 1536s/.*/    enabled: no/' /etc/suricata/suricata.yaml && sed -i '8s/.*/OPTIONS="--af-packet=eth1 --user suricata --group suricata "/' /etc/sysconfig/suricata && sudo suricata-update add-source emergingthreats https://repo/fileshare/emerging.threats.tar.gz && sudo suricata-update && sudo mkdir -p /data/suricata && sudo chown -R suricata:suricata /data/suricata && sudo systemctl enable suricata --now && sudo systemctl status suricata && exit`  

Verify suricata is operating as intended  
`curl google.com && clear && echo "waiting 5 seconds" && sleep 5 && tail -n 5 /data/suricata/eve.json`

---

Installing and Configuring Zeek  

---

`sudo -s`

`sudo yum install zeek -y && sudo yum install zeek-plugin-af_packet -y && sudo yum install zeek-plugin-kafka -y && cd /etc/zeek && sudo sed -i '67s/.*/LogDir = \/data\/zeek/; 68s/.*/lb_custom.InterfacePrefix=af_packet::/' /etc/zeek/zeekctl.cfg && sudo mv /etc/zeek/node.cfg /etc/zeek/node.cfg.bk && echo -e '[logger]\ntype=logger\nhost=localhost\n\n[manager]\ntype=manager\nhost=localhost\npin_cpus=1\n\n[proxy-1]\ntype=proxy\nhost=localhost\n\n[worker-1]\ntype=worker\nhost=localhost\ninterface=eth1\nlb_method=custom\nlb_procs=2\npin_cpus=2,3\nenv_vars=fanout_id=77' | sudo tee /etc/zeek/node.cfg >/dev/null && sudo mkdir /usr/share/zeek/site/scripts && cd /usr/share/zeek/site/scripts && url_base="https://repo/fileshare/zeek/"; files=("afpacket.zeek" "extension.zeek" "extract-files.zeek" "fsf.zeek" "json.zeek" "kafka.zeek"); for file in "${files[@]}"; do sudo curl -LO "${url_base}${file}"; done && echo -e "\n\n\n\n\n\n\n\n\n\n" | sudo tee -a /usr/share/zeek/site/local.zeek > /dev/null && sudo sed -i.bak -e '104s|.*|@load ./scripts/afpacket.zeek|' -e '105s|.*|@load ./scripts/extension.zeek|' -e '113s|.*|redef ignore_checksums = T;|' /usr/share/zeek/site/local.zeek && sudo mkdir /data/zeek && sudo chown -R zeek:zeek /data/zeek && sudo chown -R zeek:zeek /etc/zeek && sudo chown -R zeek:zeek /usr/share/zeek && sudo chown -R zeek:zeek /usr/bin/zeek && sudo chown -R zeek:zeek /usr/bin/capstats && sudo chown -R zeek:zeek /var/spool/zeek && sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/zeek && sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/capstats && sudo getcap /usr/bin/zeek && sudo getcap /usr/bin/capstats && sudo -u zeek zeekctl deploy && sudo -u zeek zeekctl status && ll /data/zeek/current/ && exit`  


---

Installing and Configuring FSF

---
`sudo -s`


`sudo yum install fsf -y && sudo mv /opt/fsf/fsf-server/conf/config.py /opt/fsf/fsf-server/conf/config.py.bk && sudo vi /opt/fsf/fsf-server/conf/config.py`  

```
#!/usr/bin/env python
#
# Basic configuration attributes for scanner. Used as default
# unless the user overrides them.
#

import socket

SCANNER_CONFIG = { 'LOG_PATH' : '/data/fsf',
                   'YARA_PATH' : '/var/lib/yara-rules/rules.yara',
                   'PID_PATH' : '/run/fsf/fsf.pid',
                   'EXPORT_PATH' : '/data/fsf/archive',
                   'TIMEOUT' : 60,
                   'MAX_DEPTH' : 10,
                   'ACTIVE_LOGGING_MODULES' : ['rockout'],
                   }

SERVER_CONFIG = { 'IP_ADDRESS' : "localhost",
                  'PORT' : 5800 }
```

`sudo sed -i '0~2d' /opt/fsf/fsf-server/conf/config.py && sudo mkdir -p /data/fsf/archive && sudo chown -R fsf: /data/fsf && sudo mv /opt/fsf/fsf-client/conf/config.py /opt/fsf/fsf-client/conf/config.py.bk && sudo vi /opt/fsf/fsf-client/conf/config.py && sudo sed -i '0~2d' /opt/fsf/fsf-client/conf/config.py && sudo systemctl enable fsf --now && sudo systemctl status fsf && echo "" && echo "Seems good to me.. testing now" && sleep 5 && clear && /opt/fsf/fsf-client/fsf_client.py --full /home/elastic/interface.sh && ll /data/fsf/ && exit`

```
#!/usr/bin/env python
#
# Basic configuration attributes for scanner client.
#

# 'IP Address' is a list. It can contain one element, or more.
# If you put multiple FSF servers in, the one your client chooses will
# be done at random. A rudimentary way to distribute tasks.
SERVER_CONFIG = { 'IP_ADDRESS' : ['localhost'],
                  'PORT' : 5800 }

# Full path to debug file if run with --suppress-report
CLIENT_CONFIG = { 'LOG_FILE' : '/tmp/client_dbg.log' }
```

---

Editing Zeek to work with FSF  

---

`ssh sensor`  

`sudo sed -i -e '106s|.*|@load ./scripts/extract-files.zeek\n@load ./scripts/fsf.zeek\n@load ./scripts/json.zeek|' -e '107d' -e '108d' /usr/share/zeek/site/local.zeek && sudo -u zeek zeekctl deploy && sudo -u zeek zeekctl status && echo "Seems good to me, testing now...." && sleep 5 && clear && curl google.com && sleep 3 && cat /data/zeek/current/files.log | jq`

`exit`

---

Installing and Configuring Zookeeper as a Cluster  

---

Begin by accessing the pipeline0 container  
`ssh pipeline0`  
`ssh pipeline1`  
`ssh pipeline2`  

Create a unique id pipeline0  
`sudo -s`  

`sudo yum install kafka zookeeper -y && sudo mkdir -p /data/zookeeper && sudo echo '1' >> /data/zookeeper/myid && cat /data/zookeeper/myid && sudo chown -R zookeeper: /data/zookeeper && sudo mv /etc/zookeeper/zoo.cfg /etc/zookeeper/zoo.cfg.bk && sudo vi /etc/zookeeper/zoo.cfg && sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent && sudo firewall-cmd --reload && sudo systemctl enable zookeeper --now && sudo systemctl status zookeeper && exit`

Create a unique id on pipeline1  
`sudo -s`  

`sudo yum install kafka zookeeper -y && sudo mkdir -p /data/zookeeper && sudo echo '2' >> /data/zookeeper/myid && cat /data/zookeeper/myid && sudo chown -R zookeeper: /data/zookeeper && sudo mv /etc/zookeeper/zoo.cfg /etc/zookeeper/zoo.cfg.bk && sudo vi /etc/zookeeper/zoo.cfg && sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent && sudo firewall-cmd --reload && sudo systemctl enable zookeeper --now && sudo systemctl status zookeeper && exit`

Create a unique id pipeline2  

`sudo -s`  

`sudo yum install kafka zookeeper -y && sudo mkdir -p /data/zookeeper && sudo echo '3' >> /data/zookeeper/myid && cat /data/zookeeper/myid && sudo chown -R zookeeper: /data/zookeeper && sudo mv /etc/zookeeper/zoo.cfg /etc/zookeeper/zoo.cfg.bk && sudo vi /etc/zookeeper/zoo.cfg && sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent && sudo firewall-cmd --reload && sudo systemctl enable zookeeper --now && sudo systemctl status zookeeper && exit`
 

```
# where zookeeper will store its data
 dataDir=/data/zookeeper

 # what port should clients like kafka connect on
 clientPort=2181

 # how many clients should be allowed to connect, 0 = unlimited
 maxClientCnxns=0

 # list of zookeeper nodes to make up the cluster
 # First port is how followers and leaders communicate
 # Second port is used during the election process to determine a leader
 server.1=pipeline0:2888:3888
 server.2=pipeline1:2888:3888
 server.3=pipeline2:2888:3888

 # more than one zookeeper node will have a unique server id.
 # Ex. server.1, server.2, etc..

 # milliseconds in which zookeeper should consider a single tick
 tickTime=2000

 # amount of ticks a follow has to connect and sync with the leader
 initLimit=5

 # amount of ticks a follower has to sync with a leader before being dropped
 syncLimit=2
```

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
`sudo mkdir -p /data/kafka && sudo chown -R kafka: /data/kafka && sudo mv /etc/kafka/server{.properties,.properties.bk} && sudo vi /etc/kafka/server.properties && sudo firewall-cmd --add-port=9092/tcp --permanent && sudo firewall-cmd --reload && sudo systemctl enable kafka --now && sudo systemctl status kafka`  

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
```
# The unique id of this broker should be different for each kafka node. Good practice is to match the kafka broker id to the zookeeper server id.
broker.id=1

# the port in wich kafka should use to communicate with other kafka clients
port=9092
# the hostname or IP address in which the server listens on
listeners=PLAINTEXT://pipeline1:9092

# hostname that will be advertised to producers and consumers
advertised.listeners=PLAINTEXT://pipeline1:9092

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
```
# The unique id of this broker should be different for each kafka node. Good practice is to match the kafka broker id to the zookeeper server id.
broker.id=2

# the port in wich kafka should use to communicate with other kafka clients
port=9092
# the hostname or IP address in which the server listens on
listeners=PLAINTEXT://pipeline2:9092

# hostname that will be advertised to producers and consumers
advertised.listeners=PLAINTEXT://pipeline2:9092

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


Verify kafka is operating as intended on pipeline0  
`ssh pipeline0`  

`sudo /usr/share/kafka/bin/kafka-topics.sh --create --zookeeper pipeline0:2181 --replication-factor 3 --partitions 3 --topic zeek-raw && sudo /usr/share/kafka/bin/kafka-topics.sh --create --zookeeper pipeline0:2181 --replication-factor 3 --partitions 3 --topic fsf-raw && sudo /usr/share/kafka/bin/kafka-topics.sh --create --zookeeper pipeline0:2181 --replication-factor 3 --partitions 3 --topic suricata-raw && sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --list`  

`exit`

---

Verify Kafka and Zeek are working together

---

`ssh sensor`  

`sudo sed -i 's/<sensor-ip>:9092/pipeline0:9092,pipeline1:9092,pipeline2:9092/g' /usr/share/zeek/site/scripts/kafka.zeek && sudo sed -i '109s|.*|@load ./scripts/kafka.zeek|' /usr/share/zeek/site/local.zeek && sudo -u zeek zeekctl deploy && sudo -u zeek zeekctl status` 

Verify streaming from the pipeline  
`ssh pipeline0`  

`clear && echo "now checking zeek..." && sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic zeek-raw`  

---

Installing and Configuring Filebeat

---

`ssh sensor`  

Begin by installing the filebeat packages and dependencies  
`sudo yum install filebeat -y && sudo mv /etc/filebeat/filebeat{.yml,.yml.bk} && cd /etc/filebeat && sudo curl -LO https://repo/fileshare/filebeat/filebeat.yml && sudo sed -i 's|"localhost:9092"|"pipeline0:9092","pipeline1:9092","pipeline2:9092"|g' /etc/filebeat/filebeat.yml && sudo systemctl enable filebeat --now && sudo systemctl status filebeat`  

Verify messages are added to suricata-raw and fsf-raw on pipeline0  
`ssh pipeline0`  

`clear && echo "now checking suricata... " && sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic suricata-raw --from-beginning`  

`clear && echo "now checking fsf..." && sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic fsf-raw --from-beginning`  

---

Installing and Configuring Elasticsearch as a Cluster

---

`ssh elastic0`  
`ssh elastic1`  
`ssh elastic2`  

Begin by installing elasticsearch  
`sudo -s`  

`sudo yum install elasticsearch -y && sudo mv /etc/elasticsearch/elasticsearch{.yml,.yml.bk} && sudo vi /etc/elasticsearch/elasticsearch.yml`  

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

`sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch/ && sudo chmod 640 /etc/elasticsearch/elasticsearch.yml && sudo mkdir /usr/lib/systemd/system/elasticsearch.service.d && sudo chmod 755 /usr/lib/systemd/system/elasticsearch.service.d && echo -e "[Service]\nLimitMEMLOCK=infinity" | sudo tee /usr/lib/systemd/system/elasticsearch.service.d/override.conf && sudo chmod 644 /usr/lib/systemd/system/elasticsearch.service.d/override.conf && sudo systemctl daemon-reload && echo -e "-Xms2g\n-Xmx2g" | sudo tee /etc/elasticsearch/jvm.options.d/jvm_override.conf > /dev/null && sudo mkdir -p /data/elasticsearch && sudo chown -R elasticsearch: /data/elasticsearch && sudo chmod 755 /data/elasticsearch && sudo firewall-cmd --add-port={9200,9300}/tcp --permanent && sudo firewall-cmd --reload && sudo systemctl enable elasticsearch --now && sudo systemctl status elasticsearch && exit`  

Verify the cluster is operating as intended   
`exit`   
`curl elastic0:9200/_cat/nodes`  

---

Installing and Configuring Kibana

---

`ssh kibana`  

`sudo yum install kibana -y && sudo mv /etc/kibana/kibana{.yml,.yml.bk} && sudo vi /etc/kibana/kibana.yml`  

```
server.port: 5601
server.host: localhost
server.name: kibana
elasticsearch.hosts: ["http://elastic0:9200","http://elastic1:9200","http://elastic:9200"]
```

Install nginx for running a proxy on kibana  
`sudo yum install nginx -y && sudo vi /etc/nginx/conf.d/kibana.conf`  
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
`sudo firewall-cmd --add-port=80/tcp --permanent && sudo firewall-cmd --reload && sudo systemctl enable nginx --now && sudo systemctl status nginx && sudo systemctl enable kibana --now && sudo systemctl status kibana` 

Curl the ecs file from the local repository  
`sudo curl -LO https://repo/fileshare/kibana/ecskibana.tar.gz && tar -zxvf ecskibana.tar.gz && sudo yum install jq -y && cd ~/ecskibana/ && sudo ./import-index-templates.sh http://elastic0:9200`  


Verify kibana is operating as intended by browsing to kibana  
`exit`  

`http://kibana`

---

Installing and Configuring Logstash as a Cluster

---

`ssh pipeline0`  
`ssh pipeline1`  
`ssh pipeline2`  

Begin by installing logstash  

`sudo yum install logstash -y && cd ~/ && sudo curl -LO https://repo/fileshare/logstash/logstash.tar.gz && sudo tar -zxvf logstash.tar.gz -C /etc/logstash && cd /etc/logstash && sudo chown -R logstash: /etc/logstash && sudo chmod -R 744 /etc/logstash/conf.d/ruby/ && cd /etc/logstash/conf.d/ && sudo sed -i s/127.0.0.1:9092/pipeline0:9092,pipeline1:9092,pipeline2:9092/g logstash-100-input-kafka-{suricata,fsf,zeek}.conf && sudo sed -i 's/"127.0.0.1"/"elastic0", "elastic1", "elastic2"/g' logstash-9999-output-elasticsearch.conf && sudo -u logstash /usr/share/logstash/bin/logstash -t --path.settings /etc/logstash`  

Enable and start logstash  
`sudo systemctl enable logstash --now && sudo systemctl status logstash`  

`sudo tail -f /var/log/logstash/logstash-plain.log`  

Test that logstash is shipping logs to elastic by browsing to kibana  

`http://kibana/app/kibana#/management/kibana/index_pattern?_g=()`  


---

Creating the index patterns in Kibana

---

Navigate to kibana and create an index pattern in stack management  

```
http://kibana/app/kibana#/management/kibana/index_pattern?_g=()

ecs-suricata-*    @timestamp
ecs-zeek-*        @timestamp
fsf-*             @timestamp
ecs-*             @timestamp
```
