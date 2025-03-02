# Metadata Operations in 3FS

## Cluster-Level Metadata Operations (File Creation)

When a client creates a file and writes data to it, the following metadata operations occur at the cluster level:

1. **Client Initiates File Creation**:
   - The client sends a `CreateReq` request to the metadata service.
   - The metadata service first authenticates the user via the `AUTHENTICATE` macro.
   - It validates the request parameters using the `req.valid()` function.

2. **Metadata Service Handling**:
   - First, the service attempts to open the file via `tryOpen()` to check if it already exists.
   - If the file exists, it returns the existing file information.
   - If not, the service proceeds with the creation process.

3. **Distributed Handling**:
   - The metadata service uses a `Distributor` to determine which server should handle the request.
   - If the current server is responsible, it processes the request itself.
   - Otherwise, it forwards the request to the appropriate server via `forward_->forward<CreateReq, CreateRsp>`.

4. **File Creation Transaction**:
   - The file creation is performed within a transaction using the key-value store (FoundationDB).
   - The metadata operation runs in a batch via `runInBatch<CreateReq, CreateRsp>`.
   - The service executes an `OpenOp` operation, which handles both file opening and creation.

5. **Inode and Directory Entry Creation**:
   - The service allocates a new inode ID via `allocateInodeId()`.
   - It creates a directory entry that links the file name to the inode.
   - It initializes a new inode with file attributes, permissions, and layout information.
   - Both inode and directory entry are stored in the KV store via `await entry.store(txn)` and `await inode.store(txn)`.

6. **Chain Allocation for File Data**:
   - The service allocates chains from a chain table for the file data.
   - It selects chains based on the stripe size and shuffles them using a seed for balanced distribution.
   - The file's inode contains this layout information, which determines where data chunks will be stored.

7. **Session Management**:
   - If the file is opened for writing, a file session is created via `createSession()`.
   - The session tracks the open file descriptor for write operations.
   - Session information is stored in the KV store for consistency.

8. **Event Tracking and Logging**:
   - The service records file creation events for monitoring via `addEvent(Event::Type::Create)`.
   - It logs metadata trace information via `addTrace()`.

## Storage Node-Level Operations (Data Writing)

Once the file is created and opened, data writing involves the following operations:

1. **Chunking and IO Preparation**:
   - The file is divided into equally sized chunks.
   - For each write operation, the client creates a `WriteIO` object using `createWriteIO()`.
   - This object contains the chain ID, chunk ID, offset, length, and pointer to the data.

2. **Client-Side Data Handling**:
   - The client prepares RDMA buffers for data transfer.
   - It uses the storage client to send write requests via `write()` or batch operations via `batchWrite()`.
   - These calls are asynchronous coroutines, represented as `CoTryTask<void>`.

3. **Chain Replication Protocol**:
   - Write requests are sent to the head target of the assigned chain.
   - The storage service handles the CRAQ protocol for data replication.
   - The implementation follows the write-all-read-any approach described in the design notes.

4. **Storage Target Processing**:
   - Each storage target:
     - Receives the write request
     - Acquires a lock for the chunk
     - Allocates space for the chunk via the chunk engine
     - Applies the update to the chunk data
     - Updates the chunk's metadata (version, checksum, etc.)
     - Forwards the request to the next target in the chain (if not the tail)

5. **Metadata Updates at Storage Targets**:
   - Each storage target maintains chunk metadata in RocksDB.
   - This metadata includes:
     - Chunk ID
     - Chunk length
     - Update version
     - Commit version
     - Checksum
     - Chain version

6. **Write Completion**:
   - The tail target commits the write and sends an acknowledgment back through the chain.
   - Each target in the chain updates its copy of the chunk metadata.
   - The client receives a success response when the write is fully committed.

7. **File Length Updates**:
   - Periodically, clients report the maximum write position to the metadata service.
   - The metadata service updates the file length in the inode if no concurrent truncate operation exists.
   - On close/fsync, the metadata service obtains the precise file length by querying storage services.

## Summary of Key Interactions

1. **Metadata and Storage Separation**:
   - Metadata operations (file creation, attributes) are handled by the metadata service.
   - Data operations (reads/writes) are handled directly between clients and storage targets.
   - The metadata service provides clients with the layout information needed to access data.

2. **Transactional Consistency**:
   - Metadata updates use FoundationDB's transactional capabilities.
   - Data operations use the CRAQ protocol to ensure strong consistency.

3. **Distributed Design**:
   - The system uses a distributed metadata service architecture.
   - Requests are routed to the appropriate metadata server based on the inode.
   - Storage targets are organized in chains that span multiple physical servers.

This architecture allows 3FS to provide a file system interface with strong consistency guarantees while achieving high performance through the direct client-to-storage data path.