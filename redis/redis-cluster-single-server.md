# Cài đặt Redis cluster (6.2) trên 1 server

Cài 6 nodes với 3 masters, 2 slaves

- 7000 (Master) - 7001 (Slave)
- 7002 (Master) - 7003 (Slave)
- 7004 (Master) - 7005 (Slave)

## Install redis

```bash
sudo apt install redis
```

## Chuẩn bị config và cài đặt

### Chuẩn bị 6 files config

```bash
sudo adduser redis
```

```bash
sudo vim /opt/redis_cluster_config.sh
```

Ghi vào file `redis_cluster_config.sh` với nội dung 

*Chú ý: cần bind IP ra ngoài để các client có thể kết nối, tránh lỗi connection refused*

```bash
CUR_DIR="/opt"
PASSWORD="password"
IP="10.10.10.105"
cd $CUR_DIR
mkdir -p ${CUR_DIR}/redis-cluster
cd ${CUR_DIR}/redis-cluster

for port in 7000 7001 7002 7003 7004 7005; do
  mkdir -p ${CUR_DIR}/redis-cluster/${port}
  cat > ${CUR_DIR}/redis-cluster/${port}/redis_${port}.conf <<EOF
port $port
appendonly no
cluster-enabled yes
cluster-node-timeout 5000
dir ${CUR_DIR}/redis-cluster/${port}/
cluster-config-file ${CUR_DIR}/redis-cluster/${port}/nodes_${port}.conf
logfile ${CUR_DIR}/redis-cluster/${port}/nodes_${port}.log
requirepass ${PASSWORD}
masterauth ${PASSWORD}
bind ${IP}
EOF
chown -R redis.redis /opt/redis-cluster
done
```

Chạy file `redis_cluster_config.sh`

```bash
sudo bash /opt/redis_cluster_config.sh
```

### Chuẩn bị 6 files systemd

```bash
sudo vim /opt/redis_cluster_systemd.sh
```

Ghi vào file `redis_cluster_systemd.sh` với nội dung 

*Chú ý địa chỉ IP là IP của server hiện tại*

```bash
CUR_DIR="/opt"
IP="10.10.10.105"
cd $CUR_DIR
for port in 7000 7001 7002 7003 7004 7005; do
  cat > /etc/systemd/system/redis_${port}.service <<EOF
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target


[Service]
ExecStart=$(which redis-server) ${CUR_DIR}/redis-cluster/${port}/redis_${port}.conf --supervised systemd
ExecStop=/bin/redis-cli -h ${IP} -p ${port} -a komatkhau shutdown
Type=notify
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
done
```

Chạy file `redis_cluster_systemd.sh`

```bash
sudo bash /opt/redis_cluster_systemd.sh
```

Restart các service

```bash
sudo systemctl daemon-reload
sudo systemctl restart redis_7001
sudo systemctl restart redis_7002
sudo systemctl restart redis_7003
sudo systemctl restart redis_7004
sudo systemctl restart redis_7005
sudo systemctl restart redis_7006
```

Kiểm tra các service đã được chạy 

```bash
ps aux | grep redis

redis      17727  0.0  0.5  64212 10668 ?        Ssl  06:13   0:06 /usr/bin/redis-server 127.0.0.1:6379
redis      19915  0.0  0.5  75988 11064 ?        Ssl  07:02   0:05 /usr/bin/redis-server 10.10.10.105:7000 [cluster]
redis      19924  0.0  0.5  66772 11128 ?        Ssl  07:02   0:04 /usr/bin/redis-server 10.10.10.105:7001 [cluster]
redis      19934  0.0  0.5  69844 11140 ?        Ssl  07:02   0:05 /usr/bin/redis-server 10.10.10.105:7002 [cluster]
redis      19943  0.0  0.5  66772 11104 ?        Ssl  07:02   0:04 /usr/bin/redis-server 10.10.10.105:7003 [cluster]
redis      19952  0.0  0.5  66772 11080 ?        Ssl  07:02   0:04 /usr/bin/redis-server 10.10.10.105:7004 [cluster]
redis      19961  0.0  0.5  66772 11172 ?        Ssl  07:02   0:04 /usr/bin/redis-server 10.10.10.105:7005 [cluster]
```

Join các nodes vào cluster, 3 nodes đầu là master, 3 node tiếp theo là slave

```bash
redis-cli -a ${PASSWORD} --cluster create 10.10.10.105:7000 10.10.10.105:7002 10.10.10.105:7004 10.10.10.105:7001 10.10.10.105:7003 10.10.10.105:7005 --cluster-replicas 1
```

## Kiểm tra 

### Kiểm tra nodes

```bash
redis-cli -a ${PASSWORD} -c -h 10.10.10.105 -p 7000 CLUSTER NODES

Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
487383b8dd8cb80a434651876e45cb0ecabe0276 10.10.10.105:7001@17001 slave d1eeee700d6536bab8a39c62b08837b1a87f8e01 0 1687249003582 1 connected
5d0c7150f21d488b7184feac65bb9ffdc7d294cf 10.10.10.105:7003@17003 slave e03e9db7c69edb9ed5acacd815f039a7bd7507fb 0 1687249002980 2 connected
d1eeee700d6536bab8a39c62b08837b1a87f8e01 10.10.10.105:7000@17000 myself,master - 0 1687249002000 1 connected 0-5460
e03e9db7c69edb9ed5acacd815f039a7bd7507fb 10.10.10.105:7002@17002 master - 0 1687249002000 2 connected 5461-10922
553d5a91bfd48a10e9986f60ed4fc09082c74f30 10.10.10.105:7004@17004 master - 0 1687249002580 3 connected 10923-16383
ea6bffadfb8479a9fc9c5050af2cb399f0ce624b 10.10.10.105:7005@17005 slave 553d5a91bfd48a10e9986f60ed4fc09082c74f30 0 1687249003000 3 connected
```