# Bitcask Implementation Guide

Welcome to your comprehensive guide for building a Bitcask key-value store from scratch! This documentation will help you understand the concepts, architecture, and implementation details needed to build your own Bitcask database in Go.

## What is Bitcask?

Bitcask is a log-structured hash table for fast, reliable key-value storage. Originally created by Basho Technologies for Riak, it's designed to provide:

- **Predictable performance** - O(1) reads and writes
- **High throughput** - Sequential disk I/O
- **Crash recovery** - Data integrity guarantees
- **Simplicity** - Understandable codebase

## Documentation Overview

This guide is organized into five comprehensive documents:

### 📚 [Quick Reference](00-quick-reference.md)
**Start here!** A high-level overview and cheat sheet covering:
- Key concepts at a glance
- Binary format reference
- Implementation checklist
- Common pitfalls
- Testing strategy
- Quick debugging tips

**Best for**: Getting oriented, quick lookups during implementation

---

### 🧠 [Core Concepts](01-bitcask-concepts.md)
Deep dive into the fundamental principles:
- What makes Bitcask unique
- Log-structured storage
- KeyDir (in-memory index)
- Data structures and formats
- Core operations (Set, Get, Delete, Merge)
- Trade-offs and design decisions
- When to use (and not use) Bitcask

**Best for**: Understanding the "why" behind the design

---

### 🏗️ [System Architecture](02-system-architecture.md)
Detailed architectural diagrams and component breakdown:
- High-level system architecture
- Component details (KeyDir, Files, Hints)
- Data flow diagrams for all operations
- Directory and file structure
- Memory layout and concurrency model
- Performance characteristics

**Best for**: Visualizing how everything fits together

---

### 📋 [Implementation Phases](03-implementation-phases.md)
Step-by-step build guide with 10 phases:
- **Phase 0**: Project setup and data structures
- **Phase 1**: Basic write operations
- **Phase 2**: Read operations
- **Phase 3**: File rotation
- **Phase 4**: Delete operations
- **Phase 5**: Startup and recovery
- **Phase 6**: Merge/compaction
- **Phase 7**: Hint files for fast startup
- **Phase 8**: CLI tool
- **Phase 9**: Optimizations
- **Phase 10**: Advanced features

Each phase includes goals, tasks, and testing strategies.

**Best for**: Following a structured implementation path

---

### 💻 [Pseudocode Reference](04-pseudocode.md)
Complete pseudocode for all operations:
- Data structures
- Open/Close database
- Set, Get, Delete operations
- File rotation logic
- Merge/compaction algorithm
- Recovery procedures
- Helper functions
- Concurrency patterns

**Best for**: Translating concepts to actual code

---

## Learning Path

### For Beginners
1. Read [Quick Reference](00-quick-reference.md) to get oriented
2. Study [Core Concepts](01-bitcask-concepts.md) to understand fundamentals
3. Review [System Architecture](02-system-architecture.md) to see the big picture
4. Follow [Implementation Phases](03-implementation-phases.md) step-by-step
5. Reference [Pseudocode](04-pseudocode.md) while coding

### For Experienced Developers
1. Skim [Quick Reference](00-quick-reference.md)
2. Read [Core Concepts](01-bitcask-concepts.md) for trade-offs
3. Jump to [Pseudocode](04-pseudocode.md) and start implementing
4. Refer back to [Implementation Phases](03-implementation-phases.md) as needed

### For Code Review
1. Use [System Architecture](02-system-architecture.md) to verify structure
2. Check [Pseudocode](04-pseudocode.md) for algorithmic correctness
3. Validate against [Core Concepts](01-bitcask-concepts.md) principles

## Key Concepts Summary

### The Big Idea
```
┌─────────────────────────────────────────────┐
│  Append-Only Log + In-Memory Index = Fast!  │
└─────────────────────────────────────────────┘

Writes: Append to log (sequential, fast)
Reads:  Index lookup + one disk seek (O(1))
Compaction: Background process (non-blocking)
```

### Core Components

1. **KeyDir** (In-Memory Hash Table)
   - Maps keys to file locations
   - O(1) lookup time
   - Only metadata in RAM, not values

2. **Data Files** (Append-Only Logs)
   - Active file: accepts writes
   - Immutable files: read-only
   - Binary format with CRC checksums

3. **Hint Files** (Recovery Optimization)
   - Speed up startup
   - Contain only keys + metadata
   - Generated during merge

4. **Merge Process** (Compaction)
   - Reclaims space
   - Runs in background
   - Produces compacted files + hints

## Quick Start

### Prerequisites
- Go 1.21 or later
- Basic understanding of file I/O
- Familiarity with hash tables
- Knowledge of binary encoding

### Initial Setup
```bash
# Create project structure
mkdir -p bitcask/cmd/bitcask
mkdir -p bitcask/pkg/bitcask
cd bitcask
go mod init github.com/yourusername/bitcask

# Create initial files (as you go)
```

### First Test
```go
// Your first working test should be:
db := Open("/tmp/testdb")
db.Set("hello", []byte("world"))
value := db.Get("hello")
// value should be []byte("world")
db.Close()
```

## Testing Strategy

### Test-Driven Development
For each phase:
1. Write tests first
2. Implement to pass tests
3. Refactor if needed
4. Move to next phase

### Test Categories
- **Unit tests**: Individual functions
- **Integration tests**: Full operations
- **Chaos tests**: Failure scenarios
- **Benchmarks**: Performance validation

## Success Metrics

Your implementation is successful when:
- ✅ All operations work correctly
- ✅ Data persists across restarts
- ✅ Recovery handles corruption gracefully
- ✅ Merge reclaims disk space
- ✅ Performance meets expectations
- ✅ Code is clean and tested

## Common Questions

**Q: Do I need to implement everything?**
A: No! Phases 0-7 give you a fully functional Bitcask. Phases 8-10 are enhancements.

**Q: What if I get stuck?**
A: Refer to the pseudocode, read the concept docs again, or implement a simpler version first.

**Q: Can I optimize from the start?**
A: No! Get it working correctly first, then optimize based on measurements.

**Q: Should I use external libraries?**
A: Minimize them. Use standard library for learning, especially for core logic.

**Q: How long will this take?**
A: Depends on your experience. Basic implementation: 3-7 days. Polished version: 2-3 weeks.

## Additional Resources

### Papers & Articles
- [Bitcask Design Paper](https://riak.com/assets/bitcask-intro.pdf) - Original specification
- Log-Structured File Systems - Foundational concepts

### Related Projects
- Riak - Uses Bitcask as storage engine
- LevelDB - Similar ideas, more complex
- RocksDB - Production-grade LSM-tree storage

### Go-Specific Resources
- [Effective Go](https://golang.org/doc/effective_go.html)
- Go's `encoding/binary` package docs
- Go's `hash/crc32` package docs
- File I/O patterns in Go

## Project Structure Recommendation

```
bitcask/
├── cmd/
│   └── bitcask/              # CLI application
│       └── main.go
├── pkg/
│   └── bitcask/              # Core library
│       ├── bitcask.go        # Main DB interface
│       ├── bitcask_test.go   # Integration tests
│       ├── entry.go          # Log entry encoding
│       ├── entry_test.go     # Entry tests
│       ├── keydir.go         # In-memory index
│       ├── keydir_test.go    # KeyDir tests
│       ├── datafile.go       # File management
│       ├── datafile_test.go  # File tests
│       ├── merge.go          # Compaction
│       └── merge_test.go     # Merge tests
├── docs/                     # This documentation
│   ├── README.md            # This file
│   ├── 00-quick-reference.md
│   ├── 01-bitcask-concepts.md
│   ├── 02-system-architecture.md
│   ├── 03-implementation-phases.md
│   └── 04-pseudocode.md
├── go.mod
├── go.sum
└── README.md                 # Project README
```

## Tips for Success

### 1. Start Simple
Don't try to implement everything at once. Get basic read/write working first.

### 2. Test Constantly
Write tests for each new feature. Test both happy and error paths.

### 3. Handle Errors Properly
Don't use `panic()`. Return errors and handle them gracefully.

### 4. Think About Crashes
Your database will crash. Design for recovery from the start.

### 5. Measure Performance
Use benchmarks to validate your assumptions about performance.

### 6. Read the Docs
When stuck, read through the documentation again. Details matter.

### 7. Keep It Clean
Write readable code. Future you will thank present you.

### 8. Iterate
Your first implementation doesn't need to be perfect. Refactor as you learn.

## Getting Help

While implementing, you can:
1. Re-read the relevant documentation section
2. Check the pseudocode for implementation details
3. Review the architecture diagrams
4. Write tests to understand expected behavior
5. Start with a simpler version, then add complexity

## What You'll Learn

By building Bitcask, you'll gain deep understanding of:
- Log-structured storage systems
- Binary file formats and encoding
- File system operations and atomicity
- In-memory data structures for disk-based systems
- Data integrity and corruption handling
- Compaction and space reclamation
- Crash recovery mechanisms
- Performance optimization techniques
- Production-ready error handling

## Next Steps

**Ready to begin?**

1. Read [00-quick-reference.md](00-quick-reference.md) for a quick overview
2. Study [01-bitcask-concepts.md](01-bitcask-concepts.md) to understand the theory
3. Review [02-system-architecture.md](02-system-architecture.md) to see the structure
4. Start Phase 0 in [03-implementation-phases.md](03-implementation-phases.md)
5. Code with [04-pseudocode.md](04-pseudocode.md) as your reference

**Happy coding!** 🚀

---

*This documentation is designed to help you learn by building. Take your time, understand each concept, and enjoy the journey of creating a production-quality storage system from scratch.*
