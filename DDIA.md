### Partitioning of Key-Value Data

##### Partitioning by Key Range

Disadvantage: the downside of key range partitioning is that certain access patterns can lead to hot spots. If the key is a timestamp, then the partitions correspond to ranges of time—e.g., one partition per day. Unfortunately, because we write data from the sensors to the database as the measurements happen, all the writes end up going to the same partition

Advantage:Within each partition, we can keep keys in sorted order. This has the advantage that range scans are easy, and you can treat the key as a concatenated index in order to fetch several related records in one query.

##### Partitioning by Hash of Key

![image-20200509191747228](/Users/gaozhipeng/Library/Application Support/typora-user-images/image-20200509191747228.png)

disadvantage: Unfortunately however, by using the hash of the key for partitioning we lose a nice property of key-range partitioning: the ability to do efficient range queries. Keys that were once adjacent are now scattered across all the partitions, so their sort order is lost.

### Partitioning and Secondary Indexes

##### Partitioning Secondary Indexes by Document

![image-20200509194432642](/Users/gaozhipeng/Library/Application Support/typora-user-images/image-20200509194432642.png)

advantage on **write**:each partition maintains its own secondary indexes, covering only the documents in that partition. It doesn’t care what data is stored in other partitions. Whenever you need to write to the database—to add, remove, or update a document—you only need to deal with the partition that contains the document ID that you are writing.

Disadvantage on **read** unless you have done something special with the document IDs, there is no reason why all the cars with a particular color or a particular make would be in the same partition. In Figure 6-4, red cars appear in both partition 0 and partition 1. Thus, if you want to search for red cars, you need to send the query to all partitions, and combine all the results you get back.

##### Partitioning Secondary Indexes by Term

![image-20200509195002419](/Users/gaozhipeng/Library/Application Support/typora-user-images/image-20200509195002419.png)

### Rebalancing Partitions

#### Strategies for Rebalancing

##### How not to do it: hash mod N

The problem with the mod N approach is that if the number of nodes N changes, most of the keys will need to be moved from one node to another.

##### Fixed number of partitions

![image-20200509200050425](/Users/gaozhipeng/Library/Application Support/typora-user-images/image-20200509200050425.png)

The partition number is fixed, but we can increase the node

##### Dynamic partitioning

When a partition grows to exceed a configured size (on HBase, the default is 10 GB), it is split into two partitions so that approximately half of the data ends up on each side of the split [26]. Conversely, if lots of data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent partition. This process is similar to what happens at the top level of a B-tree

### Operations: Automatic or Manual Rebalancing

#### Request Routing

\1. Allow clients to contact any node (e.g., via a round-robin load balancer). If that node coincidentally owns the partition to which the request applies, it can handle the request directly; otherwise, it forwards the request to the appropriate node, receives the reply, and passes the reply along to the client.

\2. Send all requests from clients to a routing tier first, which determines the node that should handle each request and forwards it accordingly. This routing tier does not itself handle any requests; it only acts as a partition-aware load balancer.

\3. Require that clients be aware of the partitioning and the assignment of partitions to nodes. In this case, a client can connect directly to the appropriate node, without any intermediary.

![image-20200509203055492](/Users/gaozhipeng/Library/Application Support/typora-user-images/image-20200509203055492.png)

Many distributed data systems rely on a separate coordination service such as Zoo‐ Keeper to keep track of this cluster metadata, as illustrated in Figure 6-8. Each node registers itself in ZooKeeper, and ZooKeeper maintains the authoritative mapping of partitions to nodes. Other actors, such as the routing tier or the partitioning-aware client, can subscribe to this information in ZooKeeper. Whenever a partition changes ownership, or a node is added or removed, ZooKeeper notifies the routing tier so that it can keep its routing information up to date

![image-20200509203418822](/Users/gaozhipeng/Library/Application Support/typora-user-images/image-20200509203418822.png)

#### Parallel Query Execution

### The Slippery Concept of a Transaction

#### The Meaning of ACID

##### Atomicity

ACID atomicity describes what happens if a client wants to make several writes, but a fault occurs after some of the writes have been processed

##### Consistency

The idea of ACID consistency is that you have certain statements about your data (invariants) that must always be true—for example, in an accounting system, credits and debits across all accounts must always be balanced

##### Isolation

That is no problem if they are reading and writing different parts of the database, but if they are accessing the same database records, you can run into concurrency problems (race conditions).

##### Durability

Durability is the promise that once a transaction has committed successfully, any data it has written will not be forgotten, even if there is a hardware fault or the database crashes.

#### Single-Object and Multi-Object Operations

Now, whenever a new message comes in, you have to increment the unread counter as well, and whenever a message is marked as read, you also have to decrement the unread counter.

![image-20200523122237752](/Users/gaozhipeng/Library/Application Support/typora-user-images/image-20200523122237752.png)

##### Single-object writes

compare-and-set operation 

##### The need for multi-object transactions

There are some use cases in which single-object inserts, updates, and deletes are sufficient. However, in many other cases writes to several different objects need to be coordinated

##### Handling errors and aborts

violating its guarantee of atomicity, isolation, or durability, it would rather abandon the transaction entirely than allow it to remain half-finished. Errors will inevitably happen, but many software developers prefer to think only about the happy path rather than the intricacies of error handling.

**Retry** If the transaction actually succeeded, but the network failed while the server triedto acknowledge the successful commit to the client (so the client thinks it failed), then retrying the transaction causes it to be performed twice