# bbolt API Operations Flow

This document provides detailed flow diagrams for key bbolt operations, showing how requests flow through the system and how different components interact.

## Database Lifecycle Operations

### Database Open Operation

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant File as File System
    participant Meta as Metadata
    participant MMap as Memory Mapping
    participant Lock as File Lock
    
    App->>DB: bolt.Open(path, mode, options)
    
    DB->>File: Open database file
    alt File doesn't exist
        File->>File: Create new file
        File->>DB: Write initial pages
    else File exists
        File-->>DB: Existing file handle
    end
    
    DB->>Lock: Acquire exclusive file lock
    alt Lock acquired
        Lock-->>DB: Lock successful
    else Lock failed
        Lock-->>DB: Lock timeout/error
        DB-->>App: Return error
    end
    
    DB->>MMap: Memory map file
    MMap->>File: Map file to virtual memory
    MMap-->>DB: Memory-mapped pointer
    
    DB->>Meta: Read metadata pages
    Meta->>Meta: Validate metadata
    alt Metadata valid
        Meta-->>DB: Database ready
        DB-->>App: *DB instance
    else Metadata invalid
        Meta-->>DB: Corruption detected
        DB-->>App: Return error
    end
```

### Database Close Operation

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant Tx as Active Transactions
    participant MMap as Memory Mapping
    participant Lock as File Lock
    participant File as File System
    
    App->>DB: db.Close()
    
    DB->>Tx: Wait for active transactions
    Tx->>Tx: Complete pending operations
    Tx-->>DB: All transactions finished
    
    DB->>MMap: Unmap memory
    MMap->>File: Sync any pending changes
    MMap-->>DB: Unmapping complete
    
    DB->>Lock: Release file lock
    Lock-->>DB: Lock released
    
    DB->>File: Close file handle
    File-->>DB: File closed
    
    DB-->>App: Close complete
```

## Transaction Operations

### Read Transaction Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant Tx as Transaction
    participant Meta as Metadata
    participant Bucket as Bucket
    participant Node as Node
    participant Page as Page
    
    App->>DB: db.View(func(tx *bolt.Tx) error {...})
    
    DB->>Tx: Begin read transaction
    Tx->>Meta: Read current metadata
    Meta-->>Tx: Consistent snapshot
    
    App->>Tx: tx.Bucket("mybucket")
    Tx->>Bucket: Access bucket
    Bucket->>Meta: Find bucket root page
    Meta-->>Bucket: Root page ID
    
    App->>Bucket: bucket.Get("mykey")
    Bucket->>Node: Navigate B+tree
    
    loop Tree traversal
        Node->>Page: Read page data
        Page-->>Node: Page content
        Node->>Node: Binary search for key
        alt Internal node
            Node->>Node: Follow child pointer
        else Leaf node found
            Node-->>Bucket: Key/value result
        end
    end
    
    Bucket-->>App: Return value
    
    App->>Tx: return nil (from function)
    Tx->>Tx: Rollback (cleanup)
    Tx-->>DB: Transaction complete
    DB-->>App: View complete
```

### Write Transaction Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant Tx as Transaction
    participant Lock as Write Lock
    participant Meta as Metadata
    participant Bucket as Bucket
    participant Node as Node
    participant Page as Page
    participant Freelist as Freelist
    participant Disk as Disk
    
    App->>DB: db.Update(func(tx *bolt.Tx) error {...})
    
    DB->>Lock: Acquire write lock
    Lock-->>DB: Write lock acquired
    
    DB->>Tx: Begin write transaction
    Tx->>Meta: Read current metadata
    Meta-->>Tx: Current state
    
    App->>Tx: tx.Bucket("mybucket")
    Tx->>Bucket: Access bucket
    
    App->>Bucket: bucket.Put("mykey", "myvalue")
    Bucket->>Node: Navigate to target leaf
    Node->>Node: Insert/update key/value
    Node->>Node: Mark node as dirty
    
    alt Node overflow
        Node->>Node: Split node
        Node->>Bucket: Update parent references
        Bucket->>Node: Propagate changes up tree
    end
    
    App->>Tx: return nil (from function)
    
    Tx->>Tx: Commit transaction
    
    loop For each dirty node
        Tx->>Freelist: Allocate new page
        Freelist-->>Tx: New page ID
        Tx->>Node: Serialize node to page
        Node->>Page: Create page data
        Page->>Disk: Write page to file
    end
    
    Tx->>Meta: Update metadata
    Meta->>Disk: Write metadata page
    Tx->>Disk: fsync()
    
    Tx->>Lock: Release write lock
    Lock-->>DB: Write lock released
    
    DB-->>App: Update complete
```

## Bucket Operations

### Bucket Creation

```mermaid
graph TD
    subgraph "Create Bucket Flow"
        Start[CreateBucket Request]
        CheckExists[Check if Bucket Exists]
        CreateRoot[Create Root Page]
        UpdateParent[Update Parent Bucket]
        AllocatePage[Allocate New Page]
        InitializePage[Initialize Empty Page]
        Success[Return New Bucket]
        
        Start --> CheckExists
        CheckExists -->|Doesn't Exist| CreateRoot
        CheckExists -->|Exists| Error[Return Error]
        CreateRoot --> AllocatePage
        AllocatePage --> InitializePage
        InitializePage --> UpdateParent
        UpdateParent --> Success
    end
    
    subgraph "Nested Bucket Creation"
        NestedCreate[Create Nested Bucket]
        ParentBucket[Access Parent Bucket]
        CreateSubBucket[Create Sub-bucket Entry]
        UpdateBTree[Update Parent B+tree]
        
        NestedCreate --> ParentBucket
        ParentBucket --> CreateSubBucket
        CreateSubBucket --> UpdateBTree
        UpdateBTree --> Success
    end
```

### Bucket Deletion

```mermaid
sequenceDiagram
    participant App as Application
    participant Tx as Transaction
    participant Bucket as Parent Bucket
    participant SubBucket as Target Bucket
    participant Node as B+tree Nodes
    participant Freelist as Freelist
    
    App->>Tx: tx.DeleteBucket("mybucket")
    Tx->>Bucket: Access parent bucket
    Bucket->>SubBucket: Find target bucket
    
    SubBucket->>SubBucket: Recursively delete sub-buckets
    
    loop For each page in bucket
        SubBucket->>Node: Get page references
        Node->>Freelist: Mark page as free
    end
    
    SubBucket->>Bucket: Remove bucket entry
    Bucket->>Node: Update parent B+tree
    Node->>Node: Remove bucket key
    
    alt Node underflow
        Node->>Node: Rebalance with siblings
    end
    
    SubBucket-->>App: Deletion complete
```

## Key/Value Operations

### Put Operation Detailed Flow

```mermaid
graph TD
    subgraph "Put Operation Flow"
        PutStart[Put Key/Value]
        FindLeaf[Navigate to Target Leaf]
        BinarySearch[Binary Search in Leaf]
        
        PutStart --> FindLeaf
        FindLeaf --> BinarySearch
        
        BinarySearch -->|Key Exists| UpdateValue[Update Existing Value]
        BinarySearch -->|Key Missing| InsertNew[Insert New Key/Value]
        
        UpdateValue --> CheckSize[Check Value Size Change]
        InsertNew --> CheckSpace[Check Available Space]
        
        CheckSize -->|Same Size| InPlaceUpdate[In-place Update]
        CheckSize -->|Different Size| Reorganize[Reorganize Node]
        
        CheckSpace -->|Space Available| DirectInsert[Direct Insert]
        CheckSpace -->|No Space| SplitNode[Split Node]
        
        InPlaceUpdate --> MarkDirty[Mark Node Dirty]
        Reorganize --> MarkDirty
        DirectInsert --> MarkDirty
        SplitNode --> UpdateParent[Update Parent References]
        UpdateParent --> MarkDirty
        
        MarkDirty --> Complete[Operation Complete]
    end
```

### Get Operation Detailed Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Bucket as Bucket
    participant Cursor as Cursor
    participant Node as Node
    participant Page as Page
    participant MMap as Memory Map
    
    App->>Bucket: bucket.Get("mykey")
    Bucket->>Cursor: Create cursor
    Cursor->>Cursor: seek("mykey")
    
    loop Tree traversal
        Cursor->>Node: Access current node
        alt Node not in cache
            Node->>Page: Read from page
            Page->>MMap: Access memory-mapped data
            MMap-->>Page: Raw page data
            Page->>Node: Deserialize page
            Node->>Node: Cache deserialized node
        end
        
        Node->>Node: Binary search for key
        alt Internal node
            Node->>Cursor: Navigate to child
        else Leaf node
            Node->>Node: Find exact key match
            alt Key found
                Node-->>Cursor: Return key/value
            else Key not found
                Node-->>Cursor: Return nil
            end
        end
    end
    
    Cursor-->>Bucket: Search result
    Bucket-->>App: Value or nil
```

### Delete Operation Flow

```mermaid
graph TD
    subgraph "Delete Operation"
        DeleteStart[Delete Key Request]
        FindKey[Locate Key in Leaf]
        RemoveEntry[Remove Key/Value Entry]
        CheckFillLevel[Check Node Fill Level]
        
        DeleteStart --> FindKey
        FindKey -->|Key Found| RemoveEntry
        FindKey -->|Key Not Found| NotFoundError[Return Not Found]
        
        RemoveEntry --> CheckFillLevel
        
        CheckFillLevel -->|Adequate Fill| MarkDirty[Mark Node Dirty]
        CheckFillLevel -->|Under-filled| Rebalance[Rebalance Node]
        
        Rebalance --> CheckSibling[Check Sibling Capacity]
        CheckSibling -->|Can Redistribute| RedistributeKeys[Redistribute Keys]
        CheckSibling -->|Cannot Redistribute| MergeNodes[Merge with Sibling]
        
        RedistributeKeys --> UpdateParent[Update Parent Node]
        MergeNodes --> UpdateParent
        UpdateParent --> MarkDirty
        
        MarkDirty --> Complete[Delete Complete]
    end
```

## Cursor Operations

### Cursor Iteration Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Cursor as Cursor
    participant Stack as Element Stack
    participant Node as B+tree Nodes
    participant Page as Page Data
    
    App->>Cursor: cursor.First()
    Cursor->>Stack: Clear navigation stack
    Cursor->>Node: Navigate to root
    
    loop Navigate to leftmost leaf
        Node->>Page: Read node page
        Page-->>Node: Node structure
        alt Internal node
            Node->>Stack: Push current position
            Node->>Node: Follow leftmost child
        else Leaf node
            Node->>Stack: Push leaf position
            Node-->>Cursor: First key/value
        end
    end
    
    Cursor-->>App: Return first item
    
    App->>Cursor: cursor.Next()
    Cursor->>Stack: Check current position
    
    alt More items in current leaf
        Stack->>Node: Advance to next item
        Node-->>Cursor: Next key/value
    else End of current leaf
        Stack->>Stack: Pop to parent level
        Stack->>Node: Find next sibling
        Node->>Node: Navigate to leftmost of sibling
        Node-->>Cursor: First item of next leaf
    end
    
    Cursor-->>App: Return next item
```

### Cursor Seek Operation

```mermaid
graph TD
    subgraph "Cursor Seek Flow"
        SeekStart[Seek to Key]
        ClearStack[Clear Element Stack]
        StartAtRoot[Start at Root Node]
        
        SeekStart --> ClearStack
        ClearStack --> StartAtRoot
        
        StartAtRoot --> BinarySearch[Binary Search in Node]
        BinarySearch --> FindPosition[Find Key Position]
        
        FindPosition -->|Internal Node| FollowChild[Follow Child Pointer]
        FindPosition -->|Leaf Node| LeafSearch[Search in Leaf]
        
        FollowChild --> PushStack[Push Position to Stack]
        PushStack --> BinarySearch
        
        LeafSearch -->|Exact Match| FoundKey[Return Exact Match]
        LeafSearch -->|Key Not Found| FoundPosition[Return Next Higher Key]
        
        FoundKey --> Complete[Seek Complete]
        FoundPosition --> Complete
    end
```

## Batch Operations

### Batch Write Operation

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant BatchMgr as Batch Manager
    participant Tx as Transaction
    participant Timer as Timer
    
    App->>DB: db.Batch(func(tx *bolt.Tx) error {...})
    DB->>BatchMgr: Add to batch queue
    
    par Batch accumulation
        DB->>Timer: Start batch timer
        and
        loop More batch requests
            App->>DB: db.Batch(...)
            DB->>BatchMgr: Add to batch queue
        end
    end
    
    alt Timer expires OR batch full
        Timer->>BatchMgr: Trigger batch execution
        and
        BatchMgr->>Tx: Begin write transaction
        
        loop For each batched operation
            BatchMgr->>Tx: Execute operation
            Tx->>Tx: Apply changes
        end
        
        Tx->>Tx: Commit all changes
        Tx-->>BatchMgr: Batch complete
        BatchMgr-->>App: All operations complete
    end
```

## Error Handling Flows

### Transaction Error Recovery

```mermaid
graph TD
    subgraph "Error Handling Flow"
        Operation[Database Operation]
        ErrorOccurred[Error Occurred]
        ErrorType{Error Type}
        
        Operation --> ErrorOccurred
        ErrorOccurred --> ErrorType
        
        ErrorType -->|Transaction Error| RollbackTx[Rollback Transaction]
        ErrorType -->|Page Error| MarkCorrupt[Mark Page Corrupt]
        ErrorType -->|Lock Error| ReleaseLocks[Release Locks]
        ErrorType -->|I/O Error| RetryOrFail[Retry or Fail]
        
        RollbackTx --> CleanupResources[Cleanup Resources]
        MarkCorrupt --> CleanupResources
        ReleaseLocks --> CleanupResources
        RetryOrFail --> CleanupResources
        
        CleanupResources --> ReturnError[Return Error to Application]
    end
```

### Consistency Check Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant Check as Consistency Checker
    participant Node as B+tree Nodes
    participant Page as Page Validator
    participant Report as Error Report
    
    App->>DB: db.Check()
    DB->>Check: Start consistency check
    
    Check->>Page: Validate all pages
    
    loop For each page
        Page->>Page: Check page header
        Page->>Page: Validate checksums
        Page->>Page: Check page references
        alt Page invalid
            Page->>Report: Add page error
        end
    end
    
    Check->>Node: Validate B+tree structure
    
    loop For each node
        Node->>Node: Check key ordering
        Node->>Node: Validate child pointers
        Node->>Node: Check fill levels
        alt Node invalid
            Node->>Report: Add tree error
        end
    end
    
    Check->>Report: Compile error report
    Report-->>App: Return check results
```

This comprehensive flow documentation shows how bbolt operations cascade through the system, providing insight into the interaction patterns between components and the data flow for both normal operations and error conditions.