
---

````markdown
# raft-kv

## Getting Started

### Build

```bash
mkdir -p raft-kv/build
cd raft-kv/build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j8
````

### Running a cluster

First install [goreman](https://github.com/mattn/goreman), which manages Procfile-based applications.

```bash
goreman start
```

### Test

Install [redis-cli](https://github.com/antirez/redis), a redis console client.

```bash
redis-cli -p 63791
127.0.0.1:63791> set mykey myvalue
OK
127.0.0.1:63791> get mykey
"myvalue"
```

Remove a node and replace the value to check cluster availability:

```bash
goreman run stop node2
redis-cli -p 63791
127.0.0.1:63791> set mykey new-value
OK
```

Bring the node back up and verify it recovers with the updated value:

```bash
redis-cli -p 63792
127.0.0.1:63792> KEYS *
1) "mykey"
127.0.0.1:63792> get mykey
"new-value"
```

### Benchmark

```bash
redis-benchmark -t set,get -n 100000 -p 63791
```

Example output:

```
====== SET ======
100000 requests completed in 1.35 seconds
73909.83 requests per second

====== GET ======
100000 requests completed in 0.95 seconds
105485.23 requests per second
```

---

## Dependencies

```bash
sudo apt-get install libglib2.0-dev
sudo apt-get install libmsgpack-dev 
sudo apt-get install libhiredis-dev 
sudo apt-get update
sudo apt-get install libboost-all-dev
sudo apt-get install libgtest-dev 
```

### Install Go and Goreman

```bash
wget -c https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local
export PATH=$PATH:/usr/local/go/bin
source ~/.profile

go install github.com/mattn/goreman@latest
```

If unable to clone due to access issues:

```bash
mkdir -p $GOPATH/src/github.com/mattn
cd $GOPATH/src/github.com/mattn
git clone https://github.com/mattn/goreman.git
cd goreman
go env -w GOPROXY=https://goproxy.cn,direct
go install
```

---

## Debugging Notes

### Bug #1

In a 3-node setup, one node repeatedly crashed. The log indicated an inconsistency between in-memory commit ID and persisted log ID:

```cpp
if (state.commit < raft_log_->committed_ || state.commit > raft_log_->last_index())
```

According to Raft theory, snapshots only keep older logs, so the check should be:

```cpp
if (state.commit > raft_log_->last_index())
```

### Bug #2

Crash due to nullptr dereferencing:

```cpp
ProgressPtr progress = get_progress(node);
if (progress) {
  LOG_INFO("%lu restored progress of %lu [%s]", id_, node, progress->string().c_str());
} else {
  LOG_ERROR("Progress for %lu not found", node);
}
```

---

## Validation

Cluster of 5 nodes:

```
node1: ./raft-kv/raft-kv --id 1 --cluster=127.0.0.1:12379,127.0.0.1:22379,127.0.0.1:32379,127.0.0.1:42379,127.0.0.1:52379 --port 63791
node2: ./raft-kv/raft-kv --id 2 --cluster=... --port 63792
node3: ./raft-kv/raft-kv --id 3 --cluster=... --port 63793
node4: ./raft-kv/raft-kv --id 4 --cluster=... --port 63794
node5: ./raft-kv/raft-kv --id 5 --cluster=... --port 63795
```

Test sequence:

```bash
goreman start                # Start 5-node cluster
redis-cli -p 63791          # Connect to node1
set mykey myvalue

goreman run stop node2      # Simulate node failure
set mykey new-value         # Update value from node1

goreman run start node2     # Restart failed node
redis-cli -p 63792
get mykey                   # Should return "new-value"
```

This validates that data is correctly synchronized after node recovery.

```

---

如需中文版本、添加简介或突出实现点（如日志模块/选举/快照等）可以随时告诉我。
```
