# bbolt Storage Engine

bbolt uses a B+tree storage engine with copy-on-write semantics to provide efficient key/value storage. This document details the internal storage structure, page management, and disk layout.

## B+Tree Structure

bbolt implements a B+tree where all values are stored in leaf nodes, and internal nodes contain only keys and pointers for navigation.

### B+Tree Overview

```mermaid
graph TD
    subgraph "B+Tree Properties"
        Root[Root Node<br/>Level 2]
        Internal1[Internal Node<br/>Level 1]
        Internal2[Internal Node<br/>Level 1]
        Leaf1[Leaf Node<br/>Level 0]
        Leaf2[Leaf Node<br/>Level 0]
        Leaf3[Leaf Node<br/>Level 0]
        Leaf4[Leaf Node<br/>Level 0]
        
        Root --> Internal1
        Root --> Internal2
        Internal1 --> Leaf1
        Internal1 --> Leaf2
        Internal2 --> Leaf3
        Internal2 --> Leaf4
        
        Leaf1 -.->|Next| Leaf2
        Leaf2 -.->|Next| Leaf3
        Leaf3 -.->|Next| Leaf4
    end
    
    subgraph "Node Contents"
        InternalContent["Internal Node:<br/>Keys: [20, 40]<br/>Pointers: [P1, P2, P3]"]
        LeafContent["Leaf Node:<br/>Keys: [10, 15]<br/>Values: [V10, V15]"]
    end
```

### Tree Operations

```mermaid
flowchart TD
    subgraph "Insert Operation"
        Insert[Insert Key/Value]
        FindLeaf[Find Target Leaf]
        CheckSpace[Check Available Space]
        
        Insert --> FindLeaf
        FindLeaf --> CheckSpace
        
        CheckSpace -->|Space Available| DirectInsert[Insert Directly]
        CheckSpace -->|No Space| SplitNode[Split Node]
        
        SplitNode --> PropagateUp[Propagate Split Up]
        PropagateUp --> UpdateParent[Update Parent Node]
        UpdateParent -->|Parent Full| SplitParent[Split Parent]
        SplitParent --> PropagateUp
    end
    
    subgraph "Delete Operation"
        Delete[Delete Key]
        FindKey[Locate Key in Leaf]
        RemoveKey[Remove Key/Value]
        CheckFill[Check Fill Percentage]
        
        Delete --> FindKey
        FindKey --> RemoveKey
        RemoveKey --> CheckFill
        
        CheckFill -->|Adequate Fill| Done[Complete]
        CheckFill -->|Under-filled| Rebalance[Rebalance with Sibling]
        Rebalance -->|Can Rebalance| RedistributeKeys[Redistribute Keys]
        Rebalance -->|Cannot Rebalance| MergeNodes[Merge with Sibling]
        MergeNodes --> PropagateDelete[Propagate Merge Up]
    end
```

## Page Structure

bbolt organizes data into fixed-size pages (typically 4KB) that map directly to operating system pages for efficient memory mapping.

### Page Types

```mermaid
graph TD
    subgraph "Page Type Hierarchy"
        Page[Generic Page]
        MetaPage[Meta Page<br/>Database metadata]
        FreelistPage[Freelist Page<br/>Available pages]
        BranchPage[Branch Page<br/>Internal B+tree nodes]
        LeafPage[Leaf Page<br/>B+tree leaf nodes]
        
        Page --> MetaPage
        Page --> FreelistPage
        Page --> BranchPage
        Page --> LeafPage
    end
    
    subgraph "Page Layout"
        Header[Page Header<br/>16 bytes]
        Data[Page Data<br/>Variable size]
        
        Header --> PageID[Page ID - 8 bytes]
        Header --> Flags[Flags - 2 bytes]
        Header --> Count[Count - 2 bytes]
        Header --> Overflow[Overflow - 4 bytes]
    end
```

### Page Header Structure

```go
type page struct {
    id       pgid     // Page ID (8 bytes)
    flags    uint16   // Page type flags (2 bytes)
    count    uint16   // Number of elements (2 bytes)
    overflow uint32   // Number of overflow pages (4 bytes)
    ptr      uintptr  // Pointer to page data
}
```

### Page Flags

```mermaid
graph LR
    subgraph "Page Type Flags"
        MetaFlag[0x01 - Meta Page]
        FreelistFlag[0x02 - Freelist Page]
        BranchFlag[0x04 - Branch Page]
        LeafFlag[0x08 - Leaf Page]
    end
    
    subgraph "Special Flags"
        OverflowFlag[Overflow pages for large values]
        DeletedFlag[Marked for deletion]
    end
```

## Disk Layout

The database file has a specific layout that enables efficient access and crash recovery.

### File Layout

```mermaid
graph TB
    subgraph "Database File Structure"
        Meta0[Meta Page 0<br/>Page ID: 0]
        Meta1[Meta Page 1<br/>Page ID: 1]
        Freelist[Freelist Page<br/>Page ID: 2]
        RootPage[Root Page<br/>Page ID: 3]
        DataPages[Data Pages<br/>Page ID: 4+]
        
        Meta0 -.->|Primary| DBInfo[Database Metadata]
        Meta1 -.->|Backup| DBInfo
        Freelist -.-> FreePageList[List of Available Pages]
        RootPage -.-> TreeRoot[B+Tree Root]
        DataPages -.-> TreeNodes[B+Tree Nodes]
    end
    
    subgraph "Meta Page Content"
        Magic[Magic Number]
        Version[Version Number]
        PageSize[Page Size]
        RootPageID[Root Page ID]
        FreelistPageID[Freelist Page ID]
        TxID[Transaction ID]
        Checksum[Checksum]
    end
```

### Meta Page Structure

```mermaid
classDiagram
    class MetaPage {
        +magic uint32
        +version uint32
        +pageSize uint32
        +flags uint32
        +root bucket
        +freelist pgid
        +pgid pgid
        +txid txid
        +checksum uint64
        +validate() error
        +copy() meta
        +write(page) error
    }
    
    class Bucket {
        +root pgid
        +sequence uint64
    }
    
    MetaPage --> Bucket : contains root bucket
```

## Storage Operations

### Key Lookup Process

```mermaid
sequenceDiagram
    participant Client as Client
    participant Bucket as Bucket
    participant Cursor as Cursor
    participant Node as Node
    participant Page as Page
    participant MMap as Memory Map
    
    Client->>Bucket: Get(key)
    Bucket->>Cursor: Seek(key)
    Cursor->>Node: Navigate tree
    
    loop Tree traversal
        Node->>Page: Read page data
        Page->>MMap: Access memory-mapped data
        MMap-->>Page: Page content
        Page-->>Node: Node structure
        Node->>Node: Binary search for key
        alt Internal node
            Node->>Node: Follow pointer to child
        else Leaf node
            Node-->>Cursor: Key/value found or not found
        end
    end
    
    Cursor-->>Bucket: Result
    Bucket-->>Client: Value or nil
```

### Page Allocation and Deallocation

```mermaid
graph TD
    subgraph "Page Allocation"
        AllocRequest[Allocation Request]
        CheckFreelist[Check Freelist]
        
        AllocRequest --> CheckFreelist
        CheckFreelist -->|Pages Available| ReusePage[Reuse Free Page]
        CheckFreelist -->|No Free Pages| ExtendFile[Extend Database File]
        
        ReusePage --> UpdateFreelist[Update Freelist]
        ExtendFile --> NewPage[Create New Page]
        
        UpdateFreelist --> AllocatedPage[Allocated Page]
        NewPage --> AllocatedPage
    end
    
    subgraph "Page Deallocation"
        DeletePage[Page to Delete]
        AddToFreelist[Add to Freelist]
        CoalesceCheck[Check for Coalescing]
        
        DeletePage --> AddToFreelist
        AddToFreelist --> CoalesceCheck
        CoalesceCheck -->|Adjacent Pages Free| CoalescePages[Coalesce Pages]
        CoalesceCheck -->|No Adjacent Pages| DeallocComplete[Deallocation Complete]
        CoalescePages --> DeallocComplete
    end
```

### Copy-on-Write Implementation

bbolt uses copy-on-write to maintain consistency during modifications:

```mermaid
sequenceDiagram
    participant Tx as Transaction
    participant OrigNode as Original Node
    participant NewNode as New Node
    participant OrigPage as Original Page
    participant NewPage as New Page
    participant Freelist as Freelist
    
    Tx->>OrigNode: Modify node
    OrigNode->>NewNode: Create copy
    OrigNode->>NewNode: Copy existing data
    NewNode->>NewNode: Apply modifications
    
    Tx->>Freelist: Allocate new page
    Freelist-->>Tx: New page ID
    
    NewNode->>NewPage: Serialize to new page
    NewPage->>Tx: Mark as dirty
    
    Note over OrigPage: Original page remains unchanged
    Note over NewPage: New page contains modifications
    
    Tx->>Tx: Commit
    Tx->>Freelist: Mark original page as free
    Tx->>OrigPage: Add to freelist for next transaction
```

## Space Management

### Freelist Management

```mermaid
graph TD
    subgraph "Freelist Structure"
        FreelistPage[Freelist Page]
        FreelistType{Freelist Type}
        
        FreelistType -->|Array| ArrayFreelist[Array-based Freelist]
        FreelistType -->|Hashmap| HashmapFreelist[Hashmap-based Freelist]
    end
    
    subgraph "Array Freelist"
        FreeArray[Free Page Array]
        PendingArray[Pending Free Array]
        
        FreeArray -.-> ImmediateReuse[Immediate Reuse]
        PendingArray -.-> DelayedReuse[Delayed Reuse]
    end
    
    subgraph "Hashmap Freelist"
        FreeMap[Free Page Hashmap]
        PendingMap[Pending Free Hashmap]
        
        FreeMap -.-> FastLookup[O(1) Lookup]
        PendingMap -.-> FastPending[O(1) Pending Operations]
    end
```

### Page Splitting and Merging

When a page becomes too full or too empty, bbolt performs splitting or merging operations:

```mermaid
graph TD
    subgraph "Page Splitting"
        FullPage[Full Page<br/>FillPercent > 100%]
        SplitPoint[Calculate Split Point]
        LeftPage[Left Page<br/>50% full]
        RightPage[Right Page<br/>50% full]
        UpdateParent[Update Parent Node]
        
        FullPage --> SplitPoint
        SplitPoint --> LeftPage
        SplitPoint --> RightPage
        LeftPage --> UpdateParent
        RightPage --> UpdateParent
    end
    
    subgraph "Page Merging"
        UnderfilledPage[Underfilled Page<br/>FillPercent < MinFill]
        CheckSibling[Check Sibling Capacity]
        MergePage[Merge with Sibling]
        UpdateParentAfterMerge[Update Parent Node]
        
        UnderfilledPage --> CheckSibling
        CheckSibling -->|Can Merge| MergePage
        CheckSibling -->|Cannot Merge| Redistribute[Redistribute Keys]
        MergePage --> UpdateParentAfterMerge
        Redistribute --> UpdateParentAfterMerge
    end
```

## Performance Optimizations

### Fill Percentage Configuration

```mermaid
graph TD
    subgraph "Fill Percentage Impact"
        FillPercent[Fill Percentage]
        
        FillPercent -->|Low 25%| MoreSplits[More Page Splits<br/>More Space Overhead<br/>Better Insert Performance]
        FillPercent -->|Medium 50%| Balanced[Balanced Performance<br/>Default Setting]
        FillPercent -->|High 90%| FewerSplits[Fewer Page Splits<br/>Better Space Utilization<br/>More Copy Overhead]
    end
    
    subgraph "Use Case Optimization"
        BulkInsert[Bulk Insert Workload]
        RandomInsert[Random Insert Workload]
        ReadHeavy[Read-Heavy Workload]
        
        BulkInsert -.-> LowFill[Low Fill Percentage]
        RandomInsert -.-> MediumFill[Medium Fill Percentage]
        ReadHeavy -.-> HighFill[High Fill Percentage]
    end
```

### Memory Mapping Benefits

```mermaid
graph TD
    subgraph "Memory Mapping Advantages"
        MMapFile[Memory-Mapped File]
        
        MMapFile --> ZeroCopy[Zero-Copy Access]
        MMapFile --> OSCache[OS Page Cache Integration]
        MMapFile --> LazyLoading[Lazy Page Loading]
        MMapFile --> VirtualMemory[Virtual Memory Benefits]
        
        ZeroCopy --> FastReads[Fast Read Operations]
        OSCache --> SharedCache[Shared Cache Across Processes]
        LazyLoading --> MemoryEfficiency[Memory Efficiency]
        VirtualMemory --> LargeDB[Support for Large Databases]
    end
    
    subgraph "Page Cache Integration"
        ReadRequest[Read Request]
        PageCache[OS Page Cache]
        DiskIO[Disk I/O]
        
        ReadRequest --> PageCache
        PageCache -->|Hit| FastAccess[Direct Memory Access]
        PageCache -->|Miss| DiskIO
        DiskIO --> LoadPage[Load Page to Cache]
        LoadPage --> FastAccess
    end
```

## Crash Recovery

bbolt ensures crash consistency through careful ordering of writes and dual metadata pages:

```mermaid
sequenceDiagram
    participant App as Application
    participant Tx as Transaction
    participant Pages as Data Pages
    participant Meta as Metadata
    participant Disk as Disk Storage
    
    App->>Tx: Commit transaction
    
    Note over Tx: Phase 1: Write data pages
    Tx->>Pages: Write all dirty pages
    Pages->>Disk: Write data to disk
    Tx->>Disk: fsync() data pages
    
    Note over Tx: Phase 2: Write metadata
    Tx->>Meta: Write new metadata page
    Meta->>Disk: Write metadata to disk
    Tx->>Disk: fsync() metadata page
    
    Note over Tx: Transaction committed
    Tx-->>App: Success
    
    Note over Disk: Crash-safe state achieved
```

### Recovery Process

```mermaid
graph TD
    subgraph "Database Open Recovery"
        OpenDB[Open Database]
        ReadMeta0[Read Meta Page 0]
        ReadMeta1[Read Meta Page 1]
        ValidateMeta[Validate Metadata]
        
        OpenDB --> ReadMeta0
        OpenDB --> ReadMeta1
        ReadMeta0 --> ValidateMeta
        ReadMeta1 --> ValidateMeta
        
        ValidateMeta -->|Both Valid| UseLatest[Use Latest Transaction ID]
        ValidateMeta -->|One Valid| UseValid[Use Valid Metadata]
        ValidateMeta -->|None Valid| CorruptDB[Database Corrupted]
        
        UseLatest --> ReadyDB[Database Ready]
        UseValid --> ReadyDB
        CorruptDB --> Error[Return Error]
    end
```

This storage engine design provides efficient access patterns, crash safety, and good performance characteristics for both read and write operations while maintaining data integrity through its B+tree structure and MVCC implementation.