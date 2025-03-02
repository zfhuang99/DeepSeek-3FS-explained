# Data Durability in 3FS

## Chain Replication Architecture

3FS uses Chain Replication with Apportioned Queries (CRAQ) for data replication. Each file chunk is replicated across multiple storage targets organized in a chain structure. Typically, a file chunk has 3 replicas distributed across different physical nodes for redundancy.

The key components responsible for maintaining data durability are:

1. **Cluster Manager (Mgmtd)**: Monitors storage target health through heartbeats and manages chain configuration
2. **Storage Services**: Manage chunks on storage targets and synchronize data between replicas
3. **Chain Tables**: Define the placement of replicas across storage targets

## Failure Detection Process

When a storage target fails, the system detects it through the following process:

1. The cluster manager (Mgmtd) relies on heartbeats to detect fail-stop failures. If a service doesn't send heartbeats for a configurable interval (e.g., T seconds), it's declared failed.
2. Storage services also monitor their own connection to the cluster manager, and will exit if they can't communicate for T/2 seconds.
3. When a failure is detected, the cluster manager updates the state of the affected storage target as "offline" and moves it to the end of the chains it participates in.

## Chain Reconfiguration

The `MgmtdChainsUpdater` class in the Mgmtd service is responsible for periodically checking and updating chain configurations. When a failure is detected:

1. The cluster manager identifies "candidate chains" that need updates due to target state changes.
2. It generates a new chain configuration using the `generateNewChain` function.
3. It increments the chain version to indicate a change in the chain.
4. It updates the chain information in persistent storage and in memory.
5. It broadcasts this updated chain configuration to all services.

## Target States and Transitions

Each storage target has both a public state (visible to clients) and a local state (for internal management). The possible public states are:

- **serving**: Service is alive and handling read/write requests
- **syncing**: Service is alive but undergoing data recovery 
- **waiting**: Service is alive but recovery hasn't started
- **lastsrv**: Service is down and was the last serving target
- **offline**: Service is down or has storage medium failure

When a target transitions from offline to waiting and then to syncing, the state transition follows carefully designed rules to ensure consistency.

## Data Recovery Process

When a previously offline storage service restarts, it goes through a detailed recovery process:

1. The service first gets the latest chain tables from the cluster manager.
2. Before starting recovery, it ensures all its targets are marked offline in the latest chain tables.
3. For each storage target, the predecessor in the chain detects that its successor is online and initiates data synchronization.
4. The predecessor sends a "dump-chunkmeta" request to get information about chunks on the recovering target.
5. It then compares the local and remote chunk metadata to determine which chunks need to be transferred.
6. The `ResyncWorker` class handles the synchronization of chunks between targets.
7. The recovery process applies a set of rules to determine which chunks to transfer:
   - If a chunk only exists on the local target, it should be transferred
   - If a chunk exists on both targets but has different versions, the one with the higher chain version takes precedence
   - If chain versions match but commit versions differ, the chunk is transferred

Once data recovery is complete, the storage service updates the local state of the target to "up-to-date" in heartbeat messages, which triggers the cluster manager to transition its public state to "serving".

## Special Cases

The system handles special cases like when all targets in a chain fail. In such situations:
- The first target becomes "lastsrv" to maintain the chain history
- When it recovers, it transitions to "serving" state
- Other targets must synchronize with it before becoming "serving"

This ensures that even in catastrophic failures, as long as one target recovers, the chain can be fully restored.