# Purpose

1. Enhance the understandability of Paxos
   - Raft separates the key elements of consensus, such as leader election, log replication, and safety
   - It enforces a stronger degree of coherency to reduce the number of states that must be considered
2. Novel features: strong leader, leader election, membership changes. 
3. **What is the common properties of consensus algorithms?**
   - Safety: never returning an incorrect result under all non-Byzantine conditions, including network delays, partitions, and packet loss, duplica- tion, and reordering.
   - Available as long as any majority of the servers are operational and can communicate with each other and with clients. 
   - They do not depend on timing to ensure the consistency of the logs: faulty clocks and extreme message delays can, at worst, cause availability problems. 
   - A command can complete as soon as a majority of the cluster has responded to a single round of remote procedure calls; a minority of slow servers need not impact overall system performance. 

# Model

## Basics

1. **How does Raft implements consensus overall?**
   - First electing a distin- guished leader, then giving the leader complete responsi- bility for managing the replicated log. 
   - The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines. 
   - A leader can fail or be- come disconnected from the other servers, in which case a new leader is elected.
2. **What are the states of each server?**
   - At any given time each server is in one of three states: leader, follower, or candidate. 
   - In normal operation there is exactly one leader and all of the other servers are followers. 
   - Followers are passive: they issue no requests on their own but simply respond to requests from leaders and candidates. 
   - The leader handles all client requests. If a client contacts a follower, the follower redirects it to the leader. 
   - The candidate is used to elect a new leader. 
3. **How to divide the terms?**
   - Terms are numbered with consecutive integers. Each election begins a new term. 
   - If an election results in a split vote, the term will end with no leader; a new term with a new election will begin shortly. 
   - Terms act as a logical clock in Raft, and they allow servers to detect obsolete information such as stale leaders. 
4. **How does terms change?**
   - Each server stores a current term number, which increases monotonically over time. 
   - Current terms are exchanged whenever servers communicate; if one server’s current term is smaller than the other’s, then it updates its current term to the larger value. 
   - If a candidate or leader discovers that its term is out of date, it immediately reverts to fol- lower state. 
   - If a server receives a request with a stale term number, it rejects the request. 
5. **How does Raft handle follower and candidate crashes?**
   - If a follower or candidate crashes, then fu- ture RequestVote and AppendEntries RPCs sent to it will fail. Raft handles these failures by retrying indefinitely. 
   - If a server crashes after completing an RPC but before responding, then it will receive the same RPC again after it restarts. Raft RPCs are idempotent, i.e. servers will ignore the RPCs that is already handled, so this causes no harm. 

### States stored on servers

1. Persistent state on all servers: These states need to be updated on stable storage before responding to RPCs, i.e. communicating with outside. 
   - ``currentTerm``: latest term server has seen (initialized to 0 on first boot, increases monotonically)
   - ``votedFor``: candidateId that received vote in current term (or ``null`` if none)
   - ``log[]``: log entries
2. Volatile state on all servers:
   - ``commitIndex``: index of highest log entry known to be committed (initialized to 0, increases monotonically)
   - ``lastApplied``: index of highest log entry applied to state machine (initialized to 0, increases monotonically)

3. Volatile state on leaders: These states need to be reinitialized after election
   - ``nextIndex[]``: for each server, index of the next log entry to send to that server (initialized to leader last log index + 1)
   - ``matchIndex[]``: for each server, index of highest log entry known to be replicated on server (initialized to 0, increases monotonically)

## Leader election

1. **How does the servers states transit?**

   - When servers start up, they begin as followers. A server remains in follower state as long as it receives valid RPCs from a leader or candidate. 
   - Leaders send periodic heartbeats (AppendEntries RPCs that carry no log entries) to all followers in order to maintain their authority. 
   - If a follower receives no communication over a period of time called the election timeout, then it assumes there is no vi- able leader and become a candidate to initiate a new election. 
   - A candidate that receives votes from a majority of the full cluster becomes the new leader. 

   ![](../imgs/Raft/01.png)

2. **How to elect a leader?**

   - To begin an election, a follower increments its current term and transitions to candidate state. 
   - It then votes for itself and issues RequestVote RPCs in parallel to each of the other servers in the cluster. 
   -  Candidate continues in this state until one of three things happens
     - It wins the election
     - Another server establishes itself as leader
     - A period of time goes by with no winner. 

3. **How is the election held?**

   - Each server will vote for at most one candidate in a given term, on a first-come-first-served basis to ensure that at most one candidate can win the election for a particular term. 
   - A candidate wins an election if it receives votes from a majority of the servers in the full cluster for the same term. 
   - Once a candidate wins an election, it becomes leader. It then sends heartbeat messages to all of the other servers to establish its authority and prevent new elections. 

4. **How to determine that a server loses the election?**

   - A candidate may receive an AppendEntries RPC from another server claiming to be leader. 
   - If the leader’s term is at least as large as the candidate’s current term, then the candidate recognizes the leader as legitimate and returns to follower state. 
   - If the term in the RPC is smaller than the candidate’s current term, then the candidate rejects the RPC and con- tinues in candidate state. 

5. **How to handle a split vote?**

   - If many followers become candidates at the same time, votes could be split so that no candidate obtains a majority. 
   - Each candidate will time out and start a new election by incre- menting its term and initiating another round of Request- Vote RPCs. 
   - Raft uses randomized election timeouts to ensure that split votes are rare and that they are resolved quickly. 
     - Election timeouts are chosen randomly from a fixed interval at the start of an election, and it waits for that timeout to elapse before starting the next election. 
     - In most cases only a single server will time out; it wins the election and sends heartbeats before any other servers time out. 

6. **How to ensure that the leader of any given term contains all of the entries committed in previous terms?**

   - A candidate must contact a majority of the cluster in order to be elected, which means that every committed entry must be present in at least one of those servers. 
   - If the candidate’s log is at least as up-to-date as any other log in that majority， then it will hold all the committed entries. 
   - In the RequestVote RPC, the voter denies its vote if its own log is more up-to-date than that of the candidate. 
   - Raft determines which of two logs is more up-to-date by comparing the index and term of the last entries in the logs. 
     - If the logs have last entries with different terms, then the log with the later term is more up-to-date. 
     - If the logs end with the same term, then whichever log is longer is more up-to-date. 

7. **What is the limitation of broadcast time and election timeout?**

   - $broadcastTime\ll electionTimeout\ll MTBF$
   - BbroadcastTime is the average time it takes a server to send RPCs in parallel to every server in the cluster and receive their responses. electionTime- out is the election timeout. MTBF is the average time between failures for a single server. 
   - The broadcast time should be an order of mag- nitude less than the election timeout so that leaders can reliably send the heartbeat messages required to keep fol- lowers from starting elections. 
   - The election timeout should be a few orders of magnitude less than MTBF so that the sys- tem makes steady progress. 

### RequestVote RPC

1. This is invoked by candidates to gather votes
2. Arguments: 
   - ``term``: candidate's term
   - ``candidateId``: candidate requesting vote
   - ``lastLogIndex``: index of candidate's last log entry
   - ``lastLogTerm``: term of candidate's last log entry
3. Results: 
   - ``term``: currentTerm, for candidate to update itself
   - ``voteGranted``: true means candidate received vote
4. Receiver implementation:
   - Reply ``false`` if term < currentTerm
   - If votedFor is ``null`` or ``candidateId``, and candidate's log is at least as up-to-date as receiver's log, grant vote

## Log replication

1. **How is clients request handled?**

   - Each client request contains a command to be executed by the replicated state machines. 
   - The leader appends the command to its log as a new entry, then is- sues AppendEntries RPCs in parallel to each of the other servers to replicate the entry. 
   - When the entry has been safely replicated, the leader applies the entry to its state machine and returns the result of that execution to the client. 
   - If followers crash or run slowly, or if network packets are lost, the leader retries Append- Entries RPCs indefinitely (even after it has responded to the client) until all followers eventually store all log en- tries.

2. **What is stored in a log entry?**

   - A command for state machine
   - A term number when entry was received by leader. 
   - An index to identify its position in the log. The index of the first log is 1. 

3. **How to apply a log entry to the state machines?**

   - The leader decides when it is safe to apply a log en- try to the state machines. 
     - Such an entry is called ``commit- ted``. 
     - Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines. 
   - A log entry is committed once the leader that created the entry has replicated it on a majority of the servers. 
   - It also commits all preceding entries in the leader’s log, including entries created by previous leaders. 
   - The leader keeps track of the highest index it knows to be committed, and it includes that index in future AppendEntries RPCs, including heartbeats, so that the other servers eventually find out that they should commit some new entries. 
   - Once a follower learns that a log entry is committed, it applies the entry to its local state machine in log order. 

4. **How to determine the consistency between logs?**

   - **Log Matching property**: If two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index. 

   - The Log Matching property is maintained through the following properties: 

     - If two entries in different logs have the same index and term, then they store the same command. 

     - If two entries in different logs have the same index and term, then the logs are identical in all preceding entries. 

5. **How to check the consistency in AppendEntries RPCs?**

   - When send- ing an AppendEntries RPC, the leader includes the index and term of the entry in its log that immediately precedes the new entries. 
   - If the follower does not find an entry in its log with the same index and term, then it refuses the new entries. 

6. **What kinds of inconsistency may incur?**

   - Leader crashes can leave the logs inconsistent. The old leader may not have fully replicated all of the entries in its log. 
   - A follower may be missing entries that are present on the leader, it may have extra entries that are not present on the leader, or both. 

7.  **How does leader handle follower inconsistencies?**

   - **Leader Append-Only** property: a leader never overwrites or deletes entries in its log. 
   - The leader handles inconsistencies by forcing the followers' logs to duplicate its own. Namely, conflicting entries in follower logs will be overwritten with entries from the leader's log. 
   - The leader must find the latest log entry where the two logs agree, delete any entries in the follower’s log after that point, and send the follower all of the leader’s entries after that point. 
   - All of these actions happen in response to the consistency check performed by AppendEntries RPCs. 
   - A leader does not need to take any special actions to restore log consistency when it comes to power. It just begins normal operation, and the logs auto- matically converge in response to failures of the Append- Entries consistency check. 

8. **How does AppendEntries RPC perform consistency check?**

   - The leader maintains a nextIndex for each follower. When a leader first comes to power, it initializes all nextIndex values to the index just after the last one in its log, i.e. assuming all followers are as up-to-date as itself. 
   - If a follower’s log is inconsistent with the leader’s, the AppendEntries consis- tency check will fail in the next AppendEntries RPC. 
   - Af- ter a rejection, the leader decrements nextIndex and retries the AppendEntries RPC. 
   - Eventually nextIndex will reach a point where the leader and follower logs match. When this happens, AppendEntries will succeed, which removes any conflicting entries in the follower’s log and appends entries from the leader’s log (if any). 
   - Once AppendEntries succeeds, the follower’s log is consistent with the leader’s, and it will remain that way for the rest of the term. 

9. **How to handle uncommited entries from previous leaders?**

   - If a leader crashes be- fore committing an entry, future leaders will attempt to finish replicating the entry. 
   - A leader cannot im- mediately conclude that an entry from a previous term is committed once it is stored on a majority of servers. 
   - Raft never commits log entries from previous terms by count- ing replicas. 
   - Only log entries from the leader’s current term are committed by counting replicas; once an entry from the current term has been committed in this way, then all prior entries are committed indirectly because of the Log Matching Property. 

## Log compation

1. 

## Reproduce and unmentioned parts

1. **How to optimize the consistency check protocol?**
   - Additional AppenEntries RPC results for fast roll back:
     - ``Xterm``: the term of the conflicting entry. 
     - ``Xindex``: the first index that is in ``Xterm``. 
     - ``Xlen``: the length of log
   - Fast roll back implementation: 
     -  If the leader doesn't have ``Xterm``, then every entries of ``Xterm`` in follower's log will causing conflict. Hence the ``nextIndex`` can backup to ``Xindex``. 
     - If the leader has ``Xterm``, the matching entry in leader's log must have a term no larger than ``Xterm``. Hence, the ``nextIndex`` should backup to the next entry of the last ``Xterm`` in leader's log. 
     - If the follower's conflicting is due to empty in ``prevLogTerm``, then ``Xterm`` is set to $-1$, and leader should backup to ``Xlen``. 