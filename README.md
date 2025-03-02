# DeepSeek-3FS explained in $6.81

DeepSeek has recently open-sourced [3FS](https://github.com/deepseek-ai/3FS), a distributed file system designed for both LLM training (data and checkpointing) and inference (KV cache). The system achieves an impressive 6.6TiB/s across 180 nodes, which translates to approximately 300Gbps per node. The architecture incorporates FoundationDB for cluster metadata, RocksDB for storage node metadata, and CRAQ for consistent replication during concurrent reads. Beyond studying the [design nodes](https://github.com/deepseek-ai/3FS/blob/main/docs/design_notes.md), one can gain a deeper understanding of 3FS by delving into the source code.

Technical deep dives:
* [metadata operations when creating and writing a file](metadata_operations.md)
* [chunks rebuilt after storage target failure](data_durability.md)
* [consistency during concurrent reads and writes](read_write_consistency.md)
* [atomicity of moving a directory](directory_move.md)
* [chunks packed into physical files on SSD](chunk_storage.md)

Specifications and invariants:
* [replication logic](storage_verification.md)
* [RDMA protocol](rdma_network_specification.md)  

Here, we enlisted [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code) to study the 3FS source code and provide detailed insights. The total cost of API usage running Claude Code amounts to $6.81. Claude Code is agentic, so all we need is to ask questions.

![3fs_durability](https://github.com/user-attachments/assets/0423379f-e6ac-461f-8bb2-3e4bd4f2da02)
