# Directory Move Operations in 3FS

This document explains how directory move (rename) operations are implemented in 3FS, with a focus on transaction handling and concurrency control.

## Overview

In 3FS, directory move operations are implemented as atomic transactions using FoundationDB as the underlying storage system. This ensures that the entire operation either completes fully or fails without leaving the filesystem in an inconsistent state. The implementation follows POSIX semantics rather than HDFS semantics.

## Transaction Implementation

Directory moves are handled by the `RenameOp` class in `src/meta/store/ops/Rename.cc`. The operation executes the following steps within a single transaction:

1. **Path Resolution**: 
   - Resolves both source and destination paths concurrently
   - Verifies the source exists and matches the expected inode ID if provided
   - Checks if the destination already exists

2. **Destination Handling**:
   - If destination is a non-empty directory: fails with `kNotEmpty` error
   - If destination is an empty directory: removes the directory
   - If destination is a file: marks it for garbage collection
   - If destination is a symlink: decreases its link count

3. **Loop Detection**:
   - Checks all destination ancestors to ensure the operation doesn't create a cycle
   - Prevents moving a directory into one of its descendants
   - Detects attempts to move into deleted directories

4. **Permission Verification**:
   - Checks write permission on both source and destination parent directories
   - Verifies directory locks (preventing concurrent modification)
   - Handles sticky bit semantics (S_ISVTX)
   - Checks immutable flag (FS_IMMUTABLE_FL)

5. **Directory Update**:
   - For directories, updates the parent pointer in the inode (stores new parent ID and name)
   - Removes the source directory entry
   - Removes the destination entry if it exists
   - Creates a new directory entry at the destination

6. **Special Handling for Trash Directory**:
   - Implements special checks when moving to/from the trash directory
   - Tracks original path information in trace logs for recovery purposes

## Concurrency Control

3FS handles concurrent operations through several mechanisms:

### 1. Optimistic Concurrency Control

The system uses FoundationDB's Serializable Snapshot Isolation (SSI):
- Each transaction tracks its read/write sets for conflict detection
- If two concurrent transactions modify overlapping sets, one will be automatically aborted and retried
- The implementation adds critical paths to the read conflict set to ensure conflicts are detected

### 2. Explicit Conflict Detection

The `RenameOp` implementation explicitly adds key metadata to the read conflict set:
```cpp
CO_RETURN_ON_ERROR(co_await Inode(srcResult->getParentId()).addIntoReadConflict(txn));
CO_RETURN_ON_ERROR(co_await srcResult->dirEntry->addIntoReadConflict(txn));
CO_RETURN_ON_ERROR(co_await Inode(dstResult->getParentId()).addIntoReadConflict(txn));
CO_RETURN_ON_ERROR(
    co_await DirEntry(dstResult->getParentId(), req_.dest.path->filename().native()).addIntoReadConflict(txn));
```

This ensures that any concurrent modifications to these key elements will trigger transaction conflicts.

### 3. Directory Locking

The implementation checks if directories are locked by any client:
```cpp
CO_RETURN_ON_ERROR(parent->asDirectory().checkLock(req_.client));
```

This prevents race conditions from concurrent operations.

### 4. Idempotent Operations

Rename operations support idempotency through client ID and request ID tracking:
```cpp
bool needIdempotent(Uuid &clientId, Uuid &requestId) const override {
  if (!req_.checkUuid()) return false;
  if (!req_.moveToTrash && !config().idempotent_rename()) return false;
  clientId = req_.client.uuid;
  requestId = req_.uuid;
  return true;
}
```

This prevents duplicate operations if a transaction returns "commit_unknown_result".

## Handling Concurrent File Creation

When a concurrent file creation happens in the same directory as a move operation:

1. The file creation transaction adds directory entries to its write set
2. The directory move transaction adds parent directories to its read conflict set
3. FoundationDB detects the conflict between read and write sets
4. One transaction (usually the later one) fails with a conflict error
5. The failed transaction is automatically retried with the latest state of the directory

This approach ensures consistency without explicit locking, even during concurrent operations.

## Error Handling and Retries

The `OperationDriver` class handles transaction retries:
- Encapsulates retry logic with exponential backoff
- Preserves idempotency during retries
- Properly handles timeouts and non-retryable errors

## Benefits of the Transaction Approach

By implementing directory moves as FoundationDB transactions, 3FS achieves:

1. **Full atomicity**: All steps succeed or none do
2. **Strong consistency**: No partial updates are visible
3. **Isolation**: Concurrent operations don't interfere with each other
4. **Durability**: Once committed, changes persist through system failures
5. **Safety**: Loop detection and permission checks within the transaction