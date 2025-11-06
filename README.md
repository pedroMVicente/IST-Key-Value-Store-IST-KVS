# IST Key-Value Store (IST-KVS)

## Project Description

IST-KVS is a high-performance, concurrent key-value storage system that demonstrates advanced operating systems concepts including multithreading, inter-process communication, and synchronization mechanisms. Built for the Operating Systems course at Instituto Superior Técnico (2024-25), this system showcases how modern storage solutions handle concurrent operations, maintain data consistency, and provide real-time updates to connected clients.

The system implements a hashtable-based storage engine where data is organized as key-value pairs. It supports batch processing through job files, performs non-blocking backups using child processes, and enables remote clients to subscribe to key changes through named pipes (FIFOs). The architecture emphasizes scalability and parallelism while ensuring atomic operations and thread-safe access to shared data structures.

### Key Features

- **Concurrent Processing**: Multi-threaded architecture processes multiple job files simultaneously
- **Non-Blocking Backups**: Fork-based child processes create snapshots without halting operations
- **Client-Server Model**: Remote clients connect via named pipes to subscribe to key updates
- **Real-Time Notifications**: Clients receive immediate notifications when subscribed keys change
- **POSIX Compliance**: File operations using POSIX file descriptors
- **Scalable Synchronization**: Optimized locking mechanisms maximize parallelism

## Architecture

### Storage Layer
- Hashtable data structure with collision handling
- Thread-safe operations using mutexes and read-write locks
- Atomic execution of all operations (WRITE, READ, DELETE)

### Processing Layer
- **Job Processing Threads**: Execute commands from `.job` files in parallel
- **Backup Processes**: Child processes handle asynchronous snapshots
- **Concurrent Backup Control**: Configurable limit on simultaneous backups

### Communication Layer (Part 2)
- **Host Thread**: Accepts client connection requests via registration FIFO
- **Manager Threads**: Dedicated threads handle each client session
- **Producer-Consumer Buffer**: Coordinates connection requests using semaphores
- **Named Pipes**: Three per client for requests, responses, and notifications
- **Signal Handling**: SIGUSR1 triggers graceful disconnection of all clients

## Commands

### Server Commands (in .job files)

```bash
WRITE [(key1,value1)(key2,value2)]    # Create or update key-value pairs
READ [key1,key2]                       # Retrieve values (returns KVSERROR if not found)
DELETE [key1,key2]                     # Remove pairs (returns KVSMISSING if not found)
SHOW                                    # Display all pairs (alphabetically sorted)
WAIT <milliseconds>                     # Introduce execution delay
BACKUP                                  # Create hashtable snapshot
HELP                                    # Display command information
```

Comments start with `#` and are ignored.

### Client Commands

```bash
SUBSCRIBE <key>      # Subscribe to key change notifications
UNSUBSCRIBE <key>    # Unsubscribe from key
DELAY <seconds>      # Delay execution
DISCONNECT           # Terminate session
```

## Usage

### Compilation

```bash
make            # Build the project
make clean      # Remove build artifacts
```

### Running the Server

**Part 1** (File processing only):
```bash
./kvs <jobs_directory> <max_concurrent_backups> <max_threads>
```

Example:
```bash
./kvs ./jobs 2 4
```

**Part 2** (With client-server support):
```bash
./kvs <jobs_directory> <max_threads> <max_concurrent_backups> <register_fifo_name>
```

Example:
```bash
./kvs ./jobs 4 2 register_fifo
```

### Running a Client (Part 2)

```bash
./client/client <client_id> <register_fifo_name>
```

Example with input file:
```bash
./client/client c1 register_fifo < test_client.txt
```

Interactive mode:
```bash
./client/client c1 register_fifo
SUBSCRIBE mykey
DELAY 5
UNSUBSCRIBE mykey
DISCONNECT
```

## Input/Output

### Input Files
- **`.job` files**: Contain sequences of server commands
- **Client input**: Commands via stdin or input redirection

### Output Files
- **`.out` files**: Results of executing each `.job` file
- **`.bck` files**: Backup snapshots named `<filename>-<backup_number>.bck`

### Example

Input file `test.job`:
```bash
# Create keys
WRITE [(user1,alice)(user2,bob)]
# Read keys
READ [user1,user2]
# Create backup
BACKUP
# Display all
SHOW
```

Output file `test.out`:
```bash
[(user1,alice)(user2,bob)]
(user1,alice)
(user2,bob)
```

Backup file `test-1.bck`:
```bash
(user1,alice)
(user2,bob)
```

## Client-Server Protocol

### Message Format

All messages use binary protocol with fixed-size fields:

| Operation | Request Format | Response Format |
|-----------|---------------|-----------------|
| Connect | `OP_CODE(1)` \| `req_pipe[40]` \| `resp_pipe[40]` \| `notif_pipe[40]` | `OP_CODE(1)` \| `result` |
| Disconnect | `OP_CODE(2)` | `OP_CODE(2)` \| `result` |
| Subscribe | `OP_CODE(3)` \| `key[41]` | `OP_CODE(3)` \| `result` |
| Unsubscribe | `OP_CODE(4)` \| `key[41]` | `OP_CODE(4)` \| `result` |

**Notifications**: `key[41]` \| `value[41]` (or `key[41]` \| `DELETED[41]`)

**Result codes**: `0` = success, `1` = error

Strings are fixed-size (40 chars) with null padding. Keys include terminating null character (41 bytes total).

## Synchronization Mechanisms

- **Mutexes**: Protect critical sections in hashtable operations
- **Read-Write Locks**: Allow multiple concurrent reads, exclusive writes
- **Semaphores**: Implement producer-consumer pattern for client connections
- **Signals**: SIGUSR1 handled only by host thread to disconnect all clients
- **Thread Masks**: Worker threads block SIGUSR1 using `pthread_sigmask()`

## Technical Constraints

- Maximum key/value length: 40 characters
- Keys and values cannot contain spaces
- Reserved strings: `KVSERROR`, `KVSMISSING`, `DELETED`
- File operations must use POSIX API (no `stdio.h` FILE streams)
- Maximum simultaneous sessions: S (defined in server code)

## Project Structure

```
.
├── server/
│   ├── kvs.c              # Main server implementation
│   ├── operations.c       # Hashtable operations
│   └── ...
├── client/
│   ├── client.c           # Client implementation
│   ├── api.c              # Client API functions
│   └── ...
├── jobs/
│   ├── test.job           # Example job file
│   └── ...
├── Makefile
└── README.md
```

## Development Environment

**Recommended**: Sigma cluster (officially supported)

**Also compatible**: macOS, Linux, Windows/WSL (community support only)

The project must compile and run correctly on the Sigma cluster for evaluation purposes.

## Testing

### Part 1 Testing
```bash
# Create sample job file
echo "WRITE [(key1,value1)]" > jobs/test.job
echo "READ [key1]" >> jobs/test.job
echo "SHOW" >> jobs/test.job

# Run server
./kvs ./jobs 2 4

# Check output
cat jobs/test.out
```

### Part 2 Testing
```bash
# Terminal 1: Start server
./kvs ./jobs 4 2 register_fifo

# Terminal 2: Run client
./client/client c1 register_fifo
SUBSCRIBE key1
# (Keep running to receive notifications)

# Terminal 3: Trigger writes
echo "WRITE [(key1,newvalue)]" > jobs/update.job
# Client in Terminal 2 will receive notification
```

## Implementation Highlights

### Part 1: Core System
- **Exercise 1**: POSIX file I/O, directory traversal, batch processing
- **Exercise 2**: Process forking, non-blocking backups, concurrent backup limiting
- **Exercise 3**: Thread pool implementation, fine-grained locking strategies

### Part 2: Client-Server
- **Exercise 1**: Named pipe creation/management, protocol implementation, subscription system, multi-threaded client handling
- **Exercise 2**: Signal handling, graceful disconnection, thread-safe signal masking

## Learning Outcomes

This project provides hands-on experience with:
- Multi-threaded programming and synchronization primitives
- Process creation and inter-process communication (IPC)
- Named pipes (FIFOs) for client-server communication
- Signal handling and thread-signal interactions
- POSIX file system API
- Producer-consumer patterns with semaphores
- Scalable locking strategies for concurrent data structures
- Non-blocking I/O operations

## Authors

Developed for Operating Systems (Sistemas Operativos) course, 2024-25  
Instituto Superior Técnico, Universidade de Lisboa  
LEIC-A / LEIC-T / LETI programs

## License

This project is for educational purposes as part of the Operating Systems course at IST.

---

**Note**: This implementation follows academic integrity guidelines. All code is original work developed for learning purposes.
