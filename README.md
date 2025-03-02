# DeepSeek-3FS explained in $6.81

DeepSeek has recently open-sourced [3FS](https://github.com/deepseek-ai/3FS), a distributed file system designed for both LLM training (data and checkpointing) and inference (KV cache). The system achieves an impressive 6.6TiB/s across 180 nodes, which translates to approximately 300Gbps per node. The architecture incorporates FoundationDB for cluster metadata, RocksDB for storage node metadata, and CRAQ for consistent replication during concurrent reads. Beyond studying the [design nodes](https://github.com/deepseek-ai/3FS/blob/main/docs/design_notes.md), one can gain a deeper understanding of 3FS by delving into the source code.

Here, we enlisted [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code) to study the 3FS source code and provide detailed insights. The total cost of API usage running Claude Code amounts to $6.81.
