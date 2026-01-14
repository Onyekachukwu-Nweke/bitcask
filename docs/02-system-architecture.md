# Bitcask System Architecture

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Bitcask Database                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Client API Layer                        │   │
│  │  (Set, Get, Delete, Merge, Close)                   │   │
│  └──────────────────┬──────────────────────────────────┘   │
│                     │                                         │
│  ┌──────────────────▼──────────────────────────────────┐   │
│  │           KeyDir (In-Memory Hash Table)             │   │
│  │                                                       │   │
│  │  Key1 -> {file: 0, pos: 0, size: 10, ts: 1234}     │   │
│  │  Key2 -> {file: 1, pos: 42, size: 20, ts: 1235}    │   │
│  │  Key3 -> {file: 2, pos: 100, size: 15, ts: 1236}   │   │
│  │  ...                                                 │   │
│  └──────────────────┬──────────────────────────────────┘   │
│                     │                                         │
│  ┌──────────────────▼──────────────────────────────────┐   │
│  │         File Manager & I/O Layer                     │   │
│  │                                                       │   │
│  │  - Active File (write)                              │   │
│  │  - Immutable Files (read-only)                      │   │
│  │  - File rotation                                     │   │
│  │  - CRC validation                                    │   │
│  └──────────────────┬──────────────────────────────────┘   │
│                     │                                         │
└─────────────────────┼─────────────────────────────────────┘
                      │
         ┌────────────▼────────────┐
         │      Disk Storage        │
         │                          │
         │  cask.0 (immutable)     │
         │  cask.0.hint            │
         │  cask.1 (immutable)     │
         │  cask.1.hint            │
         │  cask.2 (active)        │
         │  bitcask.lock           │
         └─────────────────────────┘
```

## Component Details

### 1. KeyDir (In-Memory Index)

**Purpose**: Fast lookup of key locations

**Structure**:
```
HashMap<String, KeyDirEntry>

KeyDirEntry {
    file_id: u32          // Which data file
    value_pos: u64        // Byte offset in file
    value_size: u32       // Size of value
    timestamp: u64        // Write timestamp
}
```

**Operations**:
- Insert/Update: O(1)
- Lookup: O(1)
- Delete: O(1)

**Lifecycle**:
- Built during startup by scanning data files
- Updated on every write operation
- Updated atomically during merge

### 2. Data File Structure

**File Naming Convention**:
```
cask.0          // First data file (oldest)
cask.1          // Second data file
cask.2          // Third data file
cask.3          // Active file (newest)
```

**File States**:
- **Active**: Currently accepting writes (exactly one)
- **Immutable**: Closed, read-only, candidate for merge

**Entry Layout** (binary format):
```
Offset   Size    Field
------   ----    -----
0        4       CRC-32 checksum
4        8       Timestamp (Unix time)
12       4       Key size (uint32)
16       4       Value size (uint32)
20       N       Key bytes
20+N     M       Value bytes
```

**File Rotation**:
- When active file exceeds max size (e.g., 1GB)
- Create new active file with incremented ID
- Previous active becomes immutable

### 3. Hint Files

**Purpose**: Fast startup recovery without reading all values

**File Naming**: `cask.N.hint` (corresponds to `cask.N`)

**Entry Layout**:
```
Offset   Size    Field
------   ----    -----
0        8       Timestamp
8        4       Key size
12       4       Value size
16       8       Value position in data file
24       N       Key bytes
```

**Usage**:
- Generated during merge operation
- Read during startup to rebuild KeyDir
- Much smaller than data files (no values)
- Speeds up recovery dramatically

### 4. Lock File

**File**: `bitcask.lock`

**Purpose**:
- Ensure single writer
- Prevent concurrent access corruption
- Contains process ID

**Behavior**:
- Created when database opens
- Deleted when database closes gracefully
- Stale lock detection on startup

## Data Flow Diagrams

### Write Operation Flow

```
SET(key, value)
      │
      ▼
┌─────────────────┐
│ Create Entry    │
│ - Timestamp     │
│ - Key bytes     │
│ - Value bytes   │
│ - Calculate CRC │
└────────┬────────┘
         │
         ▼
┌─────────────────┐         ┌──────────────┐
│ Write to Active │────────▶│ Sync to Disk │
│ File (append)   │         │ (optional)   │
└────────┬────────┘         └──────────────┘
         │
         ▼
┌─────────────────┐
│ Update KeyDir   │
│ (in-memory)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Check File Size │
│ Rotate if needed│
└─────────────────┘
```

### Read Operation Flow

```
GET(key)
   │
   ▼
┌─────────────────┐       ┌──────────────┐
│ Lookup in       │──NO──▶│ Return       │
│ KeyDir          │       │ NotFound     │
└────────┬────────┘       └──────────────┘
         │YES
         ▼
┌─────────────────┐
│ Get file_id,    │
│ value_pos,      │
│ value_size      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Seek to position│
│ in data file    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Read value_size │
│ bytes           │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Return value    │
└─────────────────┘
```

### Merge (Compaction) Flow

```
MERGE()
   │
   ▼
┌─────────────────────┐
│ Select immutable    │
│ files for merge     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ For each file:      │
│ Read all entries    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Keep only latest    │
│ version of each key │
│ (skip deleted)      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐       ┌──────────────────┐
│ Write to new        │──────▶│ Generate hint    │
│ merged file(s)      │       │ file(s)          │
└──────────┬──────────┘       └──────────────────┘
           │
           ▼
┌─────────────────────┐
│ Update KeyDir       │
│ with new locations  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Delete old files    │
│ atomically          │
└─────────────────────┘
```

### Startup/Recovery Flow

```
OPEN(directory)
   │
   ▼
┌─────────────────────┐       ┌──────────────────┐
│ Check for lock file │──YES─▶│ Validate/break   │
│                     │       │ stale lock       │
└──────────┬──────────┘       └──────────────────┘
           │NO
           ▼
┌─────────────────────┐
│ Create lock file    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Discover all data   │
│ files (cask.*)      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐       ┌──────────────────┐
│ For each file:      │       │ Read data file   │
│ Check for hint file │──NO──▶│ entries directly │
└──────────┬──────────┘       └────────┬─────────┘
           │YES                         │
           ▼                            │
┌─────────────────────┐                │
│ Read hint file      │                │
│ (faster)            │                │
└──────────┬──────────┘                │
           │◀───────────────────────────┘
           ▼
┌─────────────────────┐
│ Build KeyDir from   │
│ all entries         │
│ (latest wins)       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Open/create active  │
│ file for writes     │
└─────────────────────┘
```

## Directory Structure

```
/path/to/bitcask/
├── bitcask.lock              # Lock file (process ID)
├── cask.0                    # Oldest immutable data file
├── cask.0.hint              # Hint file for cask.0
├── cask.1                    # Immutable data file
├── cask.1.hint              # Hint file for cask.1
├── cask.2                    # Immutable data file
├── cask.2.hint              # Hint file for cask.2
└── cask.3                    # Active data file (current writes)
```

## Memory Layout

```
┌─────────────────────────────────────────┐
│         Process Memory Space             │
├─────────────────────────────────────────┤
│                                          │
│  KeyDir (HashMap)                       │
│  ├─ Hash buckets                        │
│  ├─ Key strings                         │
│  └─ Metadata structs (24 bytes each)   │
│                                          │
│  Open File Handles                      │
│  ├─ Active file (read/write)           │
│  └─ Immutable files (read-only)        │
│                                          │
│  Write Buffer (optional)                │
│  └─ Pending writes                      │
│                                          │
└─────────────────────────────────────────┘

Memory Usage Estimate:
- Per key: ~100 bytes (key + metadata + hash overhead)
- 1M keys: ~100 MB
- 10M keys: ~1 GB
- File handles: negligible
```

## Concurrency Model

```
┌──────────────────────────────────────────┐
│         Single Writer Process             │
├──────────────────────────────────────────┤
│                                           │
│  Write Thread                            │
│  └─ Exclusive access to active file     │
│                                           │
│  Read Threads (multiple)                 │
│  ├─ Shared read access to KeyDir        │
│  └─ Concurrent reads from data files    │
│                                           │
│  Merge Thread (background)               │
│  ├─ Operates on immutable files         │
│  └─ Atomic KeyDir updates                │
│                                           │
└──────────────────────────────────────────┘

Synchronization:
- KeyDir: Read-write lock or concurrent map
- Active file: Write lock (mutex)
- Immutable files: No locks needed
- Merge: Atomic file replacement
```

## Performance Characteristics

### Time Complexity
- **Write**: O(1) - append + hash table update
- **Read**: O(1) - hash table lookup + one disk seek
- **Delete**: O(1) - same as write
- **Merge**: O(N) - N = total entries in files being merged

### Space Complexity
- **Memory**: O(K) - K = number of unique keys
- **Disk**: O(N) - N = total data written (with amplification)

### I/O Patterns
- **Writes**: Sequential (fast, cache-friendly)
- **Reads**: Random (one seek per read)
- **Merge**: Sequential read + sequential write

### Latency Targets
- **Write**: < 1ms (without fsync), < 10ms (with fsync)
- **Read**: < 1ms (SSD), < 10ms (HDD)
- **Startup**: Depends on data size and hint files
