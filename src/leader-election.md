# Leader Election

## Overview

Raft's leader election process is designed to be simple and efficient. The leader is responsible for managing the replication of log entries to followers. If the leader fails, a new leader must be elected before log replication can continue. The leader election process is triggered by a follower detecting that it has not received a heartbeat from the leader in a certain amount of time. This timeout is known as the election timeout. When a follower times out, it starts a new election by transitioning to the candidate state and sending RequestVote RPCs to other servers in the cluster. If the candidate receives votes from a majority of the cluster, it becomes the leader.
