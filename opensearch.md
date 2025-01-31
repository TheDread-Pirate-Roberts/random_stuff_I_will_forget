# Opensearch on Ubuntu Server

I recently went through the process of setting up a dedicated homelab for automated trading and that required getting a performant opensearch service going. This doc will serve as a reference for me in the future or you reading this.

Check out the latest what the latest version of opensearch is [here](https://opensearch.org/docs/latest/install-and-configure/install-opensearch/tar/)

Download:
```
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.18.0/opensearch-2.18.0-linux-x64.tar.gz
```

Unzip:
```
tar -xvf opensearch-2.18.0-linux-x64.tar.gz
```

Opensearch/Elasticsearch sucks if it's swapping memory so turn that off:
```
sudo swapoff -a
```
Now really turn it off for good:
```
sudo vim /etc/fstab
# comment out this line ⬇️
#/swap.img      none    swap    sw      0       0
```
Restart the server and run htop, you should see 0 swap.

I forget what this did, but I added the following to `sudo vim /etc/sysctl.conf`
```
vm.swappiness=1
```

Then I allocated half my system memory to opensearch 🫨 `sudo vim ~/opensearch-2.18.0/config/jvm.options`
```
-Xms32g
-Xmx32g
```

I then added some settings `sudo vim ~/opensearch-2.18.0/config/opensearch.yml`
```
cluster.name: opensearch-cluster
node.name: node-1
discovery.type: single-node  # Since this is a single-node deployment

# Data and Logs (Ensure These Paths Exist and Have Permissions)
path.data: /mnt/opensearch-data  # Your 2TB NVMe
path.logs: /var/log/opensearch  # Logs stored separately

# Networking (Expose API if Needed)
network.host: 0.0.0.0  # Change to specific IP if you don’t want full exposure
http.port: 9200

# Security (Disable if Testing, Enable for Production)
plugins.security.disabled: true  # Disable OpenSearch security if you don’t need authentication

# Memory Optimization
bootstrap.memory_lock: true  # Prevents OS from swapping OpenSearch memory
indices.fielddata.cache.size: 40%  # Cache up to 40% of heap memory
indices.memory.index_buffer_size: 20%  # Use 20% of heap for indexing buffer
```

You should probably run a 2nd nvme for speed. Or not, whatever.

Find the drive `lsblk`
```
server@server:~/workspace/traderbros$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1                   259:0    0   1.8T  0 disk
├─nvme0n1p1               259:2    0   200M  0 part
└─nvme0n1p2               259:3    0   1.8T  0 part /mnt/opensearch-data
nvme1n1                   259:1    0 465.8G  0 disk
├─nvme1n1p1               259:4    0     1G  0 part /boot/efi
├─nvme1n1p2               259:5    0     2G  0 part /boot
└─nvme1n1p3               259:6    0 462.7G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 462.7G  0 lvm  /
```

Make a directory for the drive: `sudo mkdir -p /mnt/opensearch-data`

Then mount it! `sudo mount /dev/nvme0n1p2 /mnt/opensearch-data`

Set some file limits: `sudo vim /etc/security/limits.conf`
```
opensearch soft nofile 65536
opensearch hard nofile 65536
opensearch soft memlock unlimited
opensearch hard memlock unlimited
```

Now systemctl ftw: `sudo vim /etc/systemd/system/opensearch.service`
```
[Unit]
Description=OpenSearch Service
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=server
Group=server
ExecStart=/home/server/opensearch-2.18.0/bin/opensearch
Restart=always
RestartSec=5
LimitNOFILE=65536
LimitNPROC=4096
LimitMEMLOCK=infinity
WorkingDirectory=/home/server/opensearch-2.18.0
Environment=OPENSEARCH_JAVA_HOME=/home/server/opensearch-2.18.0/jdk
Environment=OPENSEARCH_HEAP_SIZE=32g

[Install]
WantedBy=multi-user.target
```
Restart systemctl: `sudo systemctl daemon-reload`

Enable service at startup: `sudo systemctl enable opensearch`

Create log directory on main drive: `sudo mkdir -p /var/log/opensearch`

Set drive permissions - *change user to your actual user*:
```
sudo chown -R user:user /var/log/opensearch
sudo chown -R user:user /mnt/opensearch-data
sudo chmod -R 775 /var/log/opensearch
```

Start the service `sudo systemctl start opensearch`

Is it live?: `sudo systemctl status opensearch`
```
● opensearch.service - OpenSearch Service
     Loaded: loaded (/etc/systemd/system/opensearch.service; enabled; preset: enabled)
     Active: active (running) since Fri 2025-01-31 03:21:09 UTC; 2h 16min ago
   Main PID: 945981 (java)
      Tasks: 133 (limit: 71971)
     Memory: 33.2G (peak: 33.2G)
        CPU: 1min 54.686s
     CGroup: /system.slice/opensearch.service
             └─945981 /home/server/opensearch-2.18.0/jdk/bin/java -Xshare:auto -Dopensearch.networkaddress.cache.ttl=60 -Dopensearch.networkaddress.cache.negative.ttl=10 -XX:+AlwaysPreTouch ->

```

Check the logs!: `journalctl -u opensearch -f`

Make a request!: 
```
curl --request GET   --url 'http://localhost:9200/_cat/indices?v='
```
