# Bitcask Core Concepts

## What is Bitcask?

Bitcask is a log-structured hash table for fast key-value storage, originally created by Basho Technologies for Riak. It's designed for:
- **Low latency** reads and writes
- **High throughput** under load
- **Predictable performance** (no performance cliffs)
- **Crash recovery** with data integrity
- **Simple, understandable** codebase

## Fundamental Principles

### 1. Append-Only Log Structure
- All writes (inserts, updates, deletes) are appended to an active data file
- Never modify existing data in place
- Provides crash safety: partial writes are easily detected
- Enables sequential disk I/O (fast!)

### 2. In-Memory Hash Table (KeyDir)
- Maps every key to its location in the data files
- Structure: `key -> (file_id, value_position, value_size, timestamp)`
- Enables O(1) read operations
- Only metadata in memory, not values (handles datasets larger than RAM)

### 3. Immutable Historical Files
- When active file reaches size limit, it's closed and a new one created
- Old files are immutable (read-only)
- Multiple files numbered sequentially: `cask.0`, `cask.1`, `cask.2`, etc.
- Only the newest file accepts writes

### 4. Compaction (Merge Process)
- Reclaims space from deleted/updated keys
- Merges multiple old files into compacted versions
- Creates hint files for fast startup recovery
- Runs in background without blocking operations

## Key Data Structures

### Log Entry Format
```
+--------+------------+----------+------------+-----+-------+
|  CRC   | Timestamp  | Key Size | Value Size | Key | Value |
| 4 bytes|  8 bytes   | 4 bytes  |  4 bytes   | ... |  ...  |
+--------+------------+----------+------------+-----+-------+
```

**Fields:**
- **CRC-32**: Checksum for data integrity verification
- **Timestamp**: Unix timestamp (seconds or nanoseconds)
- **Key Size**: Length of the key in bytes
- **Value Size**: Length of the value in bytes (special value for tombstones)
- **Key**: The actual key bytes
- **Value**: The actual value bytes

### KeyDir Entry (In-Memory)
```
Key (string) -> {
    file_id: int          // Which file contains this key
    value_pos: int64      // Byte offset where value starts
    value_size: int       // Size of the value
    timestamp: int64      // When this was written
}
```

### Hint File Entry
Used during startup to rebuild KeyDir quickly without reading all values:
```
+------------+----------+------------+----------+-----+
| Timestamp  | Key Size | Value Size | Position | Key |
|  8 bytes   | 4 bytes  |  4 bytes   | 8 bytes  | ... |
+------------+----------+------------+----------+-----+
```

## Core Operations

### SET (Write)
1. Create log entry with key and value
2. Calculate CRC-32 checksum
3. Append entry to active data file
4. Update KeyDir with new location
5. If file exceeds threshold, create new active file

**Characteristics:**
- O(1) time complexity
- Sequential write (fast!)
- Immediate durability (can fsync)

### GET (Read)
1. Look up key in KeyDir (hash table lookup)
2. If not found, return "not found"
3. If found, seek to position in data file
4. Read value_size bytes
5. Return value

**Characteristics:**
- O(1) time complexity
- At most one disk seek
- Value not stored in memory

### DELETE
1. Write tombstone entry (special marker)
2. Use max uint32 value for value_size field
3. Update KeyDir to mark as deleted OR remove entry
4. Actual space reclaimed during merge

**Characteristics:**
- Same cost as write
- Logical delete (physical during merge)

### MERGE (Compaction)
1. Select immutable files for compaction
2. For each key, keep only the latest value
3. Write compacted data to new file(s)
4. Create hint files for each compacted file
5. Update KeyDir atomically
6. Delete old files

**Characteristics:**
- Runs in background
- Reclaims disk space
- Improves startup time

## Trade-offs and Design Decisions

### Advantages
1. **Predictable Performance**: No B-tree rebalancing or compaction pauses
2. **Fast Writes**: Sequential, append-only writes
3. **Fast Reads**: Single disk seek per operation
4. **Crash Recovery**: Easy to detect and recover from crashes
5. **Simple**: Easy to understand and debug

### Limitations
1. **Memory Requirement**: All keys must fit in RAM (KeyDir)
2. **Write Amplification**: Old data rewritten during merge
3. **Space Amplification**: Deleted data remains until merge
4. **Single Writer**: Only one process can write at a time

### When to Use Bitcask
- Keys fit in memory (or key count is reasonable)
- Workload is read-heavy or mixed
- Need predictable latency
- Want simple, reliable storage
- Values are relatively small to medium sized

### When NOT to Use Bitcask
- Billions of keys (memory constraint)
- Very large values (single seek becomes expensive)
- Range queries needed (Bitcask is hash table, no ordering)
- Multiple concurrent writers required

## Consistency and Durability

### Crash Recovery
1. Scan all data files in order
2. Rebuild KeyDir from log entries
3. Verify CRC for each entry
4. Skip corrupted entries at end of active file
5. Use hint files to speed up recovery

### Data Integrity
- CRC-32 checksum on every entry
- Corruption detected during reads
- Partial writes at end of file are safe to discard

### Concurrency Model
- Single writer (active file locked)
- Multiple readers (immutable files)
- Merge process operates on immutable files
- Atomic KeyDir updates
