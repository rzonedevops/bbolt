# bbolt Core Components

This document provides detailed information about bbolt's core components and their relationships.

## Component Overview

```mermaid
classDiagram
    class DB {
        +StrictMode bool
        +NoSync bool
        +ReadOnly bool
        +MmapFlags int
        +path string
        +file *os.File
        +dataref []byte
        +meta0 *meta
        +meta1 *meta
        +pageSize int
        +freelist *freelist
        +stats Stats
        +Open(path, mode, options) error
        +Close() error
        +Begin(writable) *Tx
        +View(func(*Tx)) error
        +Update(func(*Tx)) error
        +Batch(func(*Tx)) error
    }

    class Tx {
        +writable bool
        +managed bool
        +db *DB
        +meta *meta
        +root Bucket
        +pages map[pgid]*page
        +stats TxStats
        +Bucket(name) *Bucket
        +CreateBucket(name) *Bucket
        +DeleteBucket(name) error
        +Commit() error
        +Rollback() error
    }

    class Bucket {
        +tx *Tx
        +buckets map[string]*Bucket
        +page *page
        +rootNode *node
        +nodes map[pgid]*node
        +FillPercent float64
        +Get(key) []byte
        +Put(key, value) error
        +Delete(key) error
        +Cursor() *Cursor
        +CreateBucket(key) *Bucket
        +DeleteBucket(key) error
        +NextSequence() uint64
    }

    class Cursor {
        +bucket *Bucket
        +stack []elemRef
        +First() (key, value)
        +Last() (key, value)
        +Next() (key, value)
        +Prev() (key, value)
        +Seek(seek) (key, value)
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
    }

    class Page {
        +id pgid
        +flags uint16
        +count uint16
        +overflow uint32
        +ptr uintptr
    }

    DB ||--o{ Tx : manages
    Tx ||--o{ Bucket : contains
    Bucket ||--o{ Cursor : creates
    Bucket ||--o{ Node : manages
    Node ||--o{ Page : represents
    Bucket ||--|| Node : root
```

## 1. Database (DB)

The `DB` struct represents the entire database and is the main entry point for all operations.

### Key Responsibilities
- File management and memory mapping
- Transaction coordination
- Metadata management
- Resource cleanup

### Important Fields
```go
type DB struct {
    // Configuration flags
    StrictMode bool     // Enable consistency checks
    NoSync     bool     // Skip fsync for performance
    ReadOnly   bool     // Read-only mode
    
    // File system interface
    path     string     // Database file path
    file     *os.File   // Open file handle
    dataref  []byte     // Memory-mapped data
    
    // Database metadata
    meta0    *meta      // Primary metadata page
    meta1    *meta      // Secondary metadata page
    pageSize int        // Database page size
    
    // Resource management
    freelist *freelist  // Free page management
    stats    Stats      // Performance statistics
}
```

### Lifecycle Operations
```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Opening : Open()
    Opening --> Open : Success
    Opening --> Closed : Error
    Open --> Closing : Close()
    Closing --> Closed : Success
    
    state Open {
        [*] --> Idle
        Idle --> Reading : Begin(false)
        Idle --> Writing : Begin(true)
        Reading --> Idle : Commit/Rollback
        Writing --> Idle : Commit/Rollback
        
        state Writing {
            [*] --> Modifying
            Modifying --> Flushing : Commit()
            Flushing --> [*] : Success
        }
    }
```

## 2. Transaction (Tx)

Transactions provide ACID guarantees and manage the scope of database operations.

### Transaction Types
- **Read-only transactions**: Multiple concurrent readers
- **Read-write transactions**: Single writer, exclusive access

### Key Responsibilities
- Isolation and consistency
- Page management during transaction lifetime
- Bucket management
- Commit/rollback operations

### Transaction Lifecycle
```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant Tx as Transaction
    participant Meta as Metadata
    participant Pages as Page Manager
    
    App->>DB: Begin(writable)
    DB->>Tx: Create transaction
    Tx->>Meta: Read metadata
    Meta-->>Tx: Current state
    
    Note over Tx: Transaction active
    
    App->>Tx: Operations (Get/Put/Delete)
    Tx->>Pages: Manage dirty pages
    
    App->>Tx: Commit()
    alt Write Transaction
        Tx->>Pages: Write dirty pages
        Pages->>DB: Sync to disk
        Tx->>Meta: Update metadata
        Meta->>DB: Write new meta page
    else Read Transaction
        Tx->>DB: Release resources
    end
    
    Tx-->>DB: Transaction complete
    DB-->>App: Success
```

## 3. Bucket

Buckets are collections of key/value pairs, similar to tables in relational databases.

### Key Features
- Nested buckets (buckets within buckets)
- B+tree storage for efficient access
- Configurable fill percentage
- Auto-incrementing sequences

### Bucket Operations
```mermaid
graph TD
    subgraph "Bucket Structure"
        RootBucket[Root Bucket]
        SubBucket1[Sub Bucket 1]
        SubBucket2[Sub Bucket 2]
        
        KV1[Key/Value Pairs]
        KV2[Key/Value Pairs]
        KV3[Key/Value Pairs]
        
        RootBucket --> SubBucket1
        RootBucket --> SubBucket2
        SubBucket1 --> KV1
        SubBucket2 --> KV2
        RootBucket --> KV3
    end
    
    subgraph "Operations"
        Get[Get Key]
        Put[Put Key/Value]
        Delete[Delete Key]
        Cursor[Create Cursor]
        NextSeq[Next Sequence]
        
        Get --> BTree[B+Tree Lookup]
        Put --> BTree
        Delete --> BTree
        Cursor --> BTree
        NextSeq --> Meta[Metadata Update]
    end
```

### Bucket Tree Structure
```mermaid
graph TB
    subgraph "Physical Storage"
        MetaPage[Meta Page]
        RootPage[Root Page]
        LeafPage1[Leaf Page 1]
        LeafPage2[Leaf Page 2]
        LeafPage3[Leaf Page 3]
        
        MetaPage --> RootPage
        RootPage --> LeafPage1
        RootPage --> LeafPage2
        RootPage --> LeafPage3
    end
    
    subgraph "Logical View"
        RootBucket["Root Bucket<br/>accounts"]
        UsersBucket["Users Bucket"]
        SettingsBucket["Settings Bucket"]
        
        User1["User 1<br/>Key: 'john'"]
        User2["User 2<br/>Key: 'jane'"]
        Setting1["Theme: 'dark'"]
        Setting2["Lang: 'en'"]
        
        RootBucket --> UsersBucket
        RootBucket --> SettingsBucket
        UsersBucket --> User1
        UsersBucket --> User2
        SettingsBucket --> Setting1
        SettingsBucket --> Setting2
    end
```

## 4. Cursor

Cursors provide efficient iteration over key/value pairs in lexicographical order.

### Cursor Operations
- Sequential traversal (First, Last, Next, Prev)
- Seeking to specific keys
- Range scans and prefix searches

### Cursor Implementation
```mermaid
graph TD
    subgraph "Cursor State"
        Stack[Element Stack]
        Position[Current Position]
        Bucket[Target Bucket]
    end
    
    subgraph "Navigation Operations"
        First[First Key]
        Last[Last Key]
        Next[Next Key]
        Prev[Previous Key]
        Seek[Seek to Key]
    end
    
    subgraph "B+Tree Traversal"
        Root[Root Node]
        Internal[Internal Nodes]
        Leaf[Leaf Nodes]
        
        Root --> Internal
        Internal --> Leaf
    end
    
    Stack --> Position
    Position --> Navigation
    Navigation --> BTreeTraversal
    
    First --> Root
    Last --> Root
    Next --> Internal
    Prev --> Internal
    Seek --> Root
```

### Cursor Traversal Example
```mermaid
sequenceDiagram
    participant App as Application
    participant Cursor as Cursor
    participant Stack as Element Stack
    participant BTree as B+Tree
    participant Node as Tree Nodes
    
    App->>Cursor: First()
    Cursor->>Stack: Clear stack
    Cursor->>BTree: Navigate to leftmost
    BTree->>Node: Find first leaf
    Node-->>BTree: Leftmost key/value
    BTree-->>Cursor: Key/value pair
    Cursor-->>App: Return first item
    
    App->>Cursor: Next()
    Cursor->>Stack: Check current position
    Stack-->>Cursor: Current element
    Cursor->>Node: Move to next item
    alt More items in node
        Node-->>Cursor: Next key/value
    else End of node
        Cursor->>BTree: Navigate to next node
        BTree->>Node: Next leaf node
        Node-->>Cursor: First item in next node
    end
    Cursor-->>App: Return next item
```

## 5. Node

Nodes represent in-memory, deserialized pages from the B+tree structure.

### Node Types
- **Leaf nodes**: Contain actual key/value pairs
- **Internal nodes**: Contain keys and pointers to child nodes

### Node Management
```mermaid
graph TD
    subgraph "Node Lifecycle"
        DiskPage[Page on Disk]
        MemoryNode[Node in Memory]
        DirtyNode[Dirty Node]
        NewPage[New Page on Disk]
        
        DiskPage -->|Deserialize| MemoryNode
        MemoryNode -->|Modify| DirtyNode
        DirtyNode -->|Serialize| NewPage
    end
    
    subgraph "Node Operations"
        Split[Split Node]
        Merge[Merge Nodes]
        Rebalance[Rebalance]
        
        Split --> Create[Create New Node]
        Merge --> Delete[Delete Node]
        Rebalance --> Redistribute[Redistribute Keys]
    end
```

### B+Tree Node Structure
```mermaid
graph TD
    subgraph "Internal Node"
        IKeys["Keys: [10, 20, 30]"]
        IPointers["Pointers: [P1, P2, P3, P4]"]
        
        IKeys -.-> IPointers
    end
    
    subgraph "Leaf Nodes"
        L1["Leaf 1<br/>Keys: [5, 8]<br/>Values: [V5, V8]"]
        L2["Leaf 2<br/>Keys: [12, 15]<br/>Values: [V12, V15]"]
        L3["Leaf 3<br/>Keys: [22, 25]<br/>Values: [V22, V25]"]
        L4["Leaf 4<br/>Keys: [35, 40]<br/>Values: [V35, V40]"]
        
        L1 -.-> L2
        L2 -.-> L3
        L3 -.-> L4
    end
    
    IPointers --> L1
    IPointers --> L2
    IPointers --> L3
    IPointers --> L4
```

## 6. Page

Pages represent the physical storage units on disk, typically 4KB in size.

### Page Types
- **Meta pages**: Database metadata (2 copies for redundancy)
- **Freelist pages**: Track available pages
- **Branch pages**: Internal B+tree nodes
- **Leaf pages**: B+tree leaf nodes

### Page Layout
```mermaid
graph TD
    subgraph "Database File Layout"
        Meta0[Meta Page 0]
        Meta1[Meta Page 1]
        Freelist[Freelist Page]
        Branch1[Branch Page]
        Leaf1[Leaf Page 1]
        Leaf2[Leaf Page 2]
        Leaf3[Leaf Page 3]
        Free1[Free Page]
        Leaf4[Leaf Page 4]
        
        Meta0 -.->|Primary| DBInfo[Database Info]
        Meta1 -.->|Backup| DBInfo
        Freelist -.-> Free1
    end
    
    subgraph "Page Structure"
        Header[Page Header]
        Data[Page Data]
        
        Header --> PageID[Page ID]
        Header --> Flags[Page Flags]
        Header --> Count[Element Count]
        Header --> Overflow[Overflow Pages]
        
        Data --> Elements[Key/Value Elements]
    end
```

## Component Interactions

The following diagram shows how components interact during a typical Put operation:

```mermaid
sequenceDiagram
    participant App as Application
    participant Tx as Transaction
    participant Bucket as Bucket
    participant Node as Node
    participant Page as Page
    participant DB as Database
    
    App->>Tx: bucket.Put(key, value)
    Tx->>Bucket: Put operation
    Bucket->>Node: Find/create leaf node
    Node->>Node: Insert key/value
    
    alt Node needs split
        Node->>Node: Split into two nodes
        Node->>Bucket: Update parent references
        Bucket->>Node: Rebalance tree
    end
    
    Tx->>Node: Mark nodes as dirty
    
    App->>Tx: Commit()
    Tx->>Node: Serialize dirty nodes
    Node->>Page: Create new pages
    Page->>DB: Write pages to disk
    DB->>DB: Update metadata
    DB-->>App: Commit successful
```

This component architecture provides a clean separation of concerns while maintaining high performance through efficient data structures and minimal copying operations.