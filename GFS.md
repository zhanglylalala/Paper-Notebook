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























