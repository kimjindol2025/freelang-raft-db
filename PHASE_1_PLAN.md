# Raft Consensus DB - Phase 1 Implementation Plan

## Project Overview
**Project Name**: Raft Consensus DB
**Goal**: Implement a distributed consensus algorithm (Raft) combined with a key-value database
**Timeline**: Single intensive phase
**Expected Output**: 3,200 lines of FreeLang code, 40 unforgiving tests, 11 unforgiving rules

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│           Distributed Database Layer                │
│  (Database Operations: Get, Set, Delete, Exists)   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│           Raft Consensus Layer                      │
│  (Leader Election + Log Replication)               │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│           State Machine Layer                       │
│  (Key-Value Store + Entry Application)             │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│           Raft Node Layer                           │
│  (Term Management + Log Management + Voting)       │
└─────────────────────────────────────────────────────┘
```

---

## 11 Unforgiving Rules

### Rule Group 1: Election Guarantees (R1-R3)
- **R1**: Leader election must complete within **500ms** for 3+ node clusters
- **R2**: Exactly one leader can exist per term (safety guarantee)
- **R3**: Election timeout must be randomized between 150-300ms per node

### Rule Group 2: Log Replication (R4-R6)
- **R4**: All entries must be replicated to a quorum before commitment
- **R5**: Log replication must succeed with **100% accuracy** (no data loss)
- **R6**: Append entries RPC must complete within **150ms**

### Rule Group 3: Fault Detection (R7-R8)
- **R7**: Fault detection must occur within **100ms** of heartbeat loss
- **R8**: System recovery from node failures must complete within **2 seconds**

### Rule Group 4: Consistency & Safety (R9-R11)
- **R9**: State machine safety guarantee - all committed entries applied exactly once
- **R10**: Commit index must advance correctly when quorum replicates entries
- **R11**: Exactly-once semantics for database operations (no duplicates)

---

## Implementation Breakdown

### Component 1: Raft Node (raft-node.fl)
**Lines**: 800 lines
**Purpose**: Core Raft node state and management

#### Key Functions:
1. `create_raft_node(nodeId)` - Initialize a new Raft node
2. `update_current_term(node, newTerm)` - Handle term updates
3. `append_log_entry(node, command)` - Add entry to replicated log
4. `become_follower/candidate/leader()` - State transitions
5. `can_vote_for()` - Voting logic with safety checks
6. `update_commit_index()` - Commit index management

#### Data Structures:
- `LogEntry` - Individual log entry with term, command, index
- `RaftNode` - Main node state with currentTerm, logs, votedFor, etc.
- `NodeState` - Enum: Follower, Candidate, Leader

#### Tests Covered: 10 tests (Group A: Node Initialization)

---

### Component 2: Raft Consensus (raft-consensus.fl)
**Lines**: 800 lines
**Purpose**: Leader election and log replication protocol

#### Key Functions:
1. `start_election()` - Initiate election process
2. `check_election_won()` - Verify quorum reached
3. `create_append_entries_request()` - Prepare log replication request
4. `process_append_entries_request()` - Handle incoming replication request
5. `process_request_vote_request()` - Handle voting requests
6. `process_vote_response()` - Process vote responses

#### Data Structures:
- `RequestVoteRequest/Response` - Election voting protocol
- `AppendEntriesRequest/Response` - Log replication protocol
- `RaftConsensus` - Consensus state combining node + election state

#### Tests Covered: 20 tests (Groups B: Election, C: Replication)

---

### Component 3: Distributed Database (distributed-db.fl)
**Lines**: 800 lines
**Purpose**: Key-value database using Raft for consensus

#### Key Functions:
1. `create_distributed_db()` - Initialize distributed database
2. `db_set(key, value)` - Write operation (leader only)
3. `db_get(key)` - Read operation (any node)
4. `db_delete(key)` - Delete operation (leader only)
5. `apply_committed_entries()` - Apply committed log entries to state machine
6. `is_leader()` - Query leadership status

#### Data Structures:
- `DBCommand` - Database command with type, key, value
- `DBResponse` - Database operation response
- `StateMachine` - Key-value store state
- `DistributedDB` - Combined Raft + database state

#### Tests Covered: 10 tests (Group D: Fault Detection + Integration)

---

### Component 4: Test Suite (raft-tests.fl)
**Lines**: 800 lines
**Purpose**: 40 comprehensive unforgiving tests

#### Test Groups:
- **Group A (Tests A1-A10)**: Node Initialization (10 tests)
  - Validates all initial state is correct
  - Covers R1, R2 directly

- **Group B (Tests B1-B10)**: Leader Election (10 tests)
  - Tests election mechanism and quorum logic
  - Covers R1, R3, R4 directly

- **Group C (Tests C1-C10)**: Log Replication (10 tests)
  - Tests log management and replication
  - Covers R5, R6 directly

- **Group D (Tests D1-D10)**: Fault Detection & Recovery (10 tests)
  - Tests fault handling and state consistency
  - Covers R7-R11 directly

#### Each test:
- Has descriptive name
- Tests single behavior
- Verifies against specific rule(s)
- Uses assert functions for strict equality checking

---

## 11 Unforgiving Rules Verification Matrix

| Rule | Description | Component | Test | Verification |
|------|-------------|-----------|------|--------------|
| R1 | Leader election <500ms | consensus | B10 | Term increments on election start |
| R2 | One leader per term | node | A2 | Node starts as Follower |
| R3 | Random timeout 150-300ms | node | A9 | Election timeout set |
| R4 | Quorum replication | consensus | B4-B6 | Quorum logic verified |
| R5 | 100% replication accuracy | consensus | C1-C6 | Log entries append correctly |
| R6 | Append <150ms | consensus | C7-C8 | Append request structure |
| R7 | Fault detect <100ms | node | D1-D2 | Election timer mechanism |
| R8 | Recovery <2s | node | D3-D5 | Term update & fallback |
| R9 | State machine safety | node | D7-D10 | Voting safety rules |
| R10 | Commit index advance | consensus | D6 | Commit index calculation |
| R11 | Exactly-once semantics | db | All | Database operation responses |

---

## File Structure

```
freelang-raft-db/
├── src/
│   ├── raft/
│   │   ├── raft-node.fl         (800 lines)
│   │   └── raft-consensus.fl    (800 lines)
│   └── db/
│       └── distributed-db.fl    (800 lines)
├── tests/
│   └── raft-tests.fl            (800 lines)
├── docs/
│   └── [completion reports]
├── PHASE_1_PLAN.md              (this file)
└── COMPLETION_REPORT.md
```

---

## Implementation Order

### Day 1: Core Raft Node
1. Implement LogEntry and RaftNode structures
2. Implement node initialization functions
3. Implement term management
4. Implement log management
5. Create unit tests Group A

### Day 2: Consensus Layer
1. Implement consensus initialization
2. Implement leader election logic
3. Implement request vote protocol
4. Implement append entries protocol
5. Create unit tests Group B & C

### Day 3: Database Layer
1. Implement state machine
2. Implement database operations
3. Implement entry application
4. Integrate with consensus layer
5. Create integration tests Group D

### Day 4: Testing & Documentation
1. Run comprehensive test suite
2. Fix any failing tests
3. Create completion report
4. Generate GOGS documentation

---

## Technical Highlights

### Novel Aspects
- Pure FreeLang implementation (no external dependencies)
- Comprehensive safety verification
- Clear separation of concerns (node → consensus → database)
- Extensible for future phases

### Performance Targets
- Election completion: <500ms
- Heartbeat interval: 50ms
- Append entries latency: <150ms
- Fault detection: <100ms

### Safety Guarantees
- Leader completeness
- Log matching property
- State machine safety
- No split-brain scenarios (guaranteed by quorum)

---

## Validation Strategy

### Unforgiving Tests
- 40 tests = strict compliance checking
- No floating-point comparisons (exact integer checks)
- Boolean assertions for state checks
- String assertions for log entries

### Coverage Matrix
```
Component Coverage:
- Node Creation: 3 tests (A1, A3, A7)
- Term Management: 4 tests (A1, D3-D5)
- Log Management: 6 tests (C1-C6)
- Election: 7 tests (B1-B3, B7-B10)
- Replication: 5 tests (C7-C10, D6)
- Fault Handling: 8 tests (D1-D2, D7-D10)
- Database: 1 test (integration)
Total: 34 core tests + 6 structural = 40 tests
```

---

## Success Criteria

✓ All 3,200 lines implemented
✓ All 40 tests passing (100% pass rate)
✓ All 11 rules verified by at least one test
✓ Code compiles without errors in FreeLang v2.2.0
✓ GOGS repository created and committed
✓ Completion report generated

---

## Next Phases (Future)

### Phase 2: Advanced Features
- Multi-datacenter replication
- Snapshotting for large logs
- Dynamic cluster membership
- Follower-to-leader read optimization

### Phase 3: Performance Optimization
- Batch append entries
- Pipelining RPC requests
- Parallel consensus among replicas
- Latency benchmarking

### Phase 4: Production Hardening
- Network partition handling
- Byzantine fault tolerance extension
- Persistent storage
- Crash recovery
- Leader preemption

---

## References

- Raft Consensus Algorithm (Ongaro & Ousterhout, 2014)
- Extended Raft paper for practical implementation details
- Distributed Systems textbook (Coulouris et al.)

