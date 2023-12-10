To access a specific node in tembo cluster
```
ssh -t username@tembo.cs.uwaterloo.ca ssh tem#
```

On each node in each region execute the following script

```
sudo timedatectl set-ntp no
timedatectl
sudo apt-get install ntp
sudo service ntp stop
sudo ntpd -b time.google.com
sudo service ntp start
sudo ntpq -p

sudo iptables -A INPUT -p tcp --dport 26257 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo ufw allow 26257/tcp
sudo ufw allow 8080/tcp

cd ~/jboulis/

curl https://binaries.cockroachdb.com/cockroach-v23.1.12.linux-amd64.tgz | tar -xz
sudo cp -i cockroach-v23.1.12.linux-amd64/cockroach /usr/local/bin/

sudo mkdir -p /usr/local/lib/cockroach
sudo cp -i cockroach-v23.1.12.linux-amd64/lib/libgeos.so /usr/local/lib/cockroach/

sudo cp -i cockroach-v23.1.12.linux-amd64/lib/libgeos_c.so /usr/local/lib/cockroach/
which cockroach
```

## With a geo-distributed cluster in 3 regions with the follwoing configuration:

### Region 1 (US-West):
- tem08 (192.168.152.108)
- tem97 (192.168.152.197)
- tem100 (192.168.152.200)

### Region 2 (EU-West):
- tem09 (192.168.152.109)
- tem98 (192.168.152.198)
- tem101 (192.168.152.201)

### Region 3 (US-East):
- tem10 (192.168.152.110)
- tem99 (192.168.152.199)
- tem102 (192.168.152.202)

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

Now, to start CRDB on every node in the cluster execute the following script:

Note: Replace EXT with the rest of the IP address of the current node you're executing this script on.

```
cd /hdd1/

sudo cockroach start \
--insecure \
--advertise-addr=192.168.152.EXT \
--join=192.168.152.108,192.168.152.197,192.168.152.200,192.168.152.109,192.168.152.198,192.168.152.201,192.168.152.110,192.168.152.199,192.168.152.202 \
--cache=.25 \
--max-sql-memory=.25 \
--locality=region=us-west-2,zone=us-west-2a \
--background
```

Now, on the chosen node for proxy (let it be tem92), execute the following:

```
cd ~/jboulis/

sudo apt-get install haproxy
curl https://binaries.cockroachdb.com/cockroach-v23.1.12.linux-amd64.tgz | tar -xz
sudo cp -i cockroach-v23.1.12.linux-amd64/cockroach /usr/local/bin/

cockroach gen haproxy --insecure --host=192.168.152.200 --port=26257

haproxy -f haproxy.cfg
```


Now, on any node (let it be tem100), execute the follwoing:

```
cockroach init --insecure --host=192.168.152.200

cockroach sql --insecure --host=192.168.152.192

-- let the map know the location of the regions
UPSERT into system.locations VALUES
        ('region', 'us-east4', 37.478397, -76.453077),
        ('region', 'us-west2', 43.804133, -120.554201),
        ('region', 'eu-west2', 51.5073509, -0.1277583);

SET CLUSTER SETTING cluster.organization = "Cockroach Labs - Production Testing";
```

