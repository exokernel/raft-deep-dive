# Raft Intro

Raft is a consensus algorithm. It's used in distributed systems to reliably and consistently replicate state between multiple nodes using an append-only log.  When raft nodes start up they first vote to elect a leader. After a leader is elected all application state writes go to the leader and the other nodes become followers. The leader sends all writes to the followers for replication and once a majority of the nodes have saved the write to their log it is considered committed and the leader responds to the writing client with success.

Raft guarantees that the log is always consistent among the leader and followers. It also designed for fault tolerance. Nodes may crash or restart (even the leader) or the system may experience network partitions. The leader maintains contact with the followers using periodic heartbeats. If there is a problem communicating with the leader the followers don't receive these heartbeats and they will elect a new leader and the system will continue operating.

Raft was designed to be easy to understand and implement. It's broken down into subproblems that function more or less independently: leader election, log replication, and cluster membership changes.
