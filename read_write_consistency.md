# Read and Write Consistency in 3FS

3FS uses a sophisticated approach to ensure strong consistency when multiple clients are simultaneously accessing the same file, such as when one client is reading while another is writing to the same file. This document explains how 3FS handles these concurrent access scenarios to maintain data consistency.

## Consistency Model

3FS employs Chain Replication with Apportioned Queries (CRAQ), a distributed replication protocol that provides strong consistency guarantees. The system is designed to be:

1. **Strongly consistent**: Clients always see the latest committed data, preventing stale reads
2. **Linearizable**: Operations appear to execute in a sequential order consistent with real-time ordering
3. **High-throughput**: The system can handle many concurrent operations without compromising performance

## Chunk-Level Locking Mechanism

When concurrent read and write operations access the same file chunk, 3FS uses the following mechanisms to ensure consistency:

1. **Chunk Locking**: When a write operation occurs, the system acquires a lock for the specific chunk being updated:
   ```cpp
   // When a write request is received:
   // 1. Once the write data is fetched into local memory buffer, a lock for the chunk 
   //    to be updated is acquired from a lock manager. 
   // 2. Concurrent writes to the same chunk are blocked. All writes are serialized at the head target.
   ```

2. **Lock Management**: Each storage target uses a `LockManager` to handle chunk locking:
   - The `ReliableUpdate` component manages the locking mechanism
   - A hash-based locking system prevents lock contention across different chunks
   - Locks are held until the write operation completes throughout the entire chain

3. **Write Propagation**: Writes propagate through the chain in order:
   - The head node acquires the lock first
   - Each node in the chain forwards the request to its successor
   - When a node receives a request, it creates both committed and pending versions

## Handling Concurrent Read and Write Operations

When a client reads a file while another client is writing to the same file, the following process ensures consistency:

1. **Reading During a Write Operation**:
   - When a read request arrives at a storage target, the target first checks if it only has a committed version
   - If only a committed version exists, this version is returned immediately to the client
   - If both committed and pending versions exist (indicating an in-progress write), the service responds with a special status code
   - The client may either wait and retry or specifically request the pending version (relaxed read)

2. **Version Management**:
   - Each chunk has a monotonically increasing version number
   - A committed version has version number `v`
   - A pending version (during a write) has version number `u = v + 1`
   - Once a write is fully committed, the pending version becomes the new committed version

3. **Write Acknowledgment**:
   - When the tail target receives a write, it atomically replaces its committed version with the pending version
   - It sends an acknowledgment message back through the chain
   - Each predecessor replaces its committed version with the pending version upon receiving the acknowledgment
   - The chunk lock is released only after this process completes

## Client-Side Handling

Clients handle consistency issues as follows:

1. **Read Retry Mechanism**:
   - If a client receives a response indicating both committed and pending versions exist, it can:
     - Wait a short interval and retry the read operation
     - Issue a relaxed read request to explicitly get the pending version

2. **Write Ordering**:
   - Writes from the same client to the same chunk are serialized via a channel ID and sequence number
   - This ensures ordering is preserved even with retries or failures

3. **Duplicate Request Detection**:
   - The `ReliableUpdate` component tracks request IDs to identify duplicate requests
   - This prevents the same write from being applied multiple times in case of retries

## Handling Failures

The system remains consistent even during failures:

1. **Node Failure During Write**:
   - If a node in the chain fails during a write, the cluster manager detects this through heartbeats
   - Failed nodes are marked offline and moved to the end of the chain
   - The chain configuration is updated and broadcast to all nodes
   - Requests are rerouted to skip the failed node
   - The system ensures writes are either fully committed or completely discarded

2. **Recovery Process**:
   - When a failed node recovers, it enters the "syncing" state
   - It receives a continuous stream of full-chunk-replace writes from its predecessor
   - Only chunks with higher chain versions or inconsistent version numbers are transferred
   - This ensures the recovering node has the latest data before serving requests

## Summary

3FS ensures strong consistency for concurrent read and write operations through a combination of:

1. Chunk-level locking that serializes writes to the same chunk
2. Version management that distinguishes between committed and pending data
3. Chain replication that ensures all nodes eventually have the same data
4. A robust recovery process that handles node failures without compromising consistency

This approach allows 3FS to offer both high performance and strong consistency guarantees, making it well-suited for AI training and inference workloads where data integrity is critical.