# Raft Intro

Raft is a consensus algorithm. It's used in distributed systems to reliably and consistently replicate state between multiple nodes using an append-only log.  When raft nodes start up they first vote to elect a leader. After a leader is elected all application state writes go to the leader and the other nodes become followers. The leader sends all writes to the followers for replication and once a majority of the nodes have saved the write to their log it is considered committed and the leader responds to the writing client with success.

Raft guarantees that the log is always consistent among the leader and followers. It also designed for fault tolerance. Nodes may crash or restart (even the leader) or the system may experience network partitions. The leader maintains contact with the followers using periodic heartbeats. If there is a problem communicating with the leader the followers don't receive these heartbeats and they will elect a new leader.

Raft was designed to be easy to understand and implement. It's broken down into subproblems that function more or less independently: leader election, log replication, and safety. It also includes a joint consensus mechanism for membership changes that allow it to continue operating as nodes are added and removed.

In this short book we will take a deep dive into how Raft works under the hood. We'll look at each subcomponent of Raft and try to explain it as clearly as possible with words, diagrams, and Golang source code.

### Proposed Outline

* 2 Consensus, what is it good for
	* Agreement on distributed state
	* Availability
	* Fault tolerance
* 3 The Goals of Raft
	* Easier to teach, easier to understand, easier to implement (than Paxos)
	* As performant (as Paxos)
	* Reasonably Fault Tolerant
* 4  How to Build a Raft (The Distinct Parts)
	* The Roles
		* Candidate
			* Send RV RPCs to peers
			* Become leader or become follower
		* Leader
			* Get client requests
			* Send AE to followers
			* Commit
			* Respond to client requests
			* Also, respond to RV, maybe step down and become follower
		* Follower
			* Wait for AE RPCs from Leader and respond to RV
	* Leader Election
	* Log Replication
	* Safety
	* Membership Changes w/ Joint Consensus
* 5 Leader Election Deep Dive
	* RequestVote RPC
	* Election Timeout
	* Handling RequestVote
	* Handling RequestVote Response
	* Leader Failures
* 6 Log Replication Deep Dive
	* AppendEntries RPC
	* Handling AppendEntries
	* Handling AppendEntries Response
	* Retrying after Failures
* 7 Safety Deep Dive
	* The State Machine Safety Property
* 8 Membership Changes Deep Dive
	* Joint Consensus
* 9 Raft in the Real World
	* Hashicorp Raft Library
	* Consul
	* RabbitMQ Quorum Queues
	* MySQL Orchestrator
