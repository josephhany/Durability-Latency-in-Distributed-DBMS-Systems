To access a specific node in tembo cluster
```
ssh -t username@tembo.cs.uwaterloo.ca ssh tem#
```

On each node in each region execute the following script

```
sudo apt install python-is-python3
wget https://downloads.yugabyte.com/releases/2.20.0.1/yugabyte-2.20.0.1-b1-linux-x86_64.tar.gz
tar xvfz yugabyte-2.20.0.1-b1-linux-x86_64.tar.gz && cd yugabyte-2.20.0.1/
./bin/post_install.sh
```

## With a geo-distributed cluster in 3 regions with the follwoing configuration:

### Region 1 (US-West):
- tem08 (192.168.152.108) (zone a)
- tem97 (192.168.152.197) (zone b)
- tem100 (192.168.152.200) (zone c)

### Region 2 (EU-West):
- tem09 (192.168.152.109) (zone a)
- tem98 (192.168.152.198) (zone b)
- tem101 (192.168.152.201) (zone c)

### Region 3 (US-East):
- tem10 (192.168.152.110) (zone a)
- tem99 (192.168.152.199) (zone b)
- tem102 (192.168.152.202) (zone c)

To simulate latencies between regions:

1) Execute the following script on all nodes in Region 1:

```
sudo tc qdisc del dev eno1 root
sudo tc qdisc add dev eno1 root handle 1: prio bands 7
sudo tc qdisc add dev eno1 parent 1:1 handle 10: netem delay 0ms 
sudo tc qdisc add dev eno1 parent 1:2 handle 20: netem  delay 10ms
sudo tc qdisc add dev eno1 parent 1:3 handle 30: netem  delay 10ms
sudo tc qdisc add dev eno1 parent 1:4 handle 40: netem  delay 10ms
sudo tc qdisc add dev eno1 parent 1:5 handle 50: netem  delay 10ms
sudo tc qdisc add dev eno1 parent 1:6 handle 60: netem  delay 10ms
sudo tc qdisc add dev eno1 parent 1:7 handle 70: netem  delay 10ms
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 2 u32 match ip dst 0.0.0.0/0 flowid 1:1
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.109 flowid 1:2
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.110 flowid 1:3
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.198 flowid 1:4
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.199 flowid 1:5
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.201 flowid 1:6
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.202 flowid 1:7
```

2) Execute the following script on all nodes in Region 1:

```
sudo tc qdisc del dev eno1 root
sudo tc qdisc add dev eno1 root handle 1: prio  bands 7
sudo tc qdisc add dev eno1 parent 1:1 handle 10: netem delay 0ms 
sudo tc qdisc add dev eno1 parent 1:2 handle 20: netem  delay 20ms
sudo tc qdisc add dev eno1 parent 1:3 handle 30: netem  delay 20ms
sudo tc qdisc add dev eno1 parent 1:4 handle 40: netem  delay 20ms
sudo tc qdisc add dev eno1 parent 1:5 handle 50: netem  delay 20ms
sudo tc qdisc add dev eno1 parent 1:6 handle 60: netem  delay 20ms
sudo tc qdisc add dev eno1 parent 1:7 handle 70: netem  delay 20ms
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 2 u32 match ip dst 0.0.0.0/0 flowid 1:1
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.108 flowid 1:2
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.109 flowid 1:3
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.197 flowid 1:4
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.198 flowid 1:5
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.200 flowid 1:6
sudo tc filter add dev eno1 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.152.201 flowid 1:7
```

Run the `yb-master` server on one nodes in each region as shown below. Note how multiple directories can be provided to the `--fs_data_dirs` flag. Replace the `--rpc_bind_addresses` value with the private IP address of the host as well as the set the `placement_cloud`,`placement_region` and `placement_zone` values appropriately.

```
sudo ./bin/yb-master \
  --master_addresses 192.168.152.108:7100,192.168.152.109:7100,192.168.152.110:7100 \
  --rpc_bind_addresses 192.168.152.108:7100 \
  --fs_data_dirs "/hdd1" \
  --placement_cloud tembo \
  --placement_region us-west \
  --placement_zone us-west-a \
  >& /hdd1/yb-master.out &
```

Make sure all the three YB-Masters are now working as expected by inspecting the INFO log. The default logs directory is always inside the first directory specified in the `--fs_data_dirs flag`
```
cat /hdd1/yb-data/master/logs/yb-master.INFO
```

Run the `yb-tserver` server on each of the six nodes as follows. Note that all of the master addresses have to be provided using the `--tserver_master_addrs` flag. Replace the `--rpc_bind_addresses` value with the private IP address of the host, and set the `placement_cloud`, `placement_region`, and `placement_zone` values appropriately.
```
sudo ./bin/yb-tserver \
  --tserver_master_addrs 192.168.152.108:7100,192.168.152.109:7100,192.168.152.110:7100 \
  --rpc_bind_addresses 192.168.152.108:9100 \
  --enable_ysql \
  --pgsql_proxy_bind_address 192.168.152.108:5433 \
  --cql_proxy_bind_address 192.168.152.108:9042 \
  --fs_data_dirs "/hdd1" \
  --placement_cloud tembo \
  --placement_region us-west \
  --placement_zone us-west-a \
  >& /hdd1/yb-tserver.out &

```

### Setting replica placement policy 
The default replica placement policy when the cluster is first created is to treat all nodes as equal irrespective of the `--placement_*` configuration flags. However, for the current deployment, we want to explicitly place one replica of each tablet in each region. The following command sets replication factor of 3 across `us-west`, `us-east`, and `eu-west` leading to such a placement.

On any host running the yb-master, run the following command:
```
sudo ./bin/yb-admin \
    --master_addresses 192.168.152.108:7100,192.168.152.109:7100,192.168.152.110:7100 \
    modify_placement_info  \
    tembo.us-west.us-west-a,tembo.eu-west.eu-west-a,tembo.us-east.us-east-c 3
```

and verify by running the following:
```
curl -s http://<any-master-ip>:7000/cluster-config
```

Confirm that the output looks similar to the following, with min_num_replicas set to 1 for each region - 
```
replication_info {
  live_replicas {
    num_replicas: 3
    placement_blocks {
      cloud_info {
        placement_cloud: "tembo"
        placement_region: "us-west"
        placement_zone: "us-west-a"
      }
      min_num_replicas: 1
    }
    placement_blocks {
      cloud_info {
        placement_cloud: "tembo"
        placement_region: "us-east"
        placement_zone: "us-east-a"
      }
      min_num_replicas: 1
    }
    placement_blocks {
      cloud_info {
        placement_cloud: "tembo"
        placement_region: "eu-west"
        placement_zone: "eu-west-a"
      }
      min_num_replicas: 1
    }
  }
}
```