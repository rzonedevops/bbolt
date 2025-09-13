# bbolt Transaction Management

bbolt implements ACID transactions with Multi-Version Concurrency Control (MVCC) to provide consistent, isolated access to the database. This document details the transaction lifecycle, concurrency model, and isolation guarantees.

## Transaction Overview

bbolt supports two types of transactions:
- **Read-only transactions**: Multiple concurrent readers with snapshot isolation
- **Read-write transactions**: Single writer with exclusive access

```mermaid
graph TB
    subgraph "Transaction Types"
        subgraph "Read Transactions"
            R1[Reader 1]
            R2[Reader 2]
            R3[Reader N]
        end
        
        subgraph "Write Transaction"
            W1[Single Writer]
        end
        
        subgraph "Database State"
            Snapshot1[Snapshot T1]
            Snapshot2[Snapshot T2]
            Current[Current State]
        end
        
        R1 --> Snapshot1
        R2 --> Snapshot2
        R3 --> Snapshot2
        W1 --> Current
        
        Current -.->|New Snapshot| Snapshot3[Snapshot T3]
    end
```

## MVCC (Multi-Version Concurrency Control)

bbolt's MVCC implementation allows multiple readers to access consistent snapshots while a single writer can modify the database without blocking readers.

### MVCC Architecture

```mermaid
graph TD
    subgraph "MVCC Timeline"
        T0[Time T0<br/>Initial State]
        T1[Time T1<br/>Reader Tx Starts]
        T2[Time T2<br/>Writer Tx Starts]
        T3[Time T3<br/>Writer Commits]
        T4[Time T4<br/>Reader Still Sees T1]
        T5[Time T5<br/>New Reader Sees T3]
        
        T0 --> T1
        T1 --> T2
        T2 --> T3
        T3 --> T4
        T4 --> T5
    end
    
    subgraph "Database Versions"
        V0[Version 0<br/>Pages: A, B, C]
        V1[Version 1<br/>Pages: A', B, C]
        
        V0 -.->|Copy-on-Write| V1
    end
    
    subgraph "Reader Isolation"
        Reader[Long Reader]
        Writer[Writer Tx]
        NewReader[New Reader]
        
        Reader -.->|Sees| V0
        Writer -.->|Modifies| V1
        NewReader -.->|Sees| V1
    end
```

### Copy-on-Write Mechanism

When a write transaction modifies a page, bbolt creates a new copy rather than modifying the original:

```mermaid
sequenceDiagram
    participant Writer as Write Transaction
    participant Page as Original Page
    participant NewPage as New Page Copy
    participant Meta as Metadata
    participant Readers as Read Transactions
    
    Note over Readers: Reading from original pages
    
    Writer->>Page: Request modification
    Writer->>NewPage: Create copy
    Writer->>NewPage: Apply modifications
    Writer->>Meta: Update page references
    
    Note over Readers: Still reading original pages
    Note over Writer: Writing to new pages
    
    Writer->>Meta: Commit transaction
    Meta->>Meta: Switch to new page layout
    
    Note over Readers: New readers see new pages
    Note over Readers: Old readers keep original pages
```

## Transaction Lifecycle

### Read-Only Transaction Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created
    Created --> Active : Begin(false)
    Active --> Reading : Operations
    Reading --> Active : Continue
    Active --> Committed : Rollback() or function return
    Committed --> [*]
    
    state Active {
        [*] --> BucketAccess
        BucketAccess --> CursorOps : Cursor()
        CursorOps --> BucketAccess : More operations
        BucketAccess --> KeyValueOps : Get()
        KeyValueOps --> BucketAccess : More operations
    }
```

### Read-Write Transaction Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created
    Created --> Acquiring : Begin(true)
    Acquiring --> Active : Acquire write lock
    Acquiring --> Failed : Lock timeout
    Failed --> [*]
    
    Active --> Modifying : Mutations
    Modifying --> Active : Continue
    Active --> Committing : Commit()
    Active --> RollingBack : Rollback()
    
    Committing --> Flushing : Write dirty pages
    Flushing --> MetaUpdate : Update metadata
    MetaUpdate --> Sync : fsync()
    Sync --> Committed : Release lock
    
    RollingBack --> Committed : Discard changes
    Committed --> [*]
    
    state Active {
        [*] --> Reading
        Reading --> Writing : Put/Delete
        Writing --> Reading : More operations
        Writing --> BucketOps : Create/Delete Bucket
        BucketOps --> Writing : More operations
    }
```

## Concurrency Control

### Lock Management

bbolt uses a combination of locks to manage concurrency:

```mermaid
graph TD
    subgraph "Lock Hierarchy"
        DBLock[Database File Lock]
        MetaLock[Metadata Lock]
        WriteLock[Write Transaction Lock]
        
        DBLock --> MetaLock
        MetaLock --> WriteLock
    end
    
    subgraph "Lock Types"
        FileLock[File Lock<br/>Process-level exclusion]
        RWMutex[Read-Write Mutex<br/>Thread-level coordination]
        
        FileLock -.-> DBLock
        RWMutex -.-> WriteLock
    end
    
    subgraph "Concurrency Rules"
        OneWriter[Single Writer]
        MultiReader[Multiple Readers]
        NoDeadlock[No Reader-Writer Deadlock]
        
        OneWriter --> WriteLock
        MultiReader --> MetaLock
        NoDeadlock --> RWMutex
    end
```

### Transaction Coordination

```mermaid
sequenceDiagram
    participant R1 as Reader 1
    participant R2 as Reader 2
    participant W as Writer
    participant DB as Database
    participant Meta as Metadata
    participant Lock as Lock Manager
    
    R1->>DB: Begin(false)
    DB->>Meta: Read metadata
    Meta-->>R1: Snapshot at T1
    
    R2->>DB: Begin(false)
    DB->>Meta: Read metadata
    Meta-->>R2: Snapshot at T1
    
    W->>DB: Begin(true)
    DB->>Lock: Acquire write lock
    Lock-->>W: Lock acquired
    DB->>Meta: Read metadata
    Meta-->>W: Current state
    
    Note over R1,R2: Readers continue with T1 snapshot
    Note over W: Writer modifies current state
    
    W->>DB: Commit()
    DB->>Meta: Write new metadata
    DB->>Lock: Release write lock
    
    Note over R1,R2: Readers still see T1
    
    participant R3 as Reader 3
    R3->>DB: Begin(false)
    DB->>Meta: Read metadata
    Meta-->>R3: Snapshot at T2 (after write)
```

## Isolation Levels

bbolt provides **Serializable** isolation, the strongest isolation level, through its MVCC implementation.

### Isolation Guarantees

```mermaid
graph TD
    subgraph "ACID Properties"
        Atomicity[Atomicity<br/>All or nothing]
        Consistency[Consistency<br/>Valid state transitions]
        Isolation[Isolation<br/>Concurrent transaction independence]
        Durability[Durability<br/>Committed changes persist]
    end
    
    subgraph "Isolation Implementation"
        Snapshot[Snapshot Isolation]
        NoPhantom[No Phantom Reads]
        NoRepeat[No Non-repeatable Reads]
        NoDirty[No Dirty Reads]
        
        Snapshot --> NoPhantom
        Snapshot --> NoRepeat
        Snapshot --> NoDirty
    end
    
    subgraph "Mechanisms"
        MVCC[Multi-Version<br/>Concurrency Control]
        COW[Copy-on-Write]
        VersionedPages[Versioned Pages]
        
        MVCC --> COW
        COW --> VersionedPages
    end
    
    Isolation --> Snapshot
    Snapshot --> MVCC
```

### Read Phenomena Prevention

| Phenomenon | bbolt Prevention |
|------------|------------------|
| **Dirty Read** | ✅ Prevented - Readers see only committed data |
| **Non-repeatable Read** | ✅ Prevented - Snapshot isolation maintains consistency |
| **Phantom Read** | ✅ Prevented - Consistent view throughout transaction |
| **Lost Update** | ✅ Prevented - Single writer prevents write conflicts |

## Transaction Implementation Details

### Transaction State Management

```mermaid
graph TD
    subgraph "Transaction Structure"
        TxStruct[Tx Struct]
        WritableFlag[writable bool]
        ManagedFlag[managed bool]
        DBRef[db *DB]
        MetaRef[meta *meta]
        RootBucket[root Bucket]
        PagesMap[pages map]
        StatsStruct[stats TxStats]
    end
    
    subgraph "State Tracking"
        Clean[Clean State]
        Dirty[Dirty Pages]
        Committed[Committed]
        RolledBack[Rolled Back]
        
        Clean --> Dirty
        Dirty --> Committed
        Dirty --> RolledBack
    end
    
    TxStruct --> WritableFlag
    TxStruct --> PagesMap
    PagesMap --> Dirty
```

### Page Management During Transactions

```mermaid
sequenceDiagram
    participant Tx as Transaction
    participant Pages as Page Map
    participant Node as Node
    participant Freelist as Freelist
    participant Disk as Disk Storage
    
    Tx->>Node: Modify data
    Node->>Node: Mark as dirty
    Node->>Pages: Track dirty page
    Pages->>Pages: Add to dirty set
    
    Tx->>Tx: Commit()
    Tx->>Pages: Get all dirty pages
    Pages-->>Tx: Dirty page list
    
    loop For each dirty page
        Tx->>Freelist: Allocate new page ID
        Freelist-->>Tx: New page ID
        Tx->>Disk: Write page with new ID
    end
    
    Tx->>Freelist: Mark old pages as free
    Tx->>Disk: Update metadata
    Tx->>Disk: fsync()
```

### Metadata Management

bbolt maintains two copies of metadata for crash consistency:

```mermaid
graph TD
    subgraph "Dual Metadata System"
        Meta0[Meta Page 0<br/>Transaction ID: N]
        Meta1[Meta Page 1<br/>Transaction ID: N-1]
        
        Meta0 -.->|Primary| Current[Current State]
        Meta1 -.->|Backup| Previous[Previous State]
    end
    
    subgraph "Update Process"
        Write1[Write to backup meta]
        Sync1[fsync backup]
        Update2[Update primary meta]
        Sync2[fsync primary]
        
        Write1 --> Sync1
        Sync1 --> Update2
        Update2 --> Sync2
    end
    
    subgraph "Recovery"
        CheckMeta0[Check Meta 0]
        CheckMeta1[Check Meta 1]
        UseValid[Use Valid Meta]
        
        CheckMeta0 --> UseValid
        CheckMeta1 --> UseValid
    end
```

## Performance Considerations

### Transaction Performance Characteristics

```mermaid
graph TD
    subgraph "Read Performance"
        MultiReader[Multiple Readers]
        NoLocking[No Lock Contention]
        MemoryMap[Memory-Mapped Access]
        
        MultiReader --> NoLocking
        NoLocking --> MemoryMap
    end
    
    subgraph "Write Performance"
        SingleWriter[Single Writer]
        BatchWrites[Batch Operations]
        COWOverhead[Copy-on-Write Cost]
        
        SingleWriter --> BatchWrites
        BatchWrites --> COWOverhead
    end
    
    subgraph "Optimization Strategies"
        BatchAPI[DB.Batch() API]
        NoSyncMode[NoSync Mode]
        ReadOnlyTx[Read-Only When Possible]
        
        BatchAPI --> BatchWrites
        NoSyncMode --> WritePerf[Improved Write Performance]
        ReadOnlyTx --> MultiReader
    end
```

### Best Practices

1. **Use Read-Only Transactions When Possible**
   ```go
   // Prefer View() for read operations
   db.View(func(tx *bolt.Tx) error {
       // Read operations only
       return nil
   })
   ```

2. **Batch Write Operations**
   ```go
   // Use Batch() for bulk writes
   db.Batch(func(tx *bolt.Tx) error {
       // Multiple writes batched together
       return nil
   })
   ```

3. **Keep Transactions Short**
   ```go
   // Avoid long-running read transactions
   // They prevent page reclamation
   ```

4. **Handle Transaction Errors**
   ```go
   tx, err := db.Begin(true)
   if err != nil {
       return err
   }
   defer tx.Rollback() // Always rollback on error
   
   // ... operations ...
   
   return tx.Commit() // Explicit commit
   ```

This transaction management system provides strong consistency guarantees while allowing high concurrency for read operations, making bbolt suitable for applications that need reliable data storage with good read performance.