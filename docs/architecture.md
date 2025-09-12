# bbolt System Architecture

bbolt is a pure Go embedded key/value database that provides ACID transactions with a B+tree storage engine. This document provides a high-level overview of the system architecture and how the major components interact.

## High-Level Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        App[User Application]
    end
    
    subgraph "bbolt API Layer"
        API[bbolt API]
        DB[DB Instance]
    end
    
    subgraph "Transaction Layer"
        TxMgr[Transaction Manager]
        ReadTx[Read-Only Transactions]
        WriteTx[Read-Write Transaction]
    end
    
    subgraph "Storage Layer"
        BucketMgr[Bucket Manager]
        Cursor[Cursors]
        BTree[B+Tree Engine]
    end
    
    subgraph "Memory Layer"
        NodeCache[Node Cache]
        PageCache[Page Cache]
        MMap[Memory Mapping]
    end
    
    subgraph "Disk Layer"
        MetaPages[Meta Pages]
        DataPages[Data Pages]
        FreeList[Free List]
        DBFile[(Database File)]
    end
    
    App --> API
    API --> DB
    DB --> TxMgr
    TxMgr --> ReadTx
    TxMgr --> WriteTx
    ReadTx --> BucketMgr
    WriteTx --> BucketMgr
    BucketMgr --> Cursor
    BucketMgr --> BTree
    Cursor --> BTree
    BTree --> NodeCache
    BTree --> PageCache
    NodeCache --> MMap
    PageCache --> MMap
    MMap --> MetaPages
    MMap --> DataPages
    MMap --> FreeList
    MetaPages --> DBFile
    DataPages --> DBFile
    FreeList --> DBFile
```

## Core Components Overview

bbolt's architecture is organized into several key layers:

### 1. Application Interface Layer
- **Purpose**: Provides the public API for database operations
- **Key Components**: `DB` struct, public methods
- **Responsibilities**: Database lifecycle, connection management, API surface

### 2. Transaction Management Layer
- **Purpose**: Handles ACID transactions and concurrency control
- **Key Components**: `Tx` struct, transaction manager
- **Responsibilities**: MVCC, isolation, consistency, transaction lifecycle

### 3. Storage Management Layer
- **Purpose**: Manages data organization and access patterns
- **Key Components**: `Bucket`, `Cursor`, B+tree operations
- **Responsibilities**: Data organization, indexing, traversal

### 4. Memory Management Layer
- **Purpose**: Optimizes memory usage and provides caching
- **Key Components**: `node`, page cache, memory mapping
- **Responsibilities**: Memory-mapped I/O, in-memory node management, caching

### 5. Disk Storage Layer
- **Purpose**: Persistent storage and file management
- **Key Components**: Database file, pages, freelist
- **Responsibilities**: Durability, space management, file I/O

## Key Architectural Principles

### 1. Single Writer, Multiple Readers (MVCC)
```mermaid
graph LR
    subgraph "Concurrency Model"
        Writer[Single Writer Transaction]
        Reader1[Reader Transaction 1]
        Reader2[Reader Transaction 2]
        Reader3[Reader Transaction N]
        
        Writer --> Database[(Database)]
        Reader1 --> Database
        Reader2 --> Database
        Reader3 --> Database
        
        Writer -.-> Lock[Exclusive Write Lock]
        Reader1 -.-> Snapshot[Consistent Snapshot]
        Reader2 -.-> Snapshot
        Reader3 -.-> Snapshot
    end
```

### 2. Copy-on-Write B+Tree
```mermaid
graph TD
    subgraph "B+Tree Structure"
        Root[Root Node]
        Internal1[Internal Node]
        Internal2[Internal Node]
        Leaf1[Leaf Node]
        Leaf2[Leaf Node]
        Leaf3[Leaf Node]
        Leaf4[Leaf Node]
        
        Root --> Internal1
        Root --> Internal2
        Internal1 --> Leaf1
        Internal1 --> Leaf2
        Internal2 --> Leaf3
        Internal2 --> Leaf4
        
        Leaf1 -.-> Leaf2
        Leaf2 -.-> Leaf3
        Leaf3 -.-> Leaf4
    end
    
    subgraph "Copy-on-Write Behavior"
        OrigPage[Original Page]
        NewPage[New Page Copy]
        
        OrigPage -.-> NewPage
        NewPage --> Modified[Modified Data]
    end
```

### 3. Memory-Mapped File I/O
```mermaid
graph TB
    subgraph "Memory Mapping"
        VirtualMem[Virtual Memory Space]
        PhysicalMem[Physical Memory]
        DiskFile[Database File on Disk]
        
        VirtualMem -.-> PhysicalMem
        PhysicalMem -.-> DiskFile
        
        subgraph "Page Management"
            ActivePages[Active Pages in Memory]
            SwappedPages[Swapped Pages on Disk]
        end
        
        PhysicalMem --> ActivePages
        DiskFile --> SwappedPages
    end
```

## Data Flow Overview

The following diagram shows how data flows through the system during a typical operation:

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant Tx as Transaction
    participant Bucket as Bucket
    participant BTree as B+Tree
    participant Node as Nodes
    participant Page as Pages
    participant File as Disk File
    
    App->>DB: Open/Create Database
    DB->>File: Memory map file
    
    App->>DB: Begin Transaction
    DB->>Tx: Create transaction
    Tx->>Tx: Lock resources
    
    App->>Tx: Get/Create Bucket
    Tx->>Bucket: Access bucket
    
    App->>Bucket: Put/Get/Delete
    Bucket->>BTree: Tree operations
    BTree->>Node: Modify nodes
    Node->>Page: Page operations
    
    App->>Tx: Commit
    Tx->>Node: Serialize dirty nodes
    Node->>Page: Write pages
    Page->>File: Sync to disk
    Tx->>Tx: Release locks
```

## Performance Characteristics

### Read Performance
- **Memory-mapped I/O**: Direct memory access to disk pages
- **B+tree structure**: O(log n) key lookups
- **Page-level caching**: OS manages page cache automatically
- **Multiple concurrent readers**: No blocking between read transactions

### Write Performance
- **Single writer**: No write-write conflicts to resolve
- **Copy-on-write**: Minimal page copying during modifications
- **Batch commits**: Efficient bulk operations via `DB.Batch()`
- **Freelist management**: Efficient space reuse

### Storage Efficiency
- **B+tree packing**: Configurable fill percentage for pages
- **Page reuse**: Freelist tracks and reuses deleted pages
- **Minimal overhead**: Simple file format with low metadata overhead
- **Compression friendly**: Sequential key storage benefits from compression

## Deployment Considerations

### File System Requirements
- **POSIX compliance**: Requires proper fsync() support
- **Memory mapping**: Benefits from efficient mmap() implementation
- **File locking**: Uses flock() for exclusive access

### Operating System Support
- **Cross-platform**: Works on Linux, macOS, Windows
- **Architecture support**: 32-bit and 64-bit systems
- **Endianness**: Database files are architecture-specific

### Resource Usage
- **Memory**: Uses memory mapping, size scales with active data
- **CPU**: Efficient B+tree operations, minimal overhead
- **Disk I/O**: Sequential writes, memory-mapped reads
- **File handles**: Single file handle per database

This architecture provides a robust foundation for an embedded database that balances simplicity, performance, and reliability.