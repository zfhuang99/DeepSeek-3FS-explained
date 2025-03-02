# Formal Verification in 3FS

This document outlines how 3FS uses formal methods to verify critical system properties and invariants, particularly focusing on the Chain Replication with Apportioned Queries (CRAQ) implementation.

## Overview

3FS employs formal verification using the P language to model and verify key components of the distributed storage system. P is a state machine-based language designed for specifying and verifying distributed systems. The formal models in 3FS focus primarily on:

1. Ensuring data consistency during normal operations
2. Maintaining correctness during failure scenarios
3. Verifying recovery mechanisms work as expected

## Components Modeled

The formal specifications model several critical components of the 3FS system:

- **MgmtService**: Management service that handles cluster configuration and chain management
- **StorageService**: Service responsible for storage operations and replication
- **StorageClient**: Client interface for storage operations
- **MgmtClient**: Client interface for management operations
- **StorageTarget** and **ChunkReplica**: Models of physical storage entities
- **WriteProcess** and **ReadProcess**: Models of read/write operation protocols
- **SyncWorker**: Handles synchronization during recovery

## Key Properties and Invariants

The formal verification focuses on proving several critical properties:

### 1. WriteComplete

This property ensures that all write operations initiated by clients eventually complete. The model verifies that there are no scenarios where write operations can get "stuck" in the system without resolution.

```p
spec WriteComplete observes eWriteReq, eWriteResp {
  // Verification that all write requests receive responses
}
```

### 2. MonotoneIncreasingVersionNumber

This property verifies that chunk version numbers always increase monotonically, maintaining strong consistency semantics. The model checks that no replica can have a higher version number followed by a lower one.

```p
spec MonotoneIncreasingVersionNumber observes eWriteOpFinishResult, eCommitOpResult {
  // Verification that version numbers always increase
  assert commitVer > chunkVer,
    format("current commit version {0} <= previous version {1}...", commitVer, chunkVer);
}
```

### 3. AllReplicasOnChainUpdated

This property ensures that all replicas in a chain eventually receive updates and have consistent data. It verifies that when a write operation completes, all appropriate replicas have the correct version of the data.

```p
spec AllReplicasOnChainUpdated observes eReadWorkDone, eWriteWorkDone, eCommitWorkDone, eNewRoutingInfo {
  // Verification that all replicas receive updates
}
```

### 4. AllReplicasInServingState

This property checks that all replicas eventually return to a serving state after failures or during recovery operations. It ensures the system can automatically recover from failures.

```p
spec AllReplicasInServingState observes eNewRoutingInfo, eSyncStartReq, eSyncDoneResp, eStopMonitorTargetStates {
  // Verification that replicas return to serving state
}
```

## Verification Scenarios

The formal verification tests these properties under various scenarios:

1. **Normal Operations**:
   - One client writing with no failures
   - Multiple clients writing with no failures

2. **Failure Scenarios**:
   - Unreliable failure detector
   - Single storage service failure
   - Multiple concurrent storage service failures

3. **Chain Configuration Variations**:
   - Short chains with failures
   - Long chains with failures

The model checker exhaustively explores all possible states within these scenarios to ensure the properties are maintained under all conditions.

## Benefits of Formal Verification

The formal modeling approach provides several key benefits for 3FS:

1. **Mathematical Guarantees**: Properties are proven correct through mathematical model checking, providing stronger assurances than traditional testing.

2. **Exhaustive Exploration**: The model checker explores all possible state combinations, including edge cases that might be missed in traditional testing.

3. **Early Detection**: Design flaws can be identified early in the development cycle, before implementation.

4. **Precise Specifications**: The formal models serve as precise specifications for the system's behavior.

5. **Confidence in Fault Tolerance**: Formal verification provides confidence that the system's fault tolerance mechanisms work correctly.

## Implementation Details

The formal verification is implemented using the P language and its associated tools:

1. **P Language**: A state machine-based language for specifying and verifying distributed systems.

2. **Model Checking**: The P compiler and checker exhaustively explore the state space to verify properties.

3. **Test Scenarios**: Defined in `PTst/TestScript.p`, covering various operational conditions.

4. **System Models**: Defined in `PSrc/` directory files like `StorageService.p`, `MgmtService.p`, etc.

5. **Property Specifications**: Defined in `PSpec/SystemSpec.p`.

All tests in the current implementation pass, indicating that the CRAQ implementation in 3FS maintains its critical properties under the modeled scenarios.