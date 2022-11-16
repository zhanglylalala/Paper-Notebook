[toc]

# Purpose

1. Contribution: provides fault tolerance while running on inexpensive commodity hardware, and it delivers high aggregate performance to a large number of clients. 
2. Difference points in design space
   - This system integreted constant monitoring, error detection, fault tolerance, and automatic recovery. 
   - Files are huge by traditional standards. Design assumptions and parameters such as I/O operation and block sizes have to be revisited. 
   - Most files are mutated by appending new data rather than overwriting existing data. Random writes within a file are practically non-existent. Once written, the files are only read, and often only seuqentially. 

# Model

## Overview

1. **What operations are supported?**

   - Usual operations: ``create``, ``delete``, ``open``, ``close``, ``read``, and ``write``

   - ``snapshot``: creates a copy of a file or a directory tree at low cost

   - ``record append``: allows multiple clients to append data to the same file concurrently while guaranteeing the atomicity

### Architecture

<img src="imgs/GFS/01.png" style="zoom:33%;" />

1. Consist of a signle *master* and multiple *chunkservers* and is accessed by multiple *clients*. 

2. Files are divided into fixed-size chunks. Each chunk is identified by an immutable and globally unique 64 bit *chunk handle* assigned by the master at the timeof chunk creation. 

   Chunkservers store chunks on local disks as Linux files and read or write chunk data specified by a chunk handle and byte range. 

   Each chunk is replicated on multiple chunkservers, three by default. 

6. Neither the client nor the chunkserver caches file data. Caches offer little benefit while causing coherence issues. But clients do cache metadata. 

#### Master

1. **What does master need to do?**
   - The master maintains all file system metadata, and controls system-wide activities. 

     - Metadata includes namespace, access control information, the mapping from files to chunks, and the current locations of chunks. 

     - System-wide activities includes chunk lease management, garbage collection of orphaned chunks, and chunk migration between chunkservers. 
   - The master periodically comminicates with each chunkserver in HeartBeat messages to give it instructions and collect its state. 
   - Clients interact with the master for metadata operations, but all data-bearing communication goes directly to the chunkservers. 
2. **How to prevent the master become a bottleneck?**
   - The idea is to minimize its involvement in reads and writes. 
   - A client asks the master which chunkservers it should contact, and caches this information for a limited time and interacts with the chunkservers directly for subsequent operations. 
3. **How does client communicate with master specifically?**
   - The client translates the file name and byte offset specified by the application into a chunk index within the file. 
   - It sends the master a requeust containing the file name and chunk index. 
   - The master replies with the corresponding chunk handle and locations of the replicas. 
   - The client caches this information using the file name and chunk index as the key. 
4. **What metadata does master need to store?**
   - Stored persistently: the file and chunk namespace, the mapping from files to chunks
     - The master will scan periodically through its entire state in the background
     - Periodic scanning is to implement chunk garbage collection, re-replication in the presence of chunkserver failures, and chunk migration to balance load and disk space usage across chunkservers. 
     - The number of chunks and hence the capacity of the whole system is limited by how much memory the master has. But not a serious limitation for less than $64$ bytes of metadata for each chunk. 
   - No need to store persistently: the locations of each chunk's replicas. 
     - The master asks each chunkserver about its chunks at master startup and whenever a chunkserver joins the cluster. 
     - The master can keep itself up-to-date thereafter because it controls all chunk placement and monitors chunkserver status. 
   - Operation record
     - The namespace and mapping are kept persistent by logging mutations to operation log
     - It is stored on the master's local disk and replicated on remote machines
5. **How does master persist?**
   - Operation log
     - The operation log contains a historical record of critical metadata changes. Not only is it the only persistent record of critical metadata, it also serves as a logical time line that defines the order of concurrent operations. 
     - The system respond to a client operation only after flushing the corresponding log record to disk both locally and remotely. 
     - The master batches several log records together before flushing thereby reducing the impact of flushing and replication on overall system thoughput. 
   - Checkpoint
     - To minimize startup time, we must keep the log small. The master checkpoints its state whenever the log grows beyond a certain size. It can recover by loading the latest checkpoint and replaying only the records after that. 
     - The checkpoint is in a compact B-tree like form. 
     - The master switches to a new log file and creates the new checkpoint in a separate thread. 
     - Older checkpoints and log files can be freely deleted, though we keep a few around to guard against catastrophes. A failure during checkpointing does not affect correctness because the recovery code detects and skips incomplete checkpoints. 

#### Chunk size

1. **What is the advantage of large chunk size?**
   - Reduce clients' need to interact with the master because reads and writes on the same chunk require only one initial request to the master for chunk location information. 
   - Reduce network overhead by keeping a persistent TCP connection to the chunkserver over an extended period of time. 
   - Reduce the size of the metadata stored on the master. 
2. **What is the disadvantage of large chunk size?**
   - Wasting space due to internal fragmentation. This can be eased through lazy space allocation. 
   - A small file consists of a small number of chunks, perhaps just one. The chunkservers storing those chunks may become hot spots if many clients are accessing the same file. This have not been a major issue. 
3. **When will hot spots problem emerge?**
   - A more common case is that an executable was written to GFS as a single-chunk file and then started on hundreds of machines at the same time. 
   - We can fix this problem by storing such executables with a higher replication factor. 

### Consistency

1. **How do we define a file region being consistent or defined?**
   - A file region is consistent if all clients will always see the same data, regardless of which replicas they read from. 
   - A region is defined after a file data mutation if it is consistent and clients will see what the mutation writes in its entirety. 
   - Concurrent successful mutations leave the region undefined but consistent: all clients see the same data, but it may not reflect what any one mutation has written. Typically, it consists of mingled fragments from multiple mutations. 
2. **How many consistency rules should be considered?**
   - File namespace mutations (e.g. file creation) are atomic. 
   - After a sequence of successful mutations, the mutated file region is guaranteed to be defined and contain the data written by the last mutation. 
3. **How do GFS guarantees the second rule?**
   - Applying mutations to a chunk in the same order on all its replicas. 
   - Using chunk version numbers to detect any replica that has become stale because it has missed mutations while its chunkserver was down. 
   - Stale replicas will never be involved in a mutation or given to clients asking the master for chunk locations. They are garbage collected at the earliest opportunity. 
4. **What is the side-effect of clients caching chunk locations?**
   - They may read from a stale replica before that information is refreshed.  
   - This window is limited by the cache entry's timeout and dthe next open of the file. 
   - As most of files are append-only, a stale replica usually returns a premature end of chunk rather than outdated data. 

### Write and record append

1. **What is the difference between write and record append?**
   - A write causes data to be written at an application-specified file offset. 
   - A record append causes data (the “record”) to be appended atomically at least once even in the presence of concurrent mutations, but at an offset of GFS’s choosing. 
     - The offset is returned to the client and marks the beginning of a defined region that contains the record. 
     - GFS may insert padding or record duplicates in between. They occupy regions considered to be inconsistent and are typically dwarfed by the amount of user data. 
   - A “regular” append is merely a write at an offset that the client believes to be the current end of file. 
2. **How does typical writing happen?**
   - A writer generates a file from beginning to end. It atomically renames the file to a permanent name after writing all the data, or periodically checkpoints how much has been successfully written. 
   - Checkpoints may also include application-level check- sums. Readers verify and process only the file region up to the last checkpoint, which is known to be in the defined state. 
   - Checkpointing allows writers to restart incrementally and keeps readers from processing successfully written file data that is still incomplete. 
3. **How does readers deal with occasional padding and duplicates?**
   - Each record prepared by the writer contains extra information like checksums so that its validity can be verified. 
   - A reader can identify and discard extra padding and record fragments using the checksums. 
   - If it cannot tolerate the occasional duplicates, it can filter them out using unique identifiers in the records, which are often needed anyway to name corresponding application entities such as web documents. 

## Data mutation

### Control & data flow

1. **How do we minimize management overhead at the master of data mutation?**

   - We use leases to maintain a consistent mutation order across replicas. 
   - The master grants a chunk lease to one of the replicas, which we call the primary. The primary picks a serial order for all mutations to the chunk. All replicas follow this order when applying mutations. 
   - The client caches who is the lease of a certain chunk for future mutations. It needs to contact the master again only when the primary becomes unreachable or replies that it no longer holds a lease. 

2. **What does leases change?**

   - A lease has an initial timeout of 60 seconds. However, as long as the chunk is being mutated, the primary can request and typically receive extensions from the master indefinitely. 
   - These extension requests and grants are piggybacked on the HeartBeat messages regularly exchanged between the master and all chunkservers. 
   - The master may sometimes try to revoke a lease before it expires (e.g., when the master wants to disable mutations on a file that is being renamed). 
   - Even if the master loses communication with a primary, it can safely grant a new lease to another replica after the old lease expires. 

3. **How does the control flow?**

   - In step 1, the client asks the master which chunkserver holds the current lease for the chunk and the locations of the other replicas. If no one has a lease, the master grants one to a replica it chooses
   - In step 2, the master replies with the identity of the primary and the locations of the other (secondary) replicas. 
   - In step 3, The client pushes the data to all the replicas in any order, instead of only sending to the lease. 
   - In step 4, once all the replicas have acknowledged receiving the data, the client sends a write request to the primary. 
   - In step 5, the primary forwards the write request to all sec- ondary replicas. 
   - In step 6, the secondaries all reply to the primary indicating that they have completed the operation. 
   - In step 7, the primary replies to the client. Any errors encoun- tered at any of the replicas are reported to the client. 

   <img src="imgs/GFS/02.png" style="zoom:25%;" />

4. **How does primary and secondary servers write data?**

   - Each chunkserver will store the data from client in an internal LRU buffer cache until the data is used or aged out. 
   - The write request from client to primary identifies the data pushed earlier to all of the replicas. 
   - The primary assigns consecutive serial numbers to all the mutations it receives, possibly from multiple clients, which provides the necessary serialization. 
   - The primary applies the mutation to its own local state in serial number order. 

5. **What would the system do if write fails?**

   - If it had failed at the primary, it would not have been assigned a serial number and forwarded. 
   - In other cases, the write may have succeeded at the primary and an arbitrary subset of the secondary replicas. The client request is considered to have failed, and the modified region is left in an inconsistent state. 
   - The client code handles such errors by retrying the failed mutation. It will make a few attempts at steps 3 through 7 before falling back to a retry from the beginning of the write. 

6. **What if a write is large or straddles a chunk boundary?**

   - GFS client code breaks it down into multiple write operations. 
   - They all follow the control flow described above but may be interleaved with and overwritten by concurrent operations from other clients. 
   - The shared file region may end up containing fragments from different clients, although the replicas will be identical because the individual operations are completed successfully in the same order on all replicas. 

7. **How to prevent primary become bottleneck of pushing data?**

   - While control flows from the client to the primary and then to all secondaries, data is pushed linearly along a carefully picked chain of chunkservers in a pipelined fashion. 
   - By decoupling the data flow from the control flow, we can improve performance by scheduling the expensive data flow based on the network topology regardless of which chunkserver is the primary. 
   - Our goals are to fully utilize each machine’s network bandwidth, avoid network bottlenecks and high-latency links, and minimize the latency to push through all the data. 
   - Each machine forwards the data to the “closest” machine in the network topology that has not received it. 
   - Our network topology is simple enough that “distances” can be accurately estimated from IP addresses.

8. **How to minimize latency of pushing data?**

   - We minimize latency by pipelining the data trans- fer over TCP connections. Once a chunkserver receives some data, it starts forwarding immediately. 
   - Pipelining is especially helpful to us because we use a switched network with full-duplex links. Sending the data immediately does not reduce the receive rate. 

### Atomic record appends

1. 

















