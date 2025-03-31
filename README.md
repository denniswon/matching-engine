# Highly-Available Order Matching Engine

A fault-tolerant matching engine implementation using the Raft consensus protocol for state machine replication.

## Overview

This project implements a highly available matching engine with a simplified interface. It handles buy, sell and cancel orders and stores outstanding orders in a replicated order book. It acknowledges to clients when their orders have been placed and notifies them of trades occurring through the order book. The replicated matching engine tolerates machine and network failures, including partitions in the network.

## Features

- **Consistency**: The matching engine provides a consistent view of the state it stores, with nodes never communicating inconsistent results to clients.
- **Failure-tolerance**: Can tolerate f machine failures when run on a cluster of 2f + 1 machines.
- **Fairness**: Actively ensures each client has equal opportunity to trade on the market.
- **Durability**: Orders and trades that are acknowledged to clients are persistently stored.

## Architecture

The system is composed of three main components:
1. **Client Port**: Handles incoming client requests
2. **Matching Engine**: Contains the business logic to find trades between orders
3. **Raft Module**: Provides a consistent log for state machine replication

### Repository Structure

The code for the matching engine is divided into two modules: **engine** and **raft**.

The raft component contains code for state-machine replication using the Raft protocol, while the engine component depends on the raft module to build a replicated matching engine and a client port.

### Implementation Details

#### Timeline of an Order

When a client places an order on the market:
1. The exchange's FIX engine passes this request to a client port
2. The port broadcasts the order request in the internal network of the exchange
3. The leader replica processes the order through the Raft module for replication
4. The replication process occurs in two stages:
   - First, the leader sends AppendEntries requests to other replicas
   - Once a majority of replicas acknowledge the entry, the leader commits it
5. When committed, the order is delivered to the local matching engine instance
6. The matching engine checks for possible trades and updates the order book
7. The leader broadcasts acknowledgements and trade notifications to clients

#### Raft Implementation

The Raft component maintains these key states:
- `current_term`: Term number, incremented during re-elections
- `voted_for`: Node ID voted for in the current term
- `commit_index`: Index of the last log entry known to be committed
- `last_applied`: Index of the last log entry applied to the state machine
- `log`: Array of log entries with commands to apply
- `next_index`: For leader nodes, tracks next entry to send to each replica
- `match_index`: For leader nodes, tracks last entry known to be replicated

The implementation uses multiple threads:
- RPC thread: Handles incoming RPC requests from other nodes
- Heartbeat thread: Sends periodic heartbeats (for leaders)
- Re-election thread: Monitors leader timeouts and initiates elections
- Client thread: Handles incoming client requests

#### Matching Engine

The matching engine is implemented as a deterministic state machine with two versions:

1. **Baseline Order Book**: 
   - Uses two priority queues (binary heaps) for buy and sell sides
   - Orders sorted primarily by price, secondarily by timestamp
   - Provides O(log n) complexity for operations

2. **Optimized Order Book**:
   - Groups orders by price into buckets as circular arrays
   - Indexes buckets using an Adaptive Radix Tree
   - Maintains a hashmap from order ID to price for faster cancellations
   - Achieves O(1) for apply operations and O(T) for checking trades
   - The Adaptive Radix Tree adaptively changes node sizes (4, 16, 48, or 256 children)
   - Compresses key prefixes for space efficiency

#### RPC Communication

The project uses gRPC for communication between components:
- `limit_order_queue`: Streaming RPC for sending orders to the engine
- `register_order_acknowledgements`: For receiving order acknowledgements
- `register_trades`: For receiving trade notifications
- `request_vote`: Used in Raft for leader election
- `append_entries`: Used in Raft for log replication

#### Failure Tolerance

The system implements exactly-once semantics using:
- Sequence numbers for client requests
- Retry mechanism for at-least-once delivery
- Deduplication in the matching engine for at-most-once processing

The state machine tracks acknowledged orders and detects duplicates in two places:
1. When new orders arrive at the leader
2. When replicated orders are applied to the matching engine

## Performance

The system achieves high throughput and low latency while maintaining consistency guarantees:

- **Failover latency**: 150ms.Adjustable based on re-election timeout configuration
- **Matching latency**: Optimized implementation achieves 40-200 nanoseconds matching latency
- **End-to-end latency**: 200ms with 750rps throughput for a single-client. Comparable to other Raft-based replicated state machines

## Implementation

The project is implemented in Rust, utilizing:
- **Tokio** for async I/O
- **gRPC** for efficient RPC communication
- **Adaptive Radix Tree** data structure for order book optimization

## Evaluation

The project has been evaluated against the following criteria:
1. Correct implementation of the Raft protocol
2. Performance compared to other log replication protocols
3. Fault tolerance with 5 nodes (surviving 2 node failures)
4. Latency and throughput under varying network conditions

## Future Work

Potential areas for extension:
- Formal verification using TLA+
- Log compaction, batching and pipelining optimizations
- Advanced trading algorithms such as bucket trading
- Persistent log storage for auditing
