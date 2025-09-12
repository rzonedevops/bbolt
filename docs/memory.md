# bbolt Memory Management

bbolt uses memory-mapped I/O and sophisticated caching strategies to provide efficient access to database pages while minimizing memory usage. This document details the memory management architecture, page caching, and optimization strategies.

## Memory Architecture Overview

```mermaid
graph TB
    subgraph "Application Memory Space"
        AppData[Application Data]
        BBoltStructs[bbolt Structures]
        NodeCache[Node Cache]
    end
    
    subgraph "Memory-Mapped Region"
        MMapRegion[Memory-Mapped Database File]
        MetaPages[Meta Pages]
        DataPages[Data Pages]
        FreelistPages[Freelist Pages]
    end
    
    subgraph "Operating System"
        PageCache[OS Page Cache]
        PhysicalRAM[Physical RAM]
        SwapSpace[Swap Space]
    end
    
    subgraph "Storage"
        DatabaseFile[(Database File)]
    end
    
    BBoltStructs --> NodeCache
    NodeCache -.-> MMapRegion
    MMapRegion --> MetaPages
    MMapRegion --> DataPages
    MMapRegion --> FreelistPages
    
    MMapRegion -.-> PageCache
    PageCache --> PhysicalRAM
    PageCache -.-> SwapSpace
    PageCache -.-> DatabaseFile
```

## Memory-Mapped I/O

bbolt uses memory mapping to provide direct access to database pages without explicit I/O operations.

### Memory Mapping Process

```mermaid
sequenceDiagram
    participant DB as Database
    participant OS as Operating System
    participant FS as File System
    participant RAM as Physical Memory
    
    DB->>OS: Open database file
    OS->>FS: Open file handle
    FS-->>OS: File descriptor
    
    DB->>OS: mmap(file, size, PROT_READ)
    OS->>FS: Map file to virtual memory
    OS->>RAM: Reserve virtual address space
    OS-->>DB: Virtual memory pointer
    
    Note over DB: Database can now access pages directly
    
    DB->>RAM: Access page data
    alt Page in RAM
        RAM-->>DB: Direct memory access
    else Page not in RAM
        RAM->>FS: Page fault - load from disk
        FS->>RAM: Load page into memory
        RAM-->>DB: Memory access complete
    end
```

### Memory Mapping Benefits

```mermaid
graph TD
    subgraph "Memory Mapping Advantages"
        DirectAccess[Direct Memory Access]
        OSManaged[OS-Managed Caching]
        ZeroCopy[Zero-Copy Operations]
        LazyLoading[Lazy Page Loading]
        SharedMemory[Shared Memory Across Processes]
        
        DirectAccess --> FastReads[Fast Read Performance]
        OSManaged --> EfficientCache[Efficient Cache Management]
        ZeroCopy --> LowOverhead[Low CPU Overhead]
        LazyLoading --> MemoryEfficiency[Memory Efficiency]
        SharedMemory --> ProcessSharing[Inter-Process Sharing]
    end
    
    subgraph "Implementation Details"
        MMAP[mmap() System Call]
        VirtualMemory[Virtual Memory Management]
        PageFaults[Page Fault Handling]
        
        MMAP --> VirtualMemory
        VirtualMemory --> PageFaults
        PageFaults --> LazyLoading
    end
```

## Node Caching Strategy

bbolt maintains an in-memory cache of deserialized B+tree nodes to avoid repeated parsing of page data.

### Node Cache Architecture

```mermaid
graph TD
    subgraph "Node Cache Structure"
        TxNodeMap[Transaction Node Map]
        BucketNodes[Bucket Node References]
        DirtyNodes[Dirty Node Set]
        CleanNodes[Clean Node Set]
        
        TxNodeMap --> BucketNodes
        BucketNodes --> DirtyNodes
        BucketNodes --> CleanNodes
    end
    
    subgraph "Node Lifecycle"
        PageData[Page Data]
        ParseNode[Parse to Node]
        CacheNode[Cache Node]
        ModifyNode[Modify Node]
        SerializeNode[Serialize Node]
        NewPage[New Page]
        
        PageData --> ParseNode
        ParseNode --> CacheNode
        CacheNode --> ModifyNode
        ModifyNode --> SerializeNode
        SerializeNode --> NewPage
    end
    
    subgraph "Cache Management"
        ReadTx[Read Transaction]
        WriteTx[Write Transaction]
        NodeReuse[Node Reuse]
        CacheEviction[Cache Eviction]
        
        ReadTx --> NodeReuse
        WriteTx --> DirtyNodes
        WriteTx --> CacheEviction
    end
```

### Node Cache Implementation

```mermaid
classDiagram
    class Transaction {
        +pages map[pgid]*page
        +nodes map[pgid]*node
        +root Bucket
        +getNode(pgid) *node
        +deleteNode(pgid)
    }
    
    class Bucket {
        +nodes map[pgid]*node
        +rootNode *node
        +node(pgid, parent) *node
        +rebalance()
    }
    
    class Node {
        +bucket *Bucket
        +isLeaf bool
        +unbalanced bool
        +spilled bool
        +key []byte
        +pgid pgid
        +parent *node
        +children nodes
        +inodes inodes
        +read(page)
        +write(page)
        +spill() error
    }
    
    Transaction ||--o{ Bucket : manages
    Bucket ||--o{ Node : caches
    Node ||--|| Node : parent/child
```

## Page Management

bbolt manages database pages through a combination of memory mapping and active page tracking.

### Page Access Pattern

```mermaid
sequenceDiagram
    participant App as Application
    participant Tx as Transaction
    participant Bucket as Bucket
    participant Node as Node
    participant Page as Page
    participant MMap as Memory Map
    
    App->>Tx: Read operation
    Tx->>Bucket: Access bucket
    Bucket->>Node: Get node for pgid
    
    alt Node in cache
        Node-->>Bucket: Return cached node
    else Node not in cache
        Node->>Page: Read page data
        Page->>MMap: Access memory-mapped region
        MMap-->>Page: Page bytes
        Page->>Node: Deserialize page to node
        Node->>Bucket: Cache and return node
    end
    
    Bucket-->>Tx: Node data
    Tx-->>App: Result
```

### Page Cache Integration

```mermaid
graph TD
    subgraph "Page Cache Hierarchy"
        L1Cache[L1: Node Cache<br/>Deserialized objects]
        L2Cache[L2: OS Page Cache<br/>Raw page data]
        L3Storage[L3: Disk Storage<br/>Database file]
        
        L1Cache -->|Cache miss| L2Cache
        L2Cache -->|Page fault| L3Storage
        L3Storage -->|Load page| L2Cache
        L2Cache -->|Parse page| L1Cache
    end
    
    subgraph "Cache Characteristics"
        NodeCacheProps[Node Cache<br/>• Small, focused<br/>• Application-managed<br/>• Structured data]
        OSCacheProps[OS Page Cache<br/>• Large capacity<br/>• OS-managed<br/>• Raw bytes]
        DiskProps[Disk Storage<br/>• Persistent<br/>• High latency<br/>• Large capacity]
        
        L1Cache -.-> NodeCacheProps
        L2Cache -.-> OSCacheProps
        L3Storage -.-> DiskProps
    end
```

## Memory Usage Patterns

### Read Operations Memory Usage

```mermaid
graph TD
    subgraph "Read Operation Memory Flow"
        ReadRequest[Read Request]
        NodeCheck[Check Node Cache]
        PageAccess[Access Memory-Mapped Page]
        ParseNode[Parse Page to Node]
        CacheNode[Cache Parsed Node]
        ReturnData[Return Data]
        
        ReadRequest --> NodeCheck
        NodeCheck -->|Hit| ReturnData
        NodeCheck -->|Miss| PageAccess
        PageAccess --> ParseNode
        ParseNode --> CacheNode
        CacheNode --> ReturnData
    end
    
    subgraph "Memory Allocation"
        StackAlloc[Stack Allocation<br/>Local variables]
        HeapAlloc[Heap Allocation<br/>Node structures]
        NoAlloc[No Allocation<br/>Memory-mapped access]
        
        ParseNode --> HeapAlloc
        PageAccess --> NoAlloc
        ReadRequest --> StackAlloc
    end
```

### Write Operations Memory Usage

```mermaid
sequenceDiagram
    participant Tx as Write Transaction
    participant Node as Node Cache
    participant Page as Page Buffer
    participant Dirty as Dirty Set
    participant MMap as Memory Map
    
    Tx->>Node: Modify node
    Node->>Node: Mark as dirty
    Node->>Dirty: Add to dirty set
    
    Note over Tx: Continue modifications
    
    Tx->>Tx: Commit()
    
    loop For each dirty node
        Tx->>Node: Serialize node
        Node->>Page: Create new page
        Page->>Dirty: Buffer for write
    end
    
    Tx->>MMap: Write dirty pages
    MMap->>MMap: Update memory-mapped region
    
    Tx->>Dirty: Clear dirty set
    Tx->>Node: Update node references
```

## Memory Optimization Strategies

### Large Database Handling

bbolt can handle databases larger than available RAM through memory mapping and OS page cache management:

```mermaid
graph TD
    subgraph "Large Database Architecture"
        VirtualMemory[Virtual Memory Space<br/>64-bit: ~18 exabytes]
        PhysicalRAM[Physical RAM<br/>Limited capacity]
        DatabaseFile[Database File<br/>Can exceed RAM size]
        
        VirtualMemory -.-> PhysicalRAM
        VirtualMemory -.-> DatabaseFile
    end
    
    subgraph "Access Patterns"
        HotData[Hot Data<br/>Frequently accessed]
        WarmData[Warm Data<br/>Occasionally accessed]
        ColdData[Cold Data<br/>Rarely accessed]
        
        HotData --> PhysicalRAM
        WarmData -.-> PhysicalRAM
        ColdData --> DatabaseFile
    end
    
    subgraph "OS Optimization"
        ReadAhead[Read-ahead caching]
        LRUEviction[LRU page eviction]
        BackgroundSync[Background sync]
        
        ReadAhead --> WarmData
        LRUEviction --> ColdData
        BackgroundSync --> DatabaseFile
    end
```

### Memory Usage Monitoring

```mermaid
graph TD
    subgraph "Memory Metrics"
        VirtualSize[Virtual Memory Size]
        ResidentSize[Resident Set Size]
        SharedMemory[Shared Memory]
        PrivateMemory[Private Memory]
        
        VirtualSize -.-> DatabaseSize[~Database File Size]
        ResidentSize -.-> ActivePages[Active Pages in RAM]
        SharedMemory -.-> OSPageCache[OS Page Cache]
        PrivateMemory -.-> ApplicationData[bbolt Structures]
    end
    
    subgraph "Monitoring Tools"
        ProcessTools[Process monitors<br/>top, ps, htop]
        SystemTools[System monitors<br/>free, vmstat]
        ProfilerTools[Go profilers<br/>pprof, trace]
        
        ProcessTools --> ResidentSize
        SystemTools --> SharedMemory
        ProfilerTools --> PrivateMemory
    end
```

## Platform-Specific Considerations

### Memory Mapping Implementation

```mermaid
graph TD
    subgraph "Platform Differences"
        Unix[Unix/Linux<br/>mmap(), madvise()]
        Windows[Windows<br/>CreateFileMapping(), MapViewOfFile()]
        
        Unix --> UnixFlags[MAP_SHARED, MAP_PRIVATE]
        Windows --> WindowsFlags[PAGE_READONLY, FILE_MAP_READ]
    end
    
    subgraph "Platform Optimizations"
        UnixOpt[Unix Optimizations<br/>• MADV_RANDOM<br/>• MADV_SEQUENTIAL<br/>• MADV_WILLNEED]
        WindowsOpt[Windows Optimizations<br/>• File caching hints<br/>• Memory mapping flags<br/>• NUMA awareness]
        
        Unix --> UnixOpt
        Windows --> WindowsOpt
    end
```

### 32-bit vs 64-bit Considerations

```mermaid
graph TD
    subgraph "Address Space Limitations"
        Bit32[32-bit Systems<br/>~4GB virtual address space]
        Bit64[64-bit Systems<br/>~18 exabytes virtual address space]
        
        Bit32 --> Limited[Limited database size<br/>~2GB practical limit]
        Bit64 --> Unlimited[Virtually unlimited<br/>database size]
    end
    
    subgraph "Implementation Strategies"
        Bit32Impl[32-bit Implementation<br/>• Careful memory management<br/>• Regular unmapping<br/>• Size limits]
        Bit64Impl[64-bit Implementation<br/>• Simple mapping<br/>• Large file support<br/>• No size concerns]
        
        Bit32 --> Bit32Impl
        Bit64 --> Bit64Impl
    end
```

## Performance Best Practices

### Memory Usage Optimization

1. **Use Read-Only Transactions When Possible**
   ```go
   // Read-only transactions don't create dirty node copies
   db.View(func(tx *bolt.Tx) error {
       // No memory allocation for modifications
       return nil
   })
   ```

2. **Keep Transactions Short**
   ```go
   // Long transactions hold memory longer
   // Prefer short, focused transactions
   ```

3. **Configure OS for Database Workloads**
   ```bash
   # Linux: Tune VM settings for database
   echo 'vm.swappiness=1' >> /etc/sysctl.conf
   echo 'vm.dirty_ratio=15' >> /etc/sysctl.conf
   ```

4. **Monitor Memory Usage**
   ```go
   // Check database statistics
   stats := db.Stats()
   fmt.Printf("Open transactions: %d\n", stats.OpenTxN)
   fmt.Printf("Pending pages: %d\n", stats.PendingPageN)
   ```

### Memory Efficiency Patterns

```mermaid
graph TD
    subgraph "Efficient Patterns"
        ShortTx[Short Transactions]
        ReadOnlyTx[Read-Only When Possible]
        BatchWrites[Batch Write Operations]
        ProperClose[Proper Resource Cleanup]
        
        ShortTx --> LessMemory[Reduced Memory Holding]
        ReadOnlyTx --> NoAllocation[No Dirty Node Allocation]
        BatchWrites --> EfficientWrites[Efficient Write Batching]
        ProperClose --> ResourceCleanup[Proper Resource Cleanup]
    end
    
    subgraph "Anti-Patterns"
        LongReadTx[Long Read Transactions]
        FrequentSmallWrites[Frequent Small Writes]
        LeakedTx[Unclosed Transactions]
        
        LongReadTx --> MemoryBloat[Memory Bloat]
        FrequentSmallWrites --> Overhead[High Overhead]
        LeakedTx --> ResourceLeak[Resource Leaks]
    end
```

This memory management system provides efficient access to database content while minimizing memory usage through intelligent caching, memory mapping, and OS integration. The architecture scales from small embedded applications to large server deployments handling multi-gigabyte databases.