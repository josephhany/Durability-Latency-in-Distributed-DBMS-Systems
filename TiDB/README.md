

Load Bench: 
nohup tiup bench tpcc -H 192.168.152.110 -p ^F+4*91Gk8vTDC3r_5 -P 4000 -D tpcc --warehouses 1000 --threads 20 prepare &
tiup bench tpcc -H 192.168.152.110 -p ^F+4*91Gk8vTDC3r_5 -P 4000 -D tpcc --warehouses 1000 --threads 100 --time 30m run

CleanUp: 
tiup bench tpcc -H 192.168.152.110 -p ^F+4*91Gk8vTDC3r_5 -P 4000 -D tpcc --warehouses 4 cleanup


BenchBase:
mvn clean compile exec:java -P mysql -Dexec.args="-b tpcc -c /hdd1/haseeb/benchbaseFolder/benchbase/config/mysql/sample_tpcc_config.xml --create=true --load=true --execute=true"

ssh -L 7070:localhost:80281 h25ahmed@tembo.cs.uwaterloo.ca -t ssh -L 91931:localhost:4000 192.168.152.200

java -jar benchbase.jar -b tpcc -c config/mysql/sample_tpcc_config.xml --create=true --load=true --execute=true


./mvnw clean package -P mysql


Setup TiDB User Before Deploying: 

sudo useradd -m -d /hdd1/tidb_user tidb
sudo usermod -aG tidb tidb
sudo passwd tidb
sudo chown -R tidb:tidb /hdd1/haseeb/
sudo chown -R tidb:tidb /hdd1/tidb_user/

1. On all servers, make a new user tidb and a new group called tidb. 
2. Make a folder on hdd1, give permission to this user using 
    `sudo chown -R tidb:tidb <location> 
3. in topology.yaml set user as tidb
4. deploy using given instructions

systemctl daemon-reload && systemctl start tidb-4001.service


#Latency Setup 

sudo tc qdisc del dev eno1 root
sudo tc qdisc add dev eno1 root handle 1: prio
sudo tc qdisc add dev eno1 parent 1:1 handle 2: netem delay 90ms
sudo tc filter add dev eno1 parent 1:0 protocol ip pref 55 handle ::55 u32 match ip dst 192.168.152.225 flowid 2:1
sudo tc qdisc add dev eno1 parent 1:1 handle 3: netem delay 30ms
sudo tc filter add dev eno1 parent 1:0 protocol ip pref 56 handle ::56 u32 match ip dst 192.168.152.223 flowid 3:1


sudo tc qdisc add dev eno1 root handle 1: prio 
sudo tc qdisc add dev eno1 parent 1:1 handle 10: netem  delay 30ms
sudo tc qdisc add dev eno1 parent 1:2 handle 20: netem  delay 15ms

sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32  match ip dst 192.168.152.225 flowid 10:1
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 2 u32  match ip dst 192.168.152.224 flowid 20:2


sudo tc qdisc del dev eno1 root
sudo tc qdisc add dev eno1 root handle 1: prio 
sudo tc qdisc add dev eno1 parent 1:1 handle 10: netem delay 0ms 
sudo tc qdisc add dev eno1 parent 1:2 handle 20: netem  delay 30ms
sudo tc qdisc add dev eno1 parent 1:3 handle 30: netem  delay 30ms
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 2 u32 match ip dst 0.0.0.0/0 flowid 1:1
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.224 flowid 1:2
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.225 flowid 1:3



## Setup Cluster

tiup cluster deploy <Name> <version> <topology.yaml>
tiup cluster deploy final 7.1.2 NewTopo.yaml





# topology
# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/hdd1/haseeb/tidb-deploy/"
  data_dir: "/hdd1/haseeb/tidb-data/"

server_configs:
  #tikv:
    #readpool.unified.max-thread-count: <The value refers to the calculation formula result of the multi-instance topology document.>
    #readpool.storage.use-unified-pool: false
    #readpool.coprocessor.use-unified-pool: true
    #storage.block-cache.capacity: "<The value refers to the calculation formula result of the multi-instance topology document.>"
    #raftstore.capacity: "<The value refers to the calculation formula result of the multi-instance topology document.>"
  pd:
    replication.location-labels: ["host"]

pd_servers:
  - host: 192.168.252.192
  - host: 192.168.252.198
  - host: 192.168.252.199

tidb_servers:
  - host: 192.168.252.192
    port: 4000
    status_port: 10080
    numa_node: "0"
  - host: 192.168.252.198
    port: 4001
    status_port: 10081
    numa_node: "1"
  - host: 192.168.252.199
    port: 4000
    status_port: 10080
    numa_node: "2"

tikv_servers:
  - host: 192.168.252.192
    port: 20160
    status_port: 20180
    numa_node: "0"
    config:
      server.labels: { host: "tikv1" }
  - host: 192.168.252.198
    port: 20161
    status_port: 20181
    numa_node: "0"
    config:
      server.labels: { host: "tikv2" }
  - host: 192.168.252.199
    port: 20160
    status_port: 20180
    numa_node: "0"
    config:
      server.labels: { host: "tikv3" }

monitoring_servers:
  - host: 192.168.152.207

grafana_servers:
  - host: 192.168.152.207

alertmanager_servers:
  - host: 192.168.152.207
