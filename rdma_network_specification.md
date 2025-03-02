# RDMA Network Specification in 3FS

This document outlines the formal modeling and verification of the RDMA (Remote Direct Memory Access) networking layer in 3FS.

## Overview

3FS employs formal verification using the P language to model and verify its RDMA communication protocol. The formal model ensures correctness, proper flow control, and reliability of the network layer. This is particularly important for RDMA, which provides low-latency, high-throughput data transfer by allowing direct access to memory on remote systems without operating system involvement.

## Components Modeled

### 1. QueuePair

The model represents RDMA queue pairs, which are the fundamental communication channel in RDMA:

- **Work Request Management**: Handles send and receive work requests
- **Completion Queues**: Manages send and receive completion queues
- **Work Request to Completion Transitions**: Models the lifecycle of requests

```p
machine QueuePair {
  var maxNumSendWRs: int;
  var maxNumRecvWRs: int;
  var postedRecvWRs: seq[tWorkRequest];
  var postedSendWRs: seq[tWorkRequest];
  var sendCompQueue: seq[tWorkComplete];
  var recvCompQueue: seq[tWorkComplete];
  var outboundQueue: seq[tXmitPacket];
  var inboundQueue: seq[tXmitPacket];
  // ...
}
```

### 2. RDMASocket

Provides a higher-level socket-like interface on top of the RDMA queue pairs:

- **Buffer Management**: Handles allocation and recycling of buffers
- **Flow Control**: Implements credit-based flow control with acknowledgments
- **Send/Receive Operations**: Provides byte-stream operations to applications

```p
machine RDMASocket {
  var qp: QueuePair;
  var bufSize: int;
  var bufNum: int;
  var flowCtrlBufNum: int;
  var unusedSendBufs: seq[tTaggedBuffer];
  var remotePostedBufNum: int;
  // ...
}
```

### 3. Network

Simulates the RDMA network fabric that connects endpoints:

- **Packet Exchange**: Handles transmission of packets between queue pairs
- **Connection Management**: Manages the establishment of connections
- **Packet Routing**: Routes packets to the appropriate destination

```p
machine Network {
  var qps: seq[QueuePair];
  // ...
}
```

## Key Properties and Invariants

The formal verification focuses on proving several critical properties:

### 1. RecvComplete

This property ensures that all bytes sent are eventually received correctly. It verifies that the system never loses data and that all sent data is properly accounted for.

```p
spec RecvComplete observes eSendBytes, eRecvBytes, eRecvBytesResp {
  // Checks received bytes never exceed sent bytes
  assert recvBytes <= sentBytes, format("error: {0} recv bytes > {1} sent bytes", recvBytes, sentBytes);
  // Ensures no pending receive bytes when in stable state
  assert pendingRecvBytes == 0, format("{0} pending recv bytes not equal to zero", pendingRecvBytes);
}
```

### 2. NoDuplicatePostedBuffers

This property ensures the integrity of buffer management, verifying that the same buffer is never posted multiple times, which could lead to memory corruption.

```p
spec NoDuplicatePostedBuffers observes ePostSend, ePostRecv, ePollSendCQReturn, ePollRecvCQReturn {
  // Verifies no buffer is posted twice
  assert !(wr.wrIdx in postedRecvBufs), format("buffer with index {0} already posted", wr.wrIdx);
  // Ensures only previously posted buffers are completed
  assert wc.wrIdx in postedRecvBufs, format("unexpected buffer index {0} returned", wc.wrIdx);
}
```

### 3. AllIterationsProcessed

This property verifies that all expected communication iterations complete successfully, ensuring the system doesn't deadlock or stall.

```p
spec AllIterationsProcessed observes eSendBytes, eRecvBytesResp, eSystemConfig {
  // Checks that all iterations complete
  if (CheckStopCondition(sendIters) && CheckStopCondition(recvIters)) {
    goto Done;
  }
}
```

## Verification Scenarios

The formal verification tests these properties under various communication patterns:

1. **Ping-Pong Communication**: Tests bidirectional back-and-forth communication
2. **One-Way Communication**: Tests unidirectional data transfer
3. **Two-Way Simultaneous Communication**: Tests concurrent bidirectional data transfer

Each test scenario is designed to verify different aspects of the RDMA protocol, ensuring it works correctly under various usage patterns.

## Buffer Management and Flow Control

A critical aspect of the formal model is its representation of RDMA buffer management and flow control:

1. **Buffer Posting**: Ensuring buffers are properly allocated and posted for receive operations
   ```p
   send qp, ePostRecv, (wrIdx = sockId * bufNum * 2 + i, opcode = WROpCode_INVALID, payload = InitBytes(bufSize, 0), length = bufSize, imm = 0);
   ```

2. **Credit-Based Flow Control**: Managing available receive buffers on the remote side
   ```p
   if (numRecvSinceLastAck == numRecvBeforeAck) {
     send qp, ePostSend, (wrIdx = -1, opcode = WROpCode_SEND_WITH_IMM, payload = default(tBytes), length = 0, imm = numRecvSinceLastAck);
     numRecvSinceLastAck = 0;
   }
   ```

3. **Work Completion**: Detecting completed operations and recycling buffers
   ```p
   if (wc.status == Status_OK) {
     if (wc.opcode == WCOpCode_SEND) {
       if (wc.wrIdx >= 0) {
         sendBuf = (bufIdx = wc.wrIdx, payload = wc.payload, length = bufSize);
         unusedSendBufs += (sizeof(unusedSendBufs), sendBuf);
       }
     }
   }
   ```

## Benefits of Formal Verification

The formal modeling of RDMA provides several key benefits for 3FS:

1. **Protocol Correctness**: Ensuring the RDMA protocol implementation is free from deadlocks, race conditions, and data corruption

2. **Buffer Management Verification**: Confirming that buffer allocation, use, and recycling are handled correctly, preventing memory leaks and corruption

3. **Flow Control Validation**: Verifying that the flow control mechanism prevents overwhelming receivers while maintaining high throughput

4. **Concurrency Handling**: Ensuring correct behavior when multiple operations occur simultaneously

5. **Completion Notification**: Validating that completion notifications are correctly processed, allowing efficient resource reuse

## Conclusion

The formal modeling of the RDMA protocol in 3FS provides mathematical guarantees that the system's networking layer maintains critical properties like data integrity, completion, and buffer management. This ensures high reliability and correctness in a complex distributed environment with demanding performance requirements.