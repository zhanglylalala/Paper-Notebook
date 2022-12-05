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