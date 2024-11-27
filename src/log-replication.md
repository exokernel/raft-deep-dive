# Log Replication

## Overview

The main operation of Raft's log replication is the AppendEntries RPC call. This call is made by the leader to each peer. It's also used as a heartbeat as described in the [Leader Election](./leader-election.md) section. Typically the heartbeats are AppendEntries calls with no log entries. However, when the leader is retrying failed AppendEntries calls, it will include log entries in the heartbeat messages to followers that are behind.

The high-level happy-path flow of log replication is as follows:

1. The leader receives a write request from a client.
2. The leader appends the new entry to its log.
3. The leader sends AppendEntries RPCs to all followers to replicate the new entry.
4. Each follower processes the AppendEntries RPC and appends the new entry to its log if the new entry is consistent with its log. It then responds to the leader letting it know if the entry was successfully replicated.
5. Once the leader has received enough successful responses to know that the log entries has been replicated to a majority of the cluster, it considers the entry committed and applies it to its state machine.
6. The leader responds to the client with the result of the write request.

This of course ignores the many failure scenarios that can occur. Raft is designed to handle these failures and ensure that the log remains consistent among all cluster members. If a log entry or entries cannot be saved among a majority then Raft ensures those entries are never committed and the client will not receive a success response. Typically, the leader will give up on retries after some timeout has elapsed at which point it will send a failure response back to the client. We'll cover the non-happy-path scenarios in the next section.
