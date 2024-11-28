# Log Replication

The main operation of Raft's log replication is the AppendEntries RPC call. This call is made by the leader to each peer. It's also used as a heartbeat as described in the [Leader Election](./leader-election.md) section. Typically the heartbeats are AppendEntries calls with no log entries. However, when the leader is retrying failed AppendEntries calls, it will include log entries in the heartbeat messages to followers that are behind.

The high-level happy-path flow of log replication is as follows:

1. The leader receives a write request from a client.
2. The leader appends the new entry to its log.
3. The leader sends AppendEntries RPCs to all followers to replicate the new entry.
4. Each follower processes the AppendEntries RPC and appends the new entry to its log if the new entry is consistent with its log. It then responds to the leader letting it know if the entry was successfully replicated.
5. Once the leader has received enough successful responses to know that the log entries have been replicated to a majority of the cluster, it considers the entry committed and applies it to its state machine. The commit index is updated.
6. The leader responds to the client with the result of the write request.
7. On the next AppendEntries RPC followers see the updated commit index and apply the entry to their state machines.

This of course ignores the many failure scenarios that can occur. Raft is designed to handle these failures and ensure that the log remains consistent among all cluster members. If a log entry or entries cannot be saved among a majority then Raft ensures those entries are never committed and the client will not receive a success response. Typically, the leader will give up on retries after some timeout has elapsed at which point it will send a failure response back to the client. We'll cover the non-happy-path scenarios in the following sections.

## Log Replication in Detail

Let's go through the log replication process in more detail and look at how it's implemented in the Hashicorp Raft library. The Hashicorp Raft library is written in Go and is a popular implementation. One of the authors of Raft, Diego Ongaro, was involved in the development of the library. We'll compare how I'm implementing it in MIT's distributed systems course to the real implementation by Hashicorp. This is mainly just to help me understand and verify the correctness of my implementation but hopefully serves as a good deep dive into how the replication works.

The client sending a write to the leader isn't very interesting so let's start at step 2 "The leader appends the new entry to its log". In my 6.5840 code this is pretty straightforward. We create a new log entry with the `command` that will be applied to the state machine (if committed) and the current `term` and append it to the log.

```go
logent := &logEntry{
	Command: command,
	Term: rf.currentTerm,
}

// Append the log entry
rf.log = append(rf.log, logent)
```

My log is just a simple slice of log entries

```go
log []*logEntry
```

TODO: Hashicorp equivalent

Next is step 3, "the leader sends `AppendEntries` RPCs to all followers". How does that look in my MIT lab code?

```go
	// Send AppendEntries RPCs to all other servers to replicate the log
	for idx := range rf.peers {
		if idx == rf.me {
			continue // don't send AppendEntries RPC to self
		}

		peerIdx := idx
		if len(rf.log) >= rf.nextIndex[peerIdx] {
			// If last log index ≥ nextIndex for a follower: send AppendEntries RPC with log entries starting at nextIndex
			entries := &AppendEntries{
				Term:         rf.currentTerm,
				LeaderId:     rf.me,
				PrevLogIndex: rf.nextIndex[peerIdx] - 1,
				PrevLogTerm:  rf.log[rf.nextIndex[peerIdx]-1].Term,
				Entries:      rf.log[rf.nextIndex[peerIdx]:],
				LeaderCommit: rf.commitIndex,
			}

			go func() {
				rf.appendEntriesAndHandleResponse(peerIdx, entries)
			}()
		}
	}
```

The `AppendEntries` RPC includes the leader's current term, its id, the index of the log entry immediately preceding new ones in for the given peer, the term of that same preceding log entry, the actual entries to append (starting at the next index we are tracking for the given peer), and the leader's commit index. This is all specified in figure 2 of the Raft paper.

TODO: Hashicorp

Okay next step is for the followers to process the `AppendEntries` RPC. This is where things get more complicated. If the `AppendEntries` has a term number less than the follower's term then the follower immediately replies false and the `AppendEntries` fails. The leader shouldn't be behind the followers!

```go
	// Initialize reply
	reply.Term = rf.currentTerm
	reply.VoteGranted = false
...
	// 1. Reply false if term < currentTerm (§5.1)
	if args.Term < rf.currentTerm {
		DPrintf("Server %d: AppendEntries RPC reply sent to server %d. Term %d < currentTerm %d", rf.me, args.LeaderId, args.Term, rf.currentTerm)
		return // Failed AppendEntries
	}
```

Next the follower needs to ensure that the log entries the leader sent are consistent with its own log. To do that it first checks that the term of the preceding log entries match.

```go
	// Note: Even heartbeats can contain log entries bc they are used to retry failed appends (e.g. follower logs is
	// inconsistent with leader). The leader will have decremented nextIndex for the follower that failed to append.
	if len(args.Entries) > 0 {
		// 2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3)
		prevLogSliceIndex := args.PrevLogIndex - 1
		if prevLogSliceIndex < 0 || prevLogSliceIndex >= len(rf.log) {
			DPrintf("Server %d: Index %d out of bounds, sliceIndex: %d", rf.me, args.PrevLogIndex, prevLogSliceIndex)
			panic("Index out of bounds")
		}
		matchingPrevIndexLogEntry := rf.log[prevLogSliceIndex]
		if matchingPrevIndexLogEntry.Term != args.PrevLogTerm {
			DPrintf("Server %d: AppendEntries RPC reply sent to server %d. Log doesn't contain an entry at prevLogIndex %d whose term matches prevLogTerm %d", rf.me, args.LeaderId, args.PrevLogIndex, args.PrevLogTerm)
			return // Failed AppendEntries
		}
...
    }
```

If the previous log entries terms match then we can proceed to check that the rest of the log is consistent. If there's a mismatch then the follower must delete the conflicting entry and everything that follows it.

```go
	if len(args.Entries) > 0 {
...
		// 3. If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it (§5.3)
		for i := prevLogSliceIndex + 1; i < len(rf.log); i++ {
			// i - prevLogSliceIndex - 1 gives the correct index into args.Entries
			if rf.log[i].Term != args.Entries[i-prevLogSliceIndex-1].Term {
				rf.log = truncateLog(rf.log, i)
				break
			}
		}
...
    }
```

Once we verify the log is consistent with the leaders log up until this point then we can append the new log entries.

```go
	if len(args.Entries) > 0 {
...
        // 4. Append any new entries not already in the log
		for i, entry := range args.Entries {
			logIndex := args.PrevLogIndex + 1 + i
			if logIndex-1 >= len(rf.log) {
				rf.log = append(rf.log, entry)
			}
		}
...
    }
```

Now the log is appended. A final check the follower does when responding to each `AppendEntries` is to move the commit index forward to match the leaders commit index.

```go
	if len(args.Entries) > 0 {
...
    }

	// 5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry) (§5.3)
	// The min() ensures that the follower's commitIndex is updated safely, preventing it committing entries that it
	// hasn't received yet. The leader's commitIndex could be beyond the follower's last log index. In that case, we
	// adjust the commitIndex to the follower's last log index.
	if args.LeaderCommit > rf.commitIndex {
		rf.commitIndex = min(args.LeaderCommit, len(rf.log))
	}
```

Finally the follower can update its term to match the leader's term and reply success.

```go
    // 6. Reply success
	reply.Term = rf.currentTerm
	reply.Success = true // Successful AppendEntries
```
