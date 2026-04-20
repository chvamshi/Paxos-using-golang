# Paxos Implementation in Go (Refactored)

A modern, production-ready implementation of the Paxos consensus algorithm in Go with support for Multi-Paxos.

## 📋 Overview

This refactored implementation provides:

- **Proper Package Structure**: Organized codebase with clear separation of concerns
- **Single Server Binary**: Unified entry point instead of separate server files
- **Multi-Paxos Support**: Consensus on multiple independent instances
- **Comprehensive Logging**: Structured logging with debug/info/error levels
- **Extensive Tests**: 7+ test cases with simulated network delays and failure scenarios
- **Failure Handling**: Graceful degradation with majority-based consensus

## 🏗️ Architecture

### Package Structure

```
github.com/chvamshi/paxos/
├── pkg/paxos/
│   ├── types.go         # Core data structures
│   ├── acceptor.go      # Acceptor implementation
│   ├── proposer.go      # Proposer with multi-proposal support
│   ├── learner.go       # Learner for consensus detection
│   └── paxos_test.go    # Comprehensive test suite
├── cmd/server/
│   └── main.go          # Single server binary
├── go.mod              # Module definition
├── Makefile            # Build targets
├── .gitignore          # Git ignore rules
└── README-REFACTORED.md # This file
```

### Core Components

#### **Acceptor** (`pkg/paxos/acceptor.go`)
- Implements Phase 1b and Phase 2b
- Maintains per-instance promise and accept state
- Thread-safe with RWMutex
- Supports multi-Paxos via instance tracking

**Key Methods:**
- `Prepare(req PrepareRequest) PrepareResponse` - Phase 1a response
- `Accept(req AcceptRequest) AcceptResponse` - Phase 2a response
- `GetAcceptedValue(instance int)` - Retrieves accepted values

#### **Proposer** (`pkg/paxos/proposer.go`)
- Implements Phase 1a and Phase 2a
- Manages proposals per instance
- Tracks promises and accepts to determine consensus
- Simulates network delays (10ms per operation)

**Key Methods:**
- `Propose(value string, instance int)` - Initiates a proposal
- `phase1(proposal *ProposalState) bool` - Prepare phase
- `phase2(proposal *ProposalState) bool` - Accept phase

#### **Learner** (`pkg/paxos/learner.go`)
- Detects when consensus is reached (quorum accepts)
- Tracks accept counts per instance
- Thread-safe value storage

**Key Methods:**
- `Learn(req LearnRequest)` - Process learn notifications
- `GetDecidedValue(instance int)` - Retrieve consensus value
- `GetAllDecidedValues()` - Retrieve all decided values

#### **Server** (`cmd/server/main.go`)
- RPC endpoint for distributed deployment
- Unified binary supporting multiple roles
- Configurable ID, port, cluster size, and logging level

## 🚀 Quick Start

### Prerequisites
- Go 1.20+
- GNU Make (optional, for convenience)

### Installation

```bash
# Clone repository
git clone https://github.com/chvamshi/Paxos-using-golang.git
cd Paxos-using-golang

# Checkout refactored branch
git checkout refactor

# Initialize Go module
go mod tidy
```

### Running Tests

```bash
# Run all tests
make test

# Or directly:
go test -v ./pkg/paxos/ -timeout 30s
```

**Test Coverage:**
- `TestSingleProposal` - Basic consensus
- `TestMultipleInstances` - Multi-Paxos with 3 instances
- `TestPreparePhase` - Phase 1 logic
- `TestAcceptPhase` - Phase 2 logic
- `TestSimulatedNetworkDelay` - Network latency simulation
- `TestConcurrentProposals` - Concurrent instance handling
- `TestMajorityAcceptance` - Quorum verification
- `TestInstanceIndependence` - Instance isolation
- `TestProposalIDComparison` - ID comparison logic

### Building the Server

```bash
# Build binary
make build

# Or directly:
go build -o paxos-server ./cmd/server/main.go
```

### Running a Single Server

```bash
# Run with defaults (id=1, port=8081, 3 nodes)
make run-server

# Or with custom parameters:
go run ./cmd/server/main.go -id=1 -port=8081 -nodes=3 -debug
```

**Server Flags:**
- `-id int` - Node ID (default: 1)
- `-port int` - Listen port (default: 8081)
- `-nodes int` - Cluster size (default: 3)
- `-debug` - Enable debug logging (default: false)

### Running a Multi-Node Demo

Open three terminal windows:

**Terminal 1:**
```bash
go run ./cmd/server/main.go -id=1 -port=8081 -nodes=3 -debug
```

**Terminal 2:**
```bash
go run ./cmd/server/main.go -id=2 -port=8082 -nodes=3 -debug
```

**Terminal 3:**
```bash
go run ./cmd/server/main.go -id=3 -port=8083 -nodes=3 -debug
```

## 📊 Paxos Algorithm

### Two-Phase Protocol

#### **Phase 1 (Prepare)**
1. Proposer selects a unique proposal number
2. Sends `PrepareRequest` to majority of acceptors
3. Acceptor responds with:
   - Promise not to accept lower-numbered proposals
   - Previously accepted value (if any) with its proposal number

#### **Phase 2 (Accept)**
1. If proposer receives majority of promises:
   - Sends `AcceptRequest` with value
   - If acceptor previously accepted a value, proposer uses that
   - Otherwise, proposer's value is sent
2. Acceptor accepts if proposal number ≥ max promised number
3. Acceptor broadcasts to learners

#### **Learning**
- Learner counts accepts from acceptors
- When quorum (majority) of acceptors accept same value → consensus
- Instance is decided, value is immutable

### Multi-Paxos

The algorithm supports multiple independent instances:
- Each instance has separate proposal/accept/learn state
- Enables replicated state machine: sequence of decisions
- Instances can be proposed concurrently

## 🔒 Failure Handling

### Failures Tolerated
- Up to f acceptors can fail where f < N/2 (N = total acceptors)
- For 3 acceptors: tolerate 1 failure (quorum = 2)
- For 5 acceptors: tolerate 2 failures (quorum = 3)

### Behavior
- Proposal succeeds if majority accepts
- Failed acceptors can rejoin (lose state if not persisted)
- No split-brain (at most one value per instance)

### Missing Features for Production
- Persistent state storage (currently in-memory)
- State recovery after restart
- Distributed RPC between nodes (currently local only)
- Timeouts and retries
- View changes for proposer rotation

## 📝 Logging

All components use structured logging with configurable levels:

```
[INFO]  Server started id=1 port=8081
[DEBUG] Phase 1: Sending prepare requests proposal_number=1 instance=0
[DEBUG] Promise granted acceptor_id=0 proposal_number=1 instance=0
[DEBUG] Phase 2: Sending accept requests proposal_number=1 value=test_value instance=0
[INFO]  Consensus reached learner_id=0 value=test_value instance=0 acceptors=2
```

Enable debug logging with `-debug` flag.

## 🧪 Test Scenarios

### Simulated Delays
- 10ms delay between each prepare/accept message
- Tests verify proposals complete despite delays
- Validates timing logic

### Concurrent Operations
- Multiple proposals on different instances
- Concurrent acceptor operations
- Thread-safe state management

### Edge Cases
- Multiple proposals with same value
- Out-of-order message delivery (implicit in async tests)
- Majority requirement validation

## 📈 Performance Characteristics

- **Time Complexity**: O(n) where n = number of acceptors
- **Message Complexity**: O(n) per phase
- **State**: O(i) where i = number of instances
- **Thread Safety**: RWMutex per component

## 🔄 Migration from Old Implementation

The refactored version maintains algorithm compatibility while improving:

| Aspect | Old | New |
|--------|-----|-----|
| Server Files | 5 separate files | 1 unified binary |
| Package | `main` only | Structured `paxos` package |
| Logging | `fmt.Println` | Structured `slog` |
| Testing | None | 9 comprehensive tests |
| Concurrency | Implicit | Explicit with goroutines |
| Configuration | Hardcoded IPs | CLI flags |
| Multi-Paxos | Partial | Full support |

## 🛣️ Future Enhancements

- [ ] Persistent state with WAL
- [ ] Distributed RPC between nodes
- [ ] Timeout-based retries
- [ ] Leader election
- [ ] View change protocol
- [ ] Client session management
- [ ] Byzantine Paxos variant
- [ ] Performance benchmarks

## 📚 References

- [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [Multi-Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)#Multi-Paxos)
- [Go RPC Documentation](https://golang.org/pkg/net/rpc/)

## 📄 License

This project maintains its original structure and is available for educational purposes.

## 🤝 Contributing

Improvements and contributions welcome! Areas for enhancement:
- Persistent storage
- Distributed deployment
- Performance optimizations
- Additional test coverage

---

**Branch:** `refactor`  
**Last Updated:** 2026-04-20