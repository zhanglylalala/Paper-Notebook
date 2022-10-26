# Purpose

1. Issues: Hard to parallel computation, distribute data, and handle server failures
2. Contribution: Proposed an interface where users only need to write relatively simple Map function and Reduce function, and the system will parallel and distribute automatically. 

# Model

1. What does users need to do? 

   - Users need to provide a Map function and a Reduce function. 

   - Map function will read the original data as key-value pairs, take one pair as input each time, and output intermediate key-value pairs. 
     $map(k1,v1)\rightarrow list(k2,v2)$
   - Reduce function will take the intermediate-key and a list of all intermediate-values for that key as input, and merge these values to form a smaller set of values. 
     $reduce(k2,list(v2))\rightarrow list(v2)$
   - When user need to implement the Mapper and Reducer as interface provided by the system, and pass to the MapReduce specification. After passing the input and output files, invoke the ``MapReduce`` function to execute. 

2. What run-time system need to do? 

   - Partition data
   - Schedule across a set of machines
   - Handle machine failures
   - Manage inter-machine communication. 

3. What is the system structure? 
   When the ``MapReduce`` function is invoked, one **master** process and several **worker** processes will be forked. 
   Master will assign work to workers, either a Map work, or a Reduce work. 

4. 
