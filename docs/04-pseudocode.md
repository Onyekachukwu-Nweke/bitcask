# Bitcask Implementation Pseudocode

This document provides detailed pseudocode for all major operations in Bitcask. Use this as a reference while implementing.

---

## Data Structures

```
// In-memory entry in KeyDir
struct KeyDirEntry {
    file_id: uint32          // Which file contains the value
    value_pos: uint64        // Byte offset in the file
    value_size: uint32       // Size of the value in bytes
    timestamp: uint64        // When this entry was written
}

// Main Bitcask structure
struct Bitcask {
    directory: string                    // Database directory path
    keydir: HashMap<string, KeyDirEntry> // In-memory index
    active_file: FileHandle              // Current write file
    active_file_id: uint32               // ID of active file
    active_file_size: uint64             // Current size of active file
    immutable_files: HashMap<uint32, FileHandle>  // Read-only files
    lock_file: FileHandle                // Process lock
    config: Config                       // Configuration options
    mutex: Lock                          // Thread safety
}

// Configuration
struct Config {
    max_file_size: uint64      // When to rotate (e.g., 1GB)
    sync_on_write: bool        // Fsync after writes
    max_key_size: uint32       // Maximum key size
    max_value_size: uint32     // Maximum value size
}

// Log entry on disk
struct LogEntry {
    crc: uint32           // CRC-32 checksum
    timestamp: uint64     // Unix timestamp
    key_size: uint32      // Length of key
    value_size: uint32    // Length of value (max for tombstone)
    key: []byte           // Key data
    value: []byte         // Value data
}

// Hint file entry
struct HintEntry {
    timestamp: uint64     // Unix timestamp
    key_size: uint32      // Length of key
    value_size: uint32    // Length of value
    value_pos: uint64     // Position in data file
    key: []byte           // Key data
}
```

---

## Core Operations

### Open Database

```
function Open(directory: string, config: Config) -> (Bitcask, error):
    // Create directory if it doesn't exist
    if not exists(directory):
        create_directory(directory)

    // Try to acquire lock
    lock_path = join(directory, "bitcask.lock")
    if exists(lock_path):
        // Check if lock is stale
        lock_pid = read_file(lock_path)
        if is_process_running(lock_pid):
            return error("database is already open")
        else:
            // Stale lock, remove it
            delete_file(lock_path)

    // Create lock file with current process ID
    lock_file = create_file(lock_path)
    write(lock_file, current_process_id())

    // Initialize Bitcask structure
    db = new Bitcask{
        directory: directory,
        keydir: new HashMap(),
        lock_file: lock_file,
        config: config,
        immutable_files: new HashMap(),
        mutex: new Lock()
    }

    // Load existing data
    error = db.load_from_disk()
    if error:
        return error

    return db, nil


function load_from_disk(db: Bitcask) -> error:
    // Find all data files
    files = list_files_matching(db.directory, "cask.*")
    // Exclude hint files
    data_files = filter(files, f => not ends_with(f, ".hint"))

    // Sort by file ID (numeric part)
    sort(data_files, by_file_id)

    // Process each file
    for file in data_files:
        file_id = extract_file_id(file)

        // Try to load from hint file first
        hint_path = file + ".hint"
        if exists(hint_path):
            error = db.load_from_hint_file(file_id, hint_path)
            if error:
                // Fall back to data file
                error = db.load_from_data_file(file_id, file)
                if error:
                    return error
        else:
            // No hint file, load from data file
            error = db.load_from_data_file(file_id, file)
            if error:
                return error

        // Open file for reading
        file_handle = open_file(file, READ_ONLY)
        db.immutable_files[file_id] = file_handle

    // Determine active file ID
    if data_files is empty:
        db.active_file_id = 0
    else:
        db.active_file_id = max(file_ids) + 1

    // Open or create active file
    active_path = join(db.directory, "cask." + db.active_file_id)
    db.active_file = open_file(active_path, APPEND | CREATE)
    db.active_file_size = file_size(db.active_file)

    return nil


function load_from_data_file(db: Bitcask, file_id: uint32, path: string) -> error:
    file = open_file(path, READ_ONLY)
    offset = 0

    while not end_of_file(file):
        // Read entry header (20 bytes)
        header = read_bytes(file, 20)
        if len(header) < 20:
            // Incomplete header, likely end of file
            break

        crc = decode_uint32(header[0:4])
        timestamp = decode_uint64(header[4:12])
        key_size = decode_uint32(header[12:16])
        value_size = decode_uint32(header[16:20])

        // Read key and value
        key = read_bytes(file, key_size)
        value = read_bytes(file, value_size)

        if len(key) != key_size or len(value) != value_size:
            // Incomplete entry, stop here
            break

        // Validate CRC
        entry_data = header[4:20] + key + value
        calculated_crc = calculate_crc32(entry_data)
        if calculated_crc != crc:
            log_warning("corrupted entry at offset", offset)
            break

        // Update KeyDir
        if value_size == MAX_UINT32:
            // Tombstone - remove from KeyDir
            db.keydir.remove(key)
        else:
            value_pos = offset + 20 + key_size
            db.keydir[key] = KeyDirEntry{
                file_id: file_id,
                value_pos: value_pos,
                value_size: value_size,
                timestamp: timestamp
            }

        // Move to next entry
        entry_size = 20 + key_size + value_size
        offset += entry_size

    close(file)
    return nil


function load_from_hint_file(db: Bitcask, file_id: uint32, path: string) -> error:
    file = open_file(path, READ_ONLY)

    while not end_of_file(file):
        // Read hint entry header (24 bytes)
        header = read_bytes(file, 24)
        if len(header) < 24:
            break

        timestamp = decode_uint64(header[0:8])
        key_size = decode_uint32(header[8:12])
        value_size = decode_uint32(header[12:16])
        value_pos = decode_uint64(header[16:24])

        // Read key
        key = read_bytes(file, key_size)
        if len(key) != key_size:
            break

        // Update KeyDir (skip tombstones)
        if value_size != MAX_UINT32:
            db.keydir[key] = KeyDirEntry{
                file_id: file_id,
                value_pos: value_pos,
                value_size: value_size,
                timestamp: timestamp
            }

    close(file)
    return nil
```

---

### Set Operation

```
function Set(db: Bitcask, key: string, value: []byte) -> error:
    // Validate inputs
    if len(key) == 0:
        return error("key cannot be empty")
    if len(key) > db.config.max_key_size:
        return error("key too large")
    if len(value) > db.config.max_value_size:
        return error("value too large")

    // Lock for thread safety
    lock(db.mutex)
    defer unlock(db.mutex)

    // Create log entry
    timestamp = current_unix_timestamp()
    entry = create_log_entry(key, value, timestamp)

    // Get current position in active file
    value_pos = db.active_file_size + 20 + len(key)

    // Write to active file
    bytes_written = write(db.active_file, entry)
    if bytes_written != len(entry):
        return error("write failed")

    // Optionally sync to disk
    if db.config.sync_on_write:
        fsync(db.active_file)

    // Update KeyDir
    db.keydir[key] = KeyDirEntry{
        file_id: db.active_file_id,
        value_pos: value_pos,
        value_size: len(value),
        timestamp: timestamp
    }

    // Update active file size
    db.active_file_size += bytes_written

    // Check if file rotation needed
    if db.active_file_size >= db.config.max_file_size:
        error = db.rotate_active_file()
        if error:
            return error

    return nil


function create_log_entry(key: string, value: []byte, timestamp: uint64) -> []byte:
    key_bytes = to_bytes(key)
    key_size = len(key_bytes)
    value_size = len(value)

    // Build entry data (without CRC)
    buffer = new ByteBuffer()

    // Header (16 bytes)
    buffer.write_uint64(timestamp)
    buffer.write_uint32(key_size)
    buffer.write_uint32(value_size)

    // Key and value
    buffer.write_bytes(key_bytes)
    buffer.write_bytes(value)

    // Calculate CRC of everything except CRC field
    data = buffer.to_bytes()
    crc = calculate_crc32(data)

    // Create final entry with CRC at the beginning
    final_buffer = new ByteBuffer()
    final_buffer.write_uint32(crc)
    final_buffer.write_bytes(data)

    return final_buffer.to_bytes()


function rotate_active_file(db: Bitcask) -> error:
    // Close current active file
    close(db.active_file)

    // Move to immutable files
    db.immutable_files[db.active_file_id] = open_file(
        join(db.directory, "cask." + db.active_file_id),
        READ_ONLY
    )

    // Create new active file
    db.active_file_id++
    active_path = join(db.directory, "cask." + db.active_file_id)
    db.active_file = open_file(active_path, APPEND | CREATE)
    db.active_file_size = 0

    return nil
```

---

### Get Operation

```
function Get(db: Bitcask, key: string) -> ([]byte, error):
    // Lock for reading KeyDir
    lock_read(db.mutex)

    // Look up key in KeyDir
    entry = db.keydir[key]
    if entry is null:
        unlock_read(db.mutex)
        return null, error("key not found")

    // Copy entry data (release lock quickly)
    file_id = entry.file_id
    value_pos = entry.value_pos
    value_size = entry.value_size

    unlock_read(db.mutex)

    // Determine which file to read from
    if file_id == db.active_file_id:
        file = db.active_file
    else:
        file = db.immutable_files[file_id]
        if file is null:
            // File might have been merged
            file_path = join(db.directory, "cask." + file_id)
            file = open_file(file_path, READ_ONLY)

    // Seek to value position
    seek(file, value_pos)

    // Read value
    value = read_bytes(file, value_size)
    if len(value) != value_size:
        return null, error("read failed")

    return value, nil
```

---

### Delete Operation

```
function Delete(db: Bitcask, key: string) -> error:
    // Lock for thread safety
    lock(db.mutex)
    defer unlock(db.mutex)

    // Check if key exists (optional)
    if key not in db.keydir:
        return nil  // Idempotent delete

    // Create tombstone entry
    timestamp = current_unix_timestamp()
    entry = create_tombstone_entry(key, timestamp)

    // Write to active file
    bytes_written = write(db.active_file, entry)
    if bytes_written != len(entry):
        return error("write failed")

    // Optionally sync
    if db.config.sync_on_write:
        fsync(db.active_file)

    // Remove from KeyDir
    db.keydir.remove(key)

    // Update active file size
    db.active_file_size += bytes_written

    // Check if rotation needed
    if db.active_file_size >= db.config.max_file_size:
        error = db.rotate_active_file()
        if error:
            return error

    return nil


function create_tombstone_entry(key: string, timestamp: uint64) -> []byte:
    key_bytes = to_bytes(key)
    key_size = len(key_bytes)
    value_size = MAX_UINT32  // Special marker for tombstone

    buffer = new ByteBuffer()
    buffer.write_uint64(timestamp)
    buffer.write_uint32(key_size)
    buffer.write_uint32(value_size)
    buffer.write_bytes(key_bytes)
    // No value for tombstone

    data = buffer.to_bytes()
    crc = calculate_crc32(data)

    final_buffer = new ByteBuffer()
    final_buffer.write_uint32(crc)
    final_buffer.write_bytes(data)

    return final_buffer.to_bytes()
```

---

### Merge/Compaction Operation

```
function Merge(db: Bitcask) -> error:
    lock(db.mutex)

    // Get list of immutable files (not active)
    immutable_file_ids = []
    for file_id in db.immutable_files.keys():
        if file_id != db.active_file_id:
            immutable_file_ids.append(file_id)

    if len(immutable_file_ids) == 0:
        unlock(db.mutex)
        return nil  // Nothing to merge

    sort(immutable_file_ids)

    unlock(db.mutex)

    // Create temporary directory for merge
    merge_dir = join(db.directory, "merge.tmp")
    create_directory(merge_dir)

    // Track latest values for each key
    latest_entries = new HashMap<string, LogEntry>()

    // Read all entries from immutable files
    for file_id in immutable_file_ids:
        file_path = join(db.directory, "cask." + file_id)
        file = open_file(file_path, READ_ONLY)

        while not end_of_file(file):
            entry = read_log_entry(file)
            if entry is null:
                break

            // Skip if we have a newer version
            if entry.key in latest_entries:
                existing = latest_entries[entry.key]
                if existing.timestamp > entry.timestamp:
                    continue

            // Skip tombstones
            if entry.value_size == MAX_UINT32:
                latest_entries.remove(entry.key)
                continue

            latest_entries[entry.key] = entry

        close(file)

    // Write merged data to new files
    merged_file_id = min(immutable_file_ids)
    merged_path = join(merge_dir, "cask." + merged_file_id)
    hint_path = join(merge_dir, "cask." + merged_file_id + ".hint")

    merged_file = open_file(merged_path, APPEND | CREATE)
    hint_file = open_file(hint_path, APPEND | CREATE)

    offset = 0
    new_keydir_entries = new HashMap()

    // Sort keys for deterministic output
    keys = latest_entries.keys()
    sort(keys)

    for key in keys:
        entry = latest_entries[key]

        // Write to merged data file
        log_entry = create_log_entry(key, entry.value, entry.timestamp)
        write(merged_file, log_entry)

        // Write to hint file
        value_pos = offset + 20 + len(key)
        hint_entry = create_hint_entry(key, entry.value_size, value_pos, entry.timestamp)
        write(hint_file, hint_entry)

        // Track new KeyDir entry
        new_keydir_entries[key] = KeyDirEntry{
            file_id: merged_file_id,
            value_pos: value_pos,
            value_size: entry.value_size,
            timestamp: entry.timestamp
        }

        offset += len(log_entry)

    close(merged_file)
    close(hint_file)

    // Atomic replacement
    lock(db.mutex)

    // Update KeyDir
    for key, entry in new_keydir_entries:
        // Only update if we still have the old version
        if key in db.keydir:
            old_entry = db.keydir[key]
            if old_entry.file_id in immutable_file_ids:
                db.keydir[key] = entry

    // Move merged files to main directory
    final_merged_path = join(db.directory, "cask." + merged_file_id)
    final_hint_path = join(db.directory, "cask." + merged_file_id + ".hint")

    rename(merged_path, final_merged_path)
    rename(hint_path, final_hint_path)

    // Close and delete old immutable files
    for file_id in immutable_file_ids:
        if file_id in db.immutable_files:
            close(db.immutable_files[file_id])
            db.immutable_files.remove(file_id)

        old_file_path = join(db.directory, "cask." + file_id)
        old_hint_path = old_file_path + ".hint"

        if file_id != merged_file_id:
            delete_file(old_file_path)
            if exists(old_hint_path):
                delete_file(old_hint_path)

    // Open merged file as immutable
    db.immutable_files[merged_file_id] = open_file(final_merged_path, READ_ONLY)

    // Clean up temp directory
    delete_directory(merge_dir)

    unlock(db.mutex)

    return nil


function create_hint_entry(key: string, value_size: uint32,
                           value_pos: uint64, timestamp: uint64) -> []byte:
    key_bytes = to_bytes(key)
    key_size = len(key_bytes)

    buffer = new ByteBuffer()
    buffer.write_uint64(timestamp)
    buffer.write_uint32(key_size)
    buffer.write_uint32(value_size)
    buffer.write_uint64(value_pos)
    buffer.write_bytes(key_bytes)

    return buffer.to_bytes()
```

---

### Close Database

```
function Close(db: Bitcask) -> error:
    lock(db.mutex)
    defer unlock(db.mutex)

    // Close active file
    if db.active_file:
        fsync(db.active_file)
        close(db.active_file)

    // Close all immutable files
    for file_id, file in db.immutable_files:
        close(file)

    // Remove lock file
    close(db.lock_file)
    lock_path = join(db.directory, "bitcask.lock")
    delete_file(lock_path)

    // Clear KeyDir
    db.keydir.clear()

    return nil
```

---

## Helper Functions

```
function calculate_crc32(data: []byte) -> uint32:
    // Use standard CRC-32 algorithm (polynomial 0xEDB88320)
    crc = 0xFFFFFFFF

    for byte in data:
        crc = crc XOR byte
        for i in 0..7:
            if crc & 1:
                crc = (crc >> 1) XOR 0xEDB88320
            else:
                crc = crc >> 1

    return NOT crc


function read_log_entry(file: FileHandle) -> LogEntry:
    // Read header
    header = read_bytes(file, 20)
    if len(header) < 20:
        return null

    crc = decode_uint32(header[0:4])
    timestamp = decode_uint64(header[4:12])
    key_size = decode_uint32(header[12:16])
    value_size = decode_uint32(header[16:20])

    // Read key and value
    key = read_bytes(file, key_size)
    value = read_bytes(file, value_size)

    if len(key) != key_size or len(value) != value_size:
        return null

    // Validate CRC
    entry_data = header[4:20] + key + value
    if calculate_crc32(entry_data) != crc:
        return null

    return LogEntry{
        crc: crc,
        timestamp: timestamp,
        key_size: key_size,
        value_size: value_size,
        key: key,
        value: value
    }


function extract_file_id(filename: string) -> uint32:
    // Extract number from "cask.123" -> 123
    parts = split(filename, ".")
    return parse_uint32(parts[1])
```

---

## Error Handling Patterns

```
// Always validate inputs
function validate_key(key: string, max_size: uint32) -> error:
    if len(key) == 0:
        return error("key cannot be empty")
    if len(key) > max_size:
        return error("key exceeds maximum size")
    return nil


// Handle file operations safely
function safe_write(file: FileHandle, data: []byte) -> error:
    bytes_written = 0
    while bytes_written < len(data):
        n = write(file, data[bytes_written:])
        if n <= 0:
            return error("write failed")
        bytes_written += n
    return nil


// Atomic file operations
function atomic_rename(old_path: string, new_path: string) -> error:
    // On Unix, rename is atomic if on same filesystem
    error = rename(old_path, new_path)
    return error
```

---

## Concurrency Patterns

```
// Read-write lock for KeyDir
struct SafeKeyDir {
    data: HashMap<string, KeyDirEntry>
    mutex: RWLock
}

function get(kd: SafeKeyDir, key: string) -> KeyDirEntry:
    lock_read(kd.mutex)
    defer unlock_read(kd.mutex)
    return kd.data[key]

function set(kd: SafeKeyDir, key: string, entry: KeyDirEntry):
    lock_write(kd.mutex)
    defer unlock_write(kd.mutex)
    kd.data[key] = entry
```

This pseudocode provides a complete reference for implementing Bitcask. Adapt it to your chosen programming language and add error handling, logging, and optimizations as needed.
