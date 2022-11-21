# Purpose

1. Contribution:
   - This paper implemented a system providing fault tolerance virtual machine (VM) based on the approach of replicating the execution of a primary VM vis a backup VM on another server. The system automatically restores redundancy after faulure. 
   - It reduces performance of real applications by less than 10%. The data bandwidth needed to keep the primary and secondary VM executing in lockstep is less than 20 Mb/s for several real applications, which allows for the possibility of implementing fault tolerance over longer distance. 
   - The system automatically restores redundancy after a failure by starting a new backup viretual machine on any available server in the local cluster. 
2. Limitation: 
   - Only support uni-processor VMs. Recording and replaying the execution of a multi-processor VM have significant performance issues because nearly every access to shared memory can be a non-deterministic operation. 
   - Only attempt to deal with fail-stop failure, which are server failures that can be detected before the failing server causes an incorrect externally visible action. 

# Model

1. **What is the usual way to implement fault-tolerance via primary/backup approach?**

   - The backup server is always available to take over is the primary server fails. 
     - The problem is that the state of the backup server must be kept nearly identical to the primary server at all times. 

   - One way is to ship shcanges to all state of the primary. The bandwidth needed to send can be very large. 
   - Another method is the state-machine approach. 
     - The idea is to model the servers as deterministic state machcines that are kept in sync by starting them from the same initial state and ensuring that they receive the same input requests in the same order. 
     - Some operations are not deterministic. Extra coordination must be used to ensure that they receive a primary and backup are kept in sync. 
     - The extra information is far less than the amount of state (mainly memory updates) that is changing in the primary. 

2. **What is the difference between physical servers and VM in state machine level?**

   - Implementing coordication to ensure deterministic execution of physical servers is difficult, particularly as processor frequencies increase. 
   - VM running on top of a hypervisor can be considered a well-defined state machine. 
   - VMs still have non-deterministic operations. Hypervisor is able to capture all the necessary information about non-deterministic operations on the primary VM and to replay these operations correctly on the backup VM. 
   - 

























