⚙️ Environment Setup
🧩 Dependencies
Install the following required libraries:

bash
sudo apt-get update
sudo apt-get install libglib2.0-dev
sudo apt-get install libmsgpack-dev
sudo apt-get install libhiredis-dev
sudo apt-get install libboost-all-dev
sudo apt-get install libgtest-dev
🛠️ Install Go
Follow the steps below to install Go (reference: this blog post):

bash
wget -c https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local
export PATH=$PATH:/usr/local/go/bin
source ~/.profile
Install goreman:

bash
go install github.com/mattn/goreman@latest
If cloning fails due to network restrictions, manually download using VPN:

bash
mkdir -p $GOPATH/src/github.com/mattn
cd $GOPATH/src/github.com/mattn
git clone https://github.com/mattn/goreman.git
cd goreman
go env -w GOPROXY=https://goproxy.cn,direct
go install
🐞 Bug Investigation
Bug fixes were primarily identified via log inspection.

🐛 Bug #1: Node Crashing Due to Commit Index Check
In a 3-node cluster, one node repeatedly crashed. Logs indicated an inconsistency between the persisted commit index and the current log:

cpp
if (state.commit < raft_log_->committed_ || state.commit > raft_log_->last_index())
This check assumes that the commit index is always smaller than the current log, which is not true when snapshots are involved. Fix:

cpp
if (state.commit > raft_log_->last_index())
🐛 Bug #2: Nullptr Dereferencing
A simple null pointer dereference when accessing a progress pointer:

cpp
ProgressPtr progress = get_progress(node);
if (progress) {
  LOG_INFO("%lu restored progress of %lu [%s]", id_, node, progress->string().c_str());
} else {
  LOG_ERROR("Progress for %lu not found", node);
}
Previously it accessed the pointer without checking:

cpp
LOG_INFO("%lu restored progress of %lu [%s]", id_, node, get_progress(id_)->string().c_str());
✅ Verification Process
A 5-node Raft cluster is launched using goreman, and the synchronization mechanism is verified via Redis CLI.

🖥️ Launch Cluster
bash
# Ensure goreman is installed: go get github.com/mattn/goreman

node1: ./raft-kv/raft-kv --id 1 --cluster=127.0.0.1:12379,... --port 63791  
node2: ./raft-kv/raft-kv --id 2 --cluster=127.0.0.1:12379,... --port 63792  
node3: ./raft-kv/raft-kv --id 3 --cluster=127.0.0.1:12379,... --port 63793  
node4: ./raft-kv/raft-kv --id 4 --cluster=127.0.0.1:12379,... --port 63794  
node5: ./raft-kv/raft-kv --id 5 --cluster=127.0.0.1:12379,... --port 63795  
🚀 Run and Test
Start the cluster:

bash
goreman start
Connect via Redis CLI:

bash
redis-cli -p 63791
set mykey myvalue
Simulate node failure and recovery:

bash
goreman run stop node2
redis-cli -p 63792   # Should fail to connect
goreman run start node2
redis-cli -p 63792
get mykey            # Should return new-value after recovery
This verifies that the restarted node successfully re-synchronizes with the cluster after failure.
