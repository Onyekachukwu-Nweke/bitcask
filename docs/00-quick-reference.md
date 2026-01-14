# Bitcask Quick Reference Guide

This document provides a quick overview of the entire Bitcask implementation. Use this as a starting point before diving into the detailed documentation.

## What You're Building

A **log-structured hash table** - a fast, reliable key-value store that:
- Appends all writes to a log (never modifies existing data)
- Keeps an in-memory index for O(1) reads
- Compacts old data in the background
- Recovers quickly from crashes

## Documentation Structure

Read in this order:

1. **[00-quick-reference.md](00-quick-reference.md)** (this file) - Overview and cheat sheet
2. **[01-bitcask-concepts.md](01-bitcask-concepts.md)** - Core concepts and theory
3. **[02-system-architecture.md](02-system-architecture.md)** - Detailed architecture diagrams
4. **[03-implementation-phases.md](03-implementation-phases.md)** - Step-by-step build guide
5. **[04-pseudocode.md](04-pseudocode.md)** - Complete pseudocode reference

## Key Concepts at a Glance

### Core Idea
```
Write: key=foo, value=bar
   └─> Append to log file
   └─> Update in-memory index: foo -> (file_id=1, pos=42, size=3)

Read: key=foo
   └─> Look up in index -> (file_id=1, pos=42, size=3)
   └─> Seek to position 42 in file 1
   └─> Read 3 bytes
   └─> Return "bar"
```

### Data Flow
```
                Write Path                         Read Path
                ----------                         ---------

User            User                              User
 │               │                                 │
 ├─ Set("k","v") │                    Get("k") ────┤
 │               │                                 │
 ▼               ▼                                 ▼
┌─────────────────────┐                    ┌──────────────┐
│  Create Log Entry   │                    │ Lookup in    │
│  ┌───────────────┐  │                    │ KeyDir       │
│  │ CRC           │  │                    │              │
│  │ Timestamp     │  │                    │ k -> {file:1,│
│  │ KeySize       │  │                    │      pos:42, │
│  │ ValueSize     │  │                    │      size:3} │
│  │ Key: "k"      │  │                    └──────┬───────┘
│  │ Value: "v"    │  │                           │
│  └───────────────┘  │                           ▼
└──────────┬──────────┘                    ┌──────────────┐
           │                                │ Open file 1  │
           ▼                                │ Seek to 42   │
    ┌─────────────┐                        │ Read 3 bytes │
    │ Append to   │                        └──────┬───────┘
    │ Active File │                               │
    └──────┬──────┘                               ▼
           │                                   Return "v"
           ▼
    ┌─────────────┐
    │ Update      │
    │ KeyDir      │
    └─────────────┘
```

## File Structure

```
your-database/
├── bitcask.lock          # Ensures single writer
├── cask.0                # Oldest immutable data file
├── cask.0.hint          # Fast recovery index for cask.0
├── cask.1                # Immutable data file
├── cask.1.hint          # Fast recovery index for cask.1
├── cask.2                # Immutable data file
├── cask.2.hint          # Fast recovery index for cask.2
└── cask.3                # Active file (accepting writes)
```

## Binary Format Cheat Sheet

### Log Entry (on disk)
```
+--------+------------+----------+------------+-----+-------+
|  CRC   | Timestamp  | Key Size | Value Size | Key | Value |
| 4 byte |   8 byte   |  4 byte  |   4 byte   | var |  var  |
+--------+------------+----------+------------+-----+-------+
     ↓          ↓           ↓           ↓         ↓      ↓
  uint32     uint64      uint32      uint32   []byte  []byte

Special: Value Size = 0xFFFFFFFF means DELETED (tombstone)
```

### Hint Entry (on disk)
```
+------------+----------+------------+----------+-----+
| Timestamp  | Key Size | Value Size | Position | Key |
|   8 byte   |  4 byte  |   4 byte   |  8 byte  | var |
+------------+----------+------------+----------+-----+
```

### KeyDir Entry (in memory)
```go
struct KeyDirEntry {
    file_id: uint32      // Which file? (e.g., 3 for cask.3)
    value_pos: uint64    // Byte offset in that file
    value_size: uint32   // How many bytes to read
    timestamp: uint64    // When was this written
}
```

## Operation Complexity

| Operation | Time | Space | Disk I/O |
|-----------|------|-------|----------|
| Set       | O(1) | O(1)  | 1 append |
| Get       | O(1) | O(1)  | 1 seek + read |
| Delete    | O(1) | O(1)  | 1 append (tombstone) |
| Merge     | O(N) | O(K)  | Read N, write M < N |

Where:
- N = total entries across all files
- K = number of unique keys
- M = number of live entries (after removing old versions)

## Implementation Checklist

### Phase 1: Basics (MVP)
- [ ] Binary encoding/decoding of log entries
- [ ] CRC-32 checksum calculation
- [ ] Open database directory
- [ ] Write entries to active file
- [ ] In-memory KeyDir (hash map)
- [ ] Read values from files
- [ ] Close database properly

### Phase 2: File Management
- [ ] File rotation when size limit reached
- [ ] Multiple immutable files
- [ ] Lock file (single writer)
- [ ] Delete operation (tombstones)

### Phase 3: Recovery
- [ ] Scan data files on startup
- [ ] Rebuild KeyDir from log
- [ ] Handle corrupted entries
- [ ] Detect and truncate incomplete writes

### Phase 4: Compaction
- [ ] Merge old files
- [ ] Remove obsolete entries
- [ ] Generate hint files
- [ ] Atomic file replacement

### Phase 5: Polish
- [ ] CLI tool
- [ ] Configuration options
- [ ] Error handling
- [ ] Tests and benchmarks

## Common Pitfalls

### 1. Endianness
```
❌ WRONG: Machine-dependent byte order
✅ RIGHT: Always use little-endian (or document your choice)
```

### 2. CRC Coverage
```
❌ WRONG: CRC covers only value
✅ RIGHT: CRC covers timestamp + sizes + key + value
```

### 3. KeyDir Updates
```
❌ WRONG: Update KeyDir before writing to disk
✅ RIGHT: Write to disk first, then update KeyDir
```

### 4. Merge Safety
```
❌ WRONG: Delete old files immediately
✅ RIGHT: Keep old files until merge is fully complete
```

### 5. Crash Recovery
```
❌ WRONG: Assume active file is complete
✅ RIGHT: Scan and validate, truncate at first error
```

## Testing Strategy

### Unit Tests
```
✓ Entry encoding/decoding
✓ CRC calculation
✓ KeyDir operations
✓ File operations
```

### Integration Tests
```
✓ Write then read
✓ Update existing key
✓ Delete then read
✓ Close and reopen
✓ Multiple file rotation
✓ Merge operation
```

### Chaos Tests
```
✓ Kill process during write
✓ Corrupt random bytes
✓ Fill disk
✓ Delete random files
```

## Performance Tips

### For Fast Writes
1. Batch writes together
2. Use buffered I/O
3. Only fsync when necessary
4. Large file sizes (fewer rotations)

### For Fast Reads
1. Keep KeyDir in memory (obvious, but critical)
2. Cache open file handles
3. Use mmap for immutable files (advanced)
4. Add bloom filters to skip missing keys (advanced)

### For Fast Recovery
1. Generate hint files during merge
2. Smaller data files = faster scan
3. Validate hint files with checksums
4. Parallel file scanning (advanced)

### For Less Disk Space
1. Merge frequently
2. Smaller file sizes (more merge opportunities)
3. Compress values (application level)
4. Expiration/TTL for old keys

## Debugging Commands

```bash
# View binary format of log entry
hexdump -C cask.0 | head -20

# Check file sizes
du -h cask.*

# Count entries in a file
# (divide file size by average entry size)
wc -c cask.0

# Verify CRC (custom tool you'll build)
bitcask verify --file cask.0

# View KeyDir state (custom tool)
bitcask dump-keydir

# Run merge manually
bitcask merge --verbose
```

## Golang-Specific Tips

### Essential Packages
```go
import (
    "encoding/binary"  // For encoding integers
    "hash/crc32"       // For checksums
    "os"               // For file operations
    "sync"             // For locks
    "path/filepath"    // For file paths
)
```

### Binary Encoding
```go
// Writing
buf := make([]byte, 4)
binary.LittleEndian.PutUint32(buf, value)
file.Write(buf)

// Reading
buf := make([]byte, 4)
file.Read(buf)
value := binary.LittleEndian.Uint32(buf)
```

### CRC Calculation
```go
data := []byte("hello")
crc := crc32.ChecksumIEEE(data)
```

### File Locking
```go
// Basic lock file (process ID)
lockFile := filepath.Join(dir, "bitcask.lock")
f, err := os.OpenFile(lockFile, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)
// If err != nil, database is already open
```

### Atomic File Operations
```go
// Write to temp file, then rename (atomic)
tmpPath := filepath.Join(dir, "cask.0.tmp")
finalPath := filepath.Join(dir, "cask.0")

file, _ := os.Create(tmpPath)
file.Write(data)
file.Close()

os.Rename(tmpPath, finalPath)  // Atomic on Unix
```

## Next Steps

1. **Read** [01-bitcask-concepts.md](01-bitcask-concepts.md) to understand the theory
2. **Study** [02-system-architecture.md](02-system-architecture.md) for the big picture
3. **Follow** [03-implementation-phases.md](03-implementation-phases.md) step by step
4. **Reference** [04-pseudocode.md](04-pseudocode.md) while coding

## Additional Resources

### Papers
- Original Bitcask paper (Basho Technologies)
- Log-Structured File Systems (Rosenblum & Ousterhout)

### Similar Systems
- LevelDB (Google) - More complex, uses LSM-trees
- RocksDB (Facebook) - Based on LevelDB
- Riak (Basho) - Uses Bitcask as storage backend
- Redis AOF - Append-only file, similar concept

### Go Learning
- Effective Go (golang.org)
- File I/O in Go
- Binary encoding in Go
- Concurrency patterns in Go

## Quick Test Plan

### Day 1: Basic I/O
```go
db.Set("hello", "world")
value := db.Get("hello")  // Should return "world"
```

### Day 2: Persistence
```go
db.Set("foo", "bar")
db.Close()
db = Open(dir)
value := db.Get("foo")  // Should still return "bar"
```

### Day 3: Updates
```go
db.Set("key", "v1")
db.Set("key", "v2")
db.Set("key", "v3")
value := db.Get("key")  // Should return "v3"
```

### Day 4: Deletes
```go
db.Set("temp", "data")
db.Delete("temp")
value := db.Get("temp")  // Should return error
```

### Day 5: Rotation
```go
for i := 0; i < 1000000; i++ {
    db.Set(fmt.Sprintf("key%d", i), largeValue)
}
// Should create multiple cask.* files
```

### Day 6: Merge
```go
// Create lots of updates
for i := 0; i < 100; i++ {
    db.Set("key", fmt.Sprintf("v%d", i))
}
sizeBefore := dirSize()
db.Merge()
sizeAfter := dirSize()
// sizeAfter should be much smaller
```

## Success Criteria

You've successfully implemented Bitcask when:

✅ You can write and read key-value pairs
✅ Data persists after closing and reopening
✅ Updates overwrite old values correctly
✅ Deletes work as expected
✅ Files rotate when they get too large
✅ Merge reclaims space from old data
✅ Database recovers from crashes gracefully
✅ Tests pass consistently
✅ Performance is reasonable (thousands of ops/sec)
✅ You understand every line of code you wrote

---

**Remember**: Start simple, test frequently, and iterate. Don't try to build everything at once. Get the basics working first, then add features incrementally.

**Good luck and have fun building!** 🚀
