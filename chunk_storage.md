# 3FS Chunk Storage Architecture

This document explains how storage nodes in 3FS pack chunks into physical data files on SSDs.

## Overview

3FS storage nodes efficiently organize logical chunks (units of file data) into physical container files on disk. The system uses a two-level storage hierarchy with multiple fixed chunk sizes to optimize for different file access patterns while maintaining efficient storage utilization.

## Physical Storage Organization

### Chunk Size Hierarchy

- The system uses a configurable set of fixed chunk sizes (default: 512KB, 1MB, 2MB, 4MB, 16MB, 64MB)
- Each chunk size has its own directory structure
- Maximum chunk size is 64MB (`kMaxChunkSize`)
- Chunks are assigned to the appropriate size category during allocation

### File Structure

- Physical storage is organized into a two-level hierarchy:
  - First level: Directories named after chunk sizes (e.g., "512KB", "1MB", "64MB")
  - Second level: Hexadecimal-named files (00-FF) that act as containers for chunks
- Default configuration creates 256 physical files per chunk size (`physical_file_count`)
- Each container file can hold multiple chunks of the same size

### Chunk Storage Format

- Chunks are packed into container files at specific offsets
- Each chunk has a fixed size allocation but can contain variable amounts of actual data
- Metadata keeps track of which physical file contains each chunk and at which offset
- Direct I/O is used when possible to bypass OS caching for better performance

## Chunk Metadata

### Chunk Identification

- Each chunk has a unique `ChunkId` (128-bit identifier)
- Chunks are logically organized into chains identified by `ChainId`
- Each chunk has a version (`ChunkVer`) and state (`ChunkState`)

### Metadata Storage

- Metadata is stored in a key-value store (LevelDB by default, configurable)
- The KV store is maintained in a directory named by the `kv_store_name` config (default: "meta")
- Metadata includes physical file location, offset, size, version, and checksum information

## Data Operations

### Read Operations

- Direct `pread` from physical files at specific offsets
- Support for both buffered and direct I/O (`normal_` and `direct_` file descriptors)
- AIO (asynchronous I/O) for efficient concurrent reading

### Write Operations

- Use `pwrite` to physical files at specific offsets
- Direct I/O is used when possible (when aligned properly)
- Write operations include version and checksum tracking
- Includes retry logic with exponential backoff

### Space Management

- `fallocate` used to allocate space for new chunks
- `FALLOC_FL_PUNCH_HOLE` used to reclaim space (marks as available but doesn't reduce file size)
- Support for emergency recycling during low space conditions
- Chunks can be garbage collected after deletion

## Benefits of This Approach

1. **Efficient Space Utilization**: Packing multiple chunks into container files reduces fragmentation
2. **Performance Optimization**: Different chunk sizes support various access patterns
3. **Scalability**: Container-based approach scales to handle billions of chunks
4. **Resource Management**: Separate metadata and data paths optimize resource usage
5. **Data Integrity**: Version tracking and checksums ensure data correctness