# Consensus

Consensus algorithms are a key component of large scale reliable distributed systems. They allow a group of machines to act as a logical unit and continue to operate in the presence of failures. The point of consensus is for all of the machines to agree on a shared state. For this they maintain a replicated append-only log.

## The Main Components of Raft

### Leader Election

Raft uses a strong form of leadership to simplify problem of maintaining replicated log. A leader is elected using randomized timeouts at startup and when the current leader becomes unreachable. At any given time a node is in one of three states: leader, follower, or candidate. Under nominal conditions there is one leader and all other servers are followers. Followers don't do anything. They simply respond to requests from the leader. The leader handles all requests from clients. If a follower receives a client request then it just forwards it to the leader.

The leader sends periodic (with a small random delay) heartbeats to its followers to let them know it is still up and functioning. If a follower doesn't hear from the leader after a certain time period, called the `election timeout` it enters a the third state called the candidate state. In this state is considers itself a candidate to become the leader and requests other members of the cluster to vote for it. If it receives votes from a majority of the cluster (including itself) then it transitions to the leader state and begins serving requests from clients and sending heartbeats to the other clusters members.

A new election starts a new `term`. The given leader will serve requests for the entire duration of the term and during that time there will be only the one leader. Some leader elections can result in a split vote and in that case the term will end with no leader and a new election and term will start. Because of the randomized timeouts split votes are very unlikely to repeat.

### Log Replication

### Persistence
