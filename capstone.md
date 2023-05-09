# Elastic NSM Engineer Capstone


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
`sudo scp elastic@repo:/home/elastic/certs/localCA.crt ~/localCA.crt`  

---

Configuring the Sensor to Utilize the Local Repository  

---

`ssh sensor`  
 
`mkdir ~/archive && sudo mv /etc/yum.repos.d/* ~/archive/ && sudo vi /etc/yum.repos.d/local.repo`    

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

`sudo yum makecache fast`  

---

Configure the rest of the containers to pull from the local repository  

---


`cp /etc/yum.repos.d/local.repo ~/archive/`

`for host in elastic{0..2} pipeline{0..2} kibana; do ssh -t elastic@$host 'sudo mkdir ~/archive && sudo mv /etc/yum.repos.d/* ~/archive/' && sudo scp /etc/yum.repos.d/local.repo elastic@$host:~/local.repo && ssh -t elastic@$host 'sudo mv ~/local.repo /etc/yum.repos.d/local.repo && sudo yum makecache fast' ; done`


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
`echo -e '# [monitor]\nDEVICE=eth1\nBOOTPROTO=none\nONBOOT=yes\nNM_CONTROLLED=no\nTYPE=Ethernet\n' | sudo tee /etc/sysconfig/network-scripts/ifcfg-eth1 >/dev/null && sudo systemctl restart network && ip a`  

---

Installing and Configuring Stenographer

---

`sudo yum install stenographer -y && cd /etc/stenographer && sudo vi config`  

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

`sudo mkdir -p /data/stenographer/{index,packets}`  
`sudo chown -R stenographer:stenographer /data/stenographer `
`sudo stenokeys.sh stenographer stenographer`  
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

`sudo yum install suricata -y && cd /etc/suricata`  

```
sed -i '56s/.*/default-log-dir: \/data\/suricata/; 60s/.*/  enabled: no/; 76s/.*/      enabled: no/; 404s/.*/      enabled: no/; 557s/.*/      enabled: no/; 580s/.*/  - interface: eth1/; 582s/.*/    threads: 3/; 981s/.*/run-as:/; 982s/.*/  user:suricata/; 983s/.*/  group:suricata/; 1434s/.*/  set-cpu-affinity: yes/; 1452s/.*/        cpu: [ "0-2" ]/; 1459s/.*/          medium: [ 1 ]/; 1460s/.*/          high: [ 2 ]/; 1461s/.*/          default: "high"/; 1500s/.*/    enabled: no/; 1516s/.*/    enabled: no/; 1521s/.*/    enabled: no/; 1527s/.*/    enabled: no/; 1536s/.*/    enabled: no/' /etc/suricata/suricata.yaml
```

`sed -i '8s/.*/OPTIONS="--af-packet=eth1 --user suricata --group suricata "/' /etc/sysconfig/suricata`  
`sudo suricata-update add-source emergingthreats https://repo/fileshare/emerging.threats.tar.gz` 
`sudo suricata-update`  
`sudo mkdir -p /data/suricata`  
`sudo chown -R suricata:suricata /data/suricata`  
`sudo systemctl enable suricata --now`   
`sudo systemctl status suricata`  

Verify suricata is operating as intended  
`curl google.com`  
`cat /data/suricata/eve.json`

---

Installing and Configuring Zeek  

---
`sudo yum install zeek -y && sudo yum install zeek-plugin-af_packet -y && sudo yum install zeek-plugin-kafka -y && cd /etc/zeek && sudo vi /etc/zeek/zeekctl.cfg`  
```
:set nu

:67     LogDir = /data/zeek  
:68     lb_custom.InterfacePrefix=af_packet::   #newline

``` 

`sudo mv /etc/zeek/node.cfg /etc/zeek/node.cfg.bk && sudo vi /etc/zeek/node.cfg`  

```
# Example ZeekControl node configuration.
#
# This example has a standalone node ready to go except for possibly changing
# the sniffing interface.

# This is a complete standalone configuration.  Most likely you will
# only need to change the interface.
#[zeek]
#type=standalone
#host=localhost
#interface=eth0

## Below is an example clustered configuration. If you use this,
## remove the [zeek] node above.

[logger]
type=logger
host=localhost

[manager]
type=manager
host=localhost
pin_cpus=1

[proxy-1]
type=proxy
host=localhost

[worker-1]
type=worker
host=localhost
interface=eth1
lb_method=custom
lb_procs=2
pin_cpus=2,3
env_vars=fanout_id=77
#
#[worker-2]
#type=worker
#host=localhost
#interface=eth0

```

`sudo mkdir /usr/share/zeek/site/scripts && cd /usr/share/zeek/site/scripts`   
`url_base="https://repo/fileshare/zeek/"; files=("afpacket.zeek" "extension.zeek" "extract-files.zeek" "fsf.zeek" "json.zeek" "kafka.zeek"); for file in "${files[@]}"; do sudo curl -LO "${url_base}${file}"; done`  
`sudo vi /usr/share/zeek/site/local.zeek` 

```
:set nu

:104 @load ./scripts/afpacket.zeek    #newline
:105 @load ./scripts/extension.zeek   #newline
:107 redef ignore_checksums = T;      #newline

```

`sudo mkdir /data/zeek && chown -R zeek:zeek /data/zeek && chown -R zeek:zeek /etc/zeek && chown -R zeek:zeek /usr/share/zeek && chown -R zeek:zeek /usr/bin/zeek && chown -R zeek:zeek /usr/bin/capstats && chown -R zeek:zeek /var/spool/zeek && sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/zeek && sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/capstats && sudo getcap /usr/bin/zeek && sudo getcap /usr/bin/capstats && sudo -u zeek zeekctl deploy && sudo -u zeek zeekctl status && ll /data/zeek/current/`  


---

Installing and Configuring FSF

---
 
`sudo yum install fsf -y && sudo vi /opt/fsf/fsf-server/conf/config.py`  

```
!/usr/bin/env python
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

Create the directories for fsf to access  
`sudo mkdir -p /data/fsf/archive && sudo chown -R fsf: /data/fsf &&sudo vi /opt/fsf/fsf-client/conf/config.py`  

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
`ssh pipeline1`  
`ssh pipeline2`  

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

Edit the firewall configuration to allow zookeeper on pipeline0,1,2  
`sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo systemctl enable zookeeper --now`

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
`sudo cp /etc/kafka/server{.properties,.properties.bk}` 
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

Installing and Configuring Elasticsearch as a Cluster

---

`ssh elastic0`  
`ssh elastic1`  
`ssh elastic2`  

Begin by installing elasticsearch  
`sudo yum install elasticsearch -y`  
`sudo mv /etc/elasticsearch/elasticsearch{.yml,.yml.bk}`  
`cd ~`  
`sudo curl -LO https://repo/fileshare/elasticsearch/elasticsearch.yml`   
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
`sudo mkdir /usr/lib/systemd/system/elasticsearch.service.d`  
`sudo chmod 755 /usr/lib/systemd/system/elasticsearch.service.d`  
`sudo vi /usr/lib/systemd/system/elasticsearch.service.d/override.conf`  

```
[Service]
LimitMEMLOCK=infinity
```

`sudo chmod 644 /usr/lib/systemd/system/elasticsearch.service.d/override.conf`  
`sudo systemctl daemon-reload`  
`sudo vi /etc/elasticsearch/jvm.options.d/jvm_override.conf`  

```
-Xms2g
-Xmx2g
```

Create and modify the data directory  
`sudo mkdir -p /data/elasticsearch`  
`sudo chown -R elasticsearch: /data/elasticsearch`  
`sudo chmod 755 /data/elasticsearch`  
`sudo firewall-cmd --add-port={9200,9300}/tcp --permanent`  
`sudo firewall-cmd --reload`  
`sudo systemctl enable elasticsearch --now`  

Verify the cluster is operating as intended  
`curl elastic0:9200/_cat/nodes`  

