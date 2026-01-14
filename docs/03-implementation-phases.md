# Bitcask Implementation Phases

This document outlines a step-by-step approach to implementing Bitcask from scratch. Each phase builds upon the previous one, allowing you to test incrementally.

## Phase 0: Project Setup and Data Structures

### Goals
- Set up project structure
- Define core data structures
- Implement binary encoding/decoding

### Tasks

#### 1. Project Structure
```
bitcask/
├── cmd/
│   └── bitcask/          # CLI application
│       └── main.go
├── pkg/
│   └── bitcask/          # Core library
│       ├── bitcask.go    # Main DB interface
│       ├── entry.go      # Log entry encoding/decoding
│       ├── keydir.go     # In-memory index
│       ├── datafile.go   # File management
│       └── merge.go      # Compaction logic
├── go.mod
└── README.md
```

#### 2. Define Core Structures

**Log Entry**:
- CRC-32 (4 bytes)
- Timestamp (8 bytes)
- Key size (4 bytes)
- Value size (4 bytes)
- Key (variable)
- Value (variable)

**KeyDir Entry**:
- File ID (uint32)
- Value position (uint64)
- Value size (uint32)
- Timestamp (uint64)

#### 3. Implement Binary Encoding

Functions needed:
- `EncodeEntry(key, value) -> []byte`
- `DecodeEntry([]byte) -> (key, value, timestamp, error)`
- `CalculateCRC(data) -> uint32`
- `ValidateCRC(entry) -> bool`

**Testing**:
- Write unit tests for encoding/decoding
- Test CRC calculation and validation
- Test with various key/value sizes
- Test error cases (corrupted data)

---

## Phase 1: Basic Write Operations (SET)

### Goals
- Open/create database directory
- Write entries to active file
- Maintain in-memory KeyDir
- Support basic SET operation

### Tasks

#### 1. Database Initialization
```go
Functions:
- Open(directory string) -> (*Bitcask, error)
- Close() -> error
- Initialize directory structure
- Create lock file
- Find or create active file
```

#### 2. Lock File Management
- Check for existing lock
- Detect stale locks
- Create lock with process ID
- Clean up on close

#### 3. File Management
- Open active file for append
- Track active file ID
- Track file size for rotation
- File naming: `cask.{id}`

#### 4. Write Implementation
```go
Function: Set(key string, value []byte) -> error

Steps:
1. Create log entry
2. Calculate CRC
3. Append to active file
4. Update KeyDir
5. Return success
```

#### 5. KeyDir Management
- Initialize empty hash map
- Insert/update entries on write
- Thread-safe access (if needed)

**Testing**:
- Open/close database
- Write single key-value pair
- Write multiple pairs
- Overwrite existing keys
- Verify KeyDir state
- Verify file contents manually

---

## Phase 2: Basic Read Operations (GET)

### Goals
- Look up keys in KeyDir
- Read values from data files
- Handle missing keys

### Tasks

#### 1. Read Implementation
```go
Function: Get(key string) -> ([]byte, error)

Steps:
1. Look up key in KeyDir
2. If not found, return error
3. Open data file by file_id
4. Seek to value_pos
5. Read value_size bytes
6. Return value
```

#### 2. File Handle Management
- Keep active file open
- Cache immutable file handles (optional)
- Close files properly

#### 3. Error Handling
- Key not found
- File read errors
- Corrupted data

**Testing**:
- Read existing keys
- Read non-existent keys
- Read after writes
- Read overwritten keys
- Verify correct values returned

---

## Phase 3: File Rotation

### Goals
- Detect when active file exceeds threshold
- Create new active file
- Mark old file as immutable

### Tasks

#### 1. Size Tracking
- Track bytes written to active file
- Define max file size constant (e.g., 10MB for testing)

#### 2. Rotation Logic
```go
Function: rotateActiveFile() -> error

Steps:
1. Close current active file
2. Increment file ID
3. Create new active file
4. Update active file reference
```

#### 3. Integration
- Check size after each write
- Rotate if threshold exceeded
- Ensure atomicity

**Testing**:
- Write enough data to trigger rotation
- Verify multiple files created
- Read from old and new files
- Check file numbering

---

## Phase 4: Delete Operations

### Goals
- Support logical deletion
- Write tombstone markers
- Update KeyDir

### Tasks

#### 1. Tombstone Format
- Use special value_size (e.g., max uint32)
- Empty value field
- Keep key and timestamp

#### 2. Delete Implementation
```go
Function: Delete(key string) -> error

Steps:
1. Check if key exists (optional)
2. Create tombstone entry
3. Write to active file
4. Remove from KeyDir or mark deleted
5. Return success
```

#### 3. Read Path Updates
- Treat deleted keys as not found

**Testing**:
- Delete existing keys
- Delete non-existent keys (idempotent)
- Read after delete
- Delete then re-insert same key

---

## Phase 5: Startup and Recovery

### Goals
- Rebuild KeyDir from data files
- Handle corrupted entries
- Fast startup with hint files (later)

### Tasks

#### 1. Scan Data Files
```go
Function: loadFromDataFiles() -> error

Steps:
1. List all cask.* files in directory
2. Sort by file ID (oldest first)
3. For each file:
   a. Open file
   b. Read entries sequentially
   c. Validate CRC
   d. Update KeyDir (latest wins)
4. Identify active file (highest ID)
```

#### 2. Handle Corruption
- Skip entries with bad CRC
- Truncate active file at first error
- Log warnings for corrupted data

#### 3. Active File Detection
- Highest numbered file becomes active
- Open for append mode
- Track current size

**Testing**:
- Close and reopen database
- Verify KeyDir rebuilt correctly
- Test with multiple files
- Introduce corrupted entry (manual test)
- Verify recovery from crash (kill process)

---

## Phase 6: Merge/Compaction

### Goals
- Reclaim space from deleted/updated keys
- Merge multiple immutable files
- Generate hint files

### Tasks

#### 1. Merge Planning
```go
Function: Merge() -> error

Steps:
1. Identify immutable files (not active)
2. Select files to merge (e.g., all, or subset)
3. Create temporary merge directory
```

#### 2. Merge Process
```
For selected files:
1. Read all entries
2. For each key, keep only latest version
3. Skip tombstones
4. Write to new compacted file
5. Generate hint file
```

#### 3. Atomic Replacement
```
Steps:
1. Write merged files with temp names
2. Update KeyDir with new locations
3. Rename temp files to final names
4. Delete old files
5. Update in-memory state
```

#### 4. Hint File Generation
```
For each entry written during merge:
1. Write hint entry (no value, just metadata)
2. Include: timestamp, key_size, value_size, position, key
```

**Testing**:
- Write many updates to same keys
- Run merge
- Verify disk space reduced
- Verify reads still work
- Check hint files created

---

## Phase 7: Hint Files for Fast Startup

### Goals
- Speed up recovery using hint files
- Fall back to data files if hints missing

### Tasks

#### 1. Hint File Reading
```go
Function: loadFromHintFile(fileID) -> error

Steps:
1. Open cask.{id}.hint
2. Read hint entries sequentially
3. Update KeyDir with metadata
4. No need to read actual values
```

#### 2. Startup Optimization
```
For each data file:
1. Check if hint file exists
2. If yes: load from hint file (fast)
3. If no: load from data file (slower)
4. Handle active file (no hint file)
```

#### 3. Hint File Format
- Same as data file but without values
- Include value position in data file
- Smaller and faster to read

**Testing**:
- Merge to create hint files
- Close and reopen database
- Measure startup time improvement
- Test with missing hint files
- Test with corrupted hint files

---

## Phase 8: CLI Tool

### Goals
- User-friendly command-line interface
- Support all operations
- Interactive mode (optional)

### Tasks

#### 1. Commands
```bash
bitcask set <key> <value>
bitcask get <key>
bitcask delete <key>
bitcask merge
bitcask list (optional)
bitcask stats (optional)
```

#### 2. CLI Structure
```go
main.go:
- Parse command-line arguments
- Open database
- Execute command
- Display result
- Close database
```

#### 3. Enhanced Features (Optional)
- Interactive REPL mode
- Batch operations from file
- Export/import data
- Database statistics

**Testing**:
- Manual testing of all commands
- Script automated tests
- Test error messages
- Test with large datasets

---

## Phase 9: Optimizations and Polish

### Goals
- Improve performance
- Add configuration options
- Better error handling and logging

### Tasks

#### 1. Write Optimizations
- Buffered writes
- Batch writes
- Fsync options (durability vs performance)

#### 2. Read Optimizations
- File handle caching
- Read-ahead buffering
- Bloom filters (advanced)

#### 3. Configuration
```go
Config struct:
- MaxFileSize
- SyncWrites (fsync after each write)
- MaxKeySize
- MaxValueSize
```

#### 4. Concurrency
- Read-write locks for KeyDir
- Multiple readers
- Single writer enforcement

#### 5. Monitoring
- Metrics (reads, writes, merges)
- Logging levels
- Statistics API

**Testing**:
- Benchmark read/write performance
- Stress test with concurrent operations
- Memory profiling
- Disk usage analysis

---

## Phase 10: Advanced Features (Optional)

### Goals
- Production-ready features
- Advanced optimizations

### Ideas

#### 1. Compaction Strategies
- Automatic background merge
- Configurable merge triggers
- Partial merges (not all files)

#### 2. Expiration (TTL)
- Time-to-live for keys
- Automatic cleanup
- Extended entry format

#### 3. Iterators
- Iterate all keys
- Stream large datasets
- Snapshot isolation

#### 4. Replication
- Write-ahead log streaming
- Follower nodes
- Consistency guarantees

#### 5. Transactions
- Batch operations
- Atomic multi-key updates
- Rollback support

---

## Testing Strategy

### Unit Tests
- Each component in isolation
- Entry encoding/decoding
- KeyDir operations
- File operations

### Integration Tests
- Full database lifecycle
- Write + read + delete cycles
- Crash recovery
- Merge operations

### Performance Tests
- Throughput benchmarks
- Latency percentiles
- Memory usage
- Disk space efficiency

### Chaos Tests
- Kill process during writes
- Corrupt data files
- Delete files manually
- Fill disk

---

## Milestones

### Milestone 1: Basic KV Store
- Phases 0-2 complete
- Can write and read data
- Single file only

### Milestone 2: Full Lifecycle
- Phases 3-5 complete
- File rotation works
- Deletion supported
- Recovery functional

### Milestone 3: Production Ready
- Phases 6-9 complete
- Merge/compaction works
- Fast startup with hints
- CLI tool available
- Optimized and tested

### Milestone 4: Advanced (Optional)
- Phase 10 features
- Battle-tested
- Well-documented
- Community-ready

---

## Common Pitfalls to Avoid

1. **Not testing recovery**: Always test crash scenarios
2. **Forgetting CRC validation**: Always verify data integrity
3. **Race conditions**: Careful with concurrent access
4. **Memory leaks**: Close files properly
5. **Not handling corruption**: Partial writes will happen
6. **Ignoring performance**: Profile early and often
7. **Complex before simple**: Get basics working first
8. **Poor error handling**: Don't panic, return errors
9. **No backups during merge**: Keep old files until success
10. **Blocking on merge**: Run compaction in background

---

## Recommended Reading

Before each phase, review:
- The original Bitcask paper
- Go's `os` and `io` packages documentation
- Binary encoding in Go (`encoding/binary`)
- Hash tables and CRC algorithms
- File systems and fsync semantics

## Next Steps

1. Start with Phase 0
2. Write tests for each component
3. Verify each phase before moving on
4. Keep the implementation simple
5. Optimize only after measuring
6. Document as you go
7. Have fun learning!
