# Consensus

Consensus algorithms are a key component of large scale reliable distributed systems. They allow a group of machines to act as a logical unit and continue to operate in the presence of failures. The point of consensus is for all of the machines to agree on a shared state. For this they maintain a replicated append-only log. The group can continue to operate in the presence of faults and network partitions as long as a majority of them can still communicate. You typically want an odd number of nodes so that the group isn't divisible into two equally sized groups.

You often see consensus algorithms in play when you have a control plane that needs fault tolerance. For example, nomad and consul masters, vault servers, and mysql orchestrator all use raft. You also see it in the data plane when you need reliable replication of state, e.g. rabbitmq quorum queues and mysql clustering (this uses Paxos but the point stands).

## The Main Components of Raft

### Leader Election

Raft uses a strong form of leadership to simplify problem of maintaining replicated log. A leader is elected using randomized timeouts at startup and when the current leader becomes unreachable. At any given time a node is in one of three states: leader, follower, or candidate. Under nominal conditions there is one leader and all other servers are followers. Followers don't do anything. They simply respond to requests from the leader. The leader handles all requests from clients. If a follower receives a client request then it just forwards it to the leader.

The leader sends periodic (with a small random delay) heartbeats to its followers to let them know it is still up and functioning. If a follower doesn't hear from the leader after a certain time period, called the `election timeout` it enters a the third state called the candidate state. In this state is considers itself a candidate to become the leader and requests other members of the cluster to vote for it. If it receives votes from a majority of the cluster (including itself) then it transitions to the leader state and begins serving requests from clients and sending heartbeats to the other clusters members.

A new election starts a new `term`. The given leader will serve requests for the entire duration of the term and during that time there will be only the one leader. Some leader elections can result in a split vote and in that case the term will end with no leader and a new election and term will start. Because of the randomized timeouts split votes are very unlikely to repeat.

### Log Replication

Once a leader is elected it replicates all writes it receives to the other nodes, after appending the write to its own log. The followers append the write to their own log after doing some consistency checks and respond to the leader. When the leader sees that the log entry is replicated to a majority of the cluster it considers the log entry committed and applies it to its state.

The followers do a lot of checks to make sure their log is consistent with the leaders log. If any of those checks fail then the leader, which keeps track of the point up to which logs have been applied on each follower, tries to find the last point where the log of the follower was consistent with its own log and resend log entries to the follower from that point on. This fixes the follower's log to be consistent with the leaders.

### Safety

### Membership Changes
