# IST Key-Value Store (IST-KVS)

## Overview

IST-KVS is a concurrent key-value storage system developed as part of the Operating Systems course (2024-25) at Instituto Superior Técnico. The system stores data as key-value pairs in a hashtable and supports parallel processing, backup operations, and client-server communication through named pipes.

## Features

- **Key-Value Storage**: Store and retrieve data using string keys and values (max 40 characters each)
- **Batch Processing**: Process command files (`.job`) from a specified directory
- **Concurrent Execution**: Multiple threads process different job files simultaneously
- **Non-blocking Backups**: Asynchronous backup operations using child processes
- **Client-Server Architecture**: Remote clients can subscribe to key changes and receive notifications
- **POSIX Interface**: File operations using POSIX file descriptors

## Project Structure

The project is divided into two parts:

### Part 1: Core Functionality
- **Exercise 1**: File system interaction and batch processing
- **Exercise 2**: Non-blocking backup mechanism using `fork()`
- **Exercise 3**: Parallelization with multiple threads

### Part 2: Client-Server Communication
- **Exercise 1**: Client interaction via named pipes (FIFOs)
- **Exercise 2**: Connection termination using signals (SIGUSR1)

## Commands

### Server Commands (in .job files)

- **WRITE**: Create or update key-value pairs
  ```
  WRITE [(key1,value1)(key2,value2)]
  ```

- **READ**: Retrieve values for specified keys
  ```
  READ [key1,key2]
  ```
  Returns `KVSERROR` for non-existent keys.

- **DELETE**: Remove key-value pairs
  ```
  DELETE [key1,key2]
  ```
  Returns `KVSMISSING` for non-existent keys.

- **SHOW**: Display all stored key-value pairs (alphabetically sorted)
  ```
  SHOW
  ```

- **WAIT**: Introduce execution delay (in milliseconds)
  ```
  WAIT 2000
  ```

- **BACKUP**: Create a backup of the current hashtable state
  ```
  BACKUP
  ```

- **HELP**: Display available commands and usage information
  ```
  HELP
  ```

### Client Commands

- **SUBSCRIBE**: Subscribe to a key for change notifications
  ```
  SUBSCRIBE key1
  ```

- **UNSUBSCRIBE**: Unsubscribe from a key
  ```
  UNSUBSCRIBE key1
  ```

- **DELAY**: Delay client execution (in seconds)
  ```
  DELAY 5
  ```

- **DISCONNECT**: Terminate the session
  ```
  DISCONNECT
  ```

## Usage

### Server

```bash
./kvs <jobs_directory> <max_concurrent_backups> <max_threads> <register_fifo_name>
```

**Parameters:**
- `jobs_directory`: Directory containing `.job` files
- `max_concurrent_backups`: Maximum number of simultaneous backup operations (Part 1: 3rd parameter)
- `max_threads`: Number of threads for parallel job processing (Part 1: 2nd parameter)
- `register_fifo_name`: Named pipe for client registration (Part 2 only)

**Part 1 Example:**
```bash
./kvs ./jobs 2 4
```

**Part 2 Example:**
```bash
./kvs ./jobs 4 2 register_fifo
```

### Client

```bash
./client/client <client_id> <register_fifo_name>
```

**Example:**
```bash
./client/client c1 register_fifo < test_client.txt
```

## Output Files

- **`.out` files**: Results of command execution for each `.job` file
- **`.bck` files**: Backup snapshots named as `<filename>-<backup_number>.bck`

**Example:**
- Input: `test.job`
- Outputs: `test.out`, `test-1.bck`, `test-2.bck`

## Client-Server Protocol

### Message Format

All messages follow a binary protocol with fixed-size fields:

- **Connect**: `OP_CODE(1) | req_pipe[40] | resp_pipe[40] | notif_pipe[40]`
- **Disconnect**: `OP_CODE(2)`
- **Subscribe**: `OP_CODE(3) | key[41]`
- **Unsubscribe**: `OP_CODE(4) | key[41]`
- **Notifications**: `key[41] | value[41]` (or `DELETED` for deleted keys)

Response messages include: `OP_CODE | result` (0 = success, 1 = error)

## Architecture

### Part 1 Architecture
- Main thread reads and distributes job files
- Worker threads process jobs concurrently
- Fork-based child processes handle backups
- Synchronization ensures atomic operations

### Part 2 Architecture
- **Host Thread**: Receives client connection requests via register FIFO
- **Manager Threads**: Handle client requests (S simultaneous sessions)
- **Job Processing Threads**: Execute commands from `.job` files
- **Producer-Consumer Buffer**: Coordinates connection requests
- **Named Pipes**: Three per client (requests, responses, notifications)

## Synchronization

- **Mutexes**: Protect hashtable access
- **Read-Write Locks**: Optimize concurrent read operations
- **Semaphores**: Implement producer-consumer buffer for client connections
- **Signals**: SIGUSR1 terminates all client connections

## Compilation

```bash
make
```

Clean build artifacts:
```bash
make clean
```

## Testing Environment

The project should compile and run correctly on the **Sigma cluster**. While development on other platforms (macOS, Windows/WSL) is allowed, official support is only provided for the Sigma environment.

## Important Notes

- Keys and values cannot contain spaces
- Maximum size: 40 characters for keys and values
- Comments in `.job` files start with `#`
- The string `KVSERROR` is reserved for read errors
- The string `KVSMISSING` is reserved for delete errors
- The string `DELETED` is reserved for deletion notifications
- All file operations must use POSIX file descriptors (not stdio.h)
- Client processes use two threads: main thread for commands, second thread for notifications

## Submission

- **Part 1 Deadline**: December 13, 2024, 23:59
- **Part 2 Deadline**: January 13, 2025, 23:59
- **Format**: ZIP file containing source code and Makefile
- **Platform**: Submit via Fénix
- **Requirements**: No binaries, `make clean` must remove all compiled files

## Academic Integrity

This project must be original work. Code sharing between groups or use of external sources will result in:
- Failure for all involved groups
- Report to LEIC coordination and IST Pedagogical Council

## Authors

Developed for the Operating Systems course at Instituto Superior Técnico, LEIC-A/LEIC-T/LETI programs.

---

For questions and support, refer to course materials and instructors.
