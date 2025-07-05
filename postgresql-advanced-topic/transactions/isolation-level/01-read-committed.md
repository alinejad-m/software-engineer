# Read Committed Isolation Level in PostgreSQL

## Overview

Read Committed is the default isolation level in PostgreSQL. In this isolation level:
- Each query sees only data committed before the query began
- A query never sees uncommitted data or changes committed during query execution by concurrent transactions
- Two successive queries in the same transaction might see different data

## Visual Representation

```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant DB as Database
    participant T2 as Transaction 2
    
    Note over T1,T2: Initial value: account_balance = 1000
    
    T1->>DB: BEGIN
    T2->>DB: BEGIN
    
    T1->>DB: SELECT account_balance<br/>(reads 1000)
    
    T2->>DB: UPDATE account SET balance = 800
    T2->>DB: COMMIT
    
    Note over DB: Balance is now 800
    
    T1->>DB: SELECT account_balance<br/>(reads 800)
    Note right of T1: Second read sees<br/>new committed value
    
    T1->>DB: COMMIT
```

## When to Use Read Committed

### Ideal Use Cases
1. **General Purpose Applications**
   - Default choice for most applications
   - Balanced between consistency and performance
   - Suitable for OLTP systems

2. **Multi-user Applications**
   - Web applications
   - Customer portals
   - Business applications

3. **Reporting Systems**
   - Real-time dashboards
   - Live data monitoring
   - Current state queries

### Common Scenarios
1. **Banking Transactions**
```sql
BEGIN;
-- Check account balance (sees committed data only)
SELECT balance FROM accounts WHERE id = 123;
-- Process withdrawal
UPDATE accounts SET balance = balance - 100 WHERE id = 123;
COMMIT;
```

2. **Inventory Management**
```sql
BEGIN;
-- Check current stock (sees committed data only)
SELECT quantity FROM inventory WHERE product_id = 456;
-- Update stock level
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 456;
COMMIT;
```

## How It Works

### Key Characteristics

```mermaid
graph TB
    subgraph "Read Committed Behavior"
        A[Query Starts] --> B{Sees What?}
        B -->|Yes| C[Committed Before Query]
        B -->|No| D[Uncommitted Data]
        B -->|No| E[Changes During Query]
        
        style A fill:#f9f,stroke:#333,stroke-width:2px,color:black
        style B fill:#bbf,stroke:#333,stroke-width:2px,color:black
        style C fill:#dfd,stroke:#333,stroke-width:2px,color:black
        style D fill:#fdd,stroke:#333,stroke-width:2px,color:black
        style E fill:#fdd,stroke:#333,stroke-width:2px,color:black
    end
```

1. **Read Rules**
   - Only sees data committed before query start
   - Never sees uncommitted changes
   - Never sees changes committed during query execution

2. **Write Rules**
   - Writers don't block readers
   - Readers don't block writers
   - Write locks held until commit

## Advantages and Disadvantages

### Pros
1. **Performance**
   - Good concurrency
   - Minimal locking overhead
   - Efficient for read-heavy workloads

2. **Data Consistency**
   - No dirty reads
   - Committed data is always consistent
   - Suitable for most business applications

3. **Usability**
   - Default level
   - Well understood
   - Predictable behavior

### Cons
1. **Non-Repeatable Reads**
```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant DB as Database
    participant T2 as Transaction 2
    
    T1->>DB: BEGIN
    T2->>DB: BEGIN
    
    T1->>DB: SELECT price FROM products<br/>WHERE id = 1 (returns $10)
    
    T2->>DB: UPDATE products<br/>SET price = $15<br/>WHERE id = 1
    T2->>DB: COMMIT
    
    T1->>DB: SELECT price FROM products<br/>WHERE id = 1 (returns $15)
    Note right of T1: Same query returns<br/>different results
    
    T1->>DB: COMMIT
```

2. **Phantom Reads**
```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant DB as Database
    participant T2 as Transaction 2
    
    T1->>DB: BEGIN
    T2->>DB: BEGIN
    
    T1->>DB: SELECT * FROM products<br/>WHERE price < 100<br/>(returns 5 rows)
    
    T2->>DB: INSERT INTO products<br/>(id, price) VALUES (6, 50)
    T2->>DB: COMMIT
    
    T1->>DB: SELECT * FROM products<br/>WHERE price < 100<br/>(returns 6 rows)
    Note right of T1: New row appears<br/>in result set
    
    T1->>DB: COMMIT
```

## Best Practices

1. **Transaction Design**
   - Keep transactions short
   - Minimize the number of statements
   - Avoid long-running transactions

2. **Error Handling**
```sql
BEGIN;
SAVEPOINT my_savepoint;

-- Attempt operation
UPDATE accounts SET balance = balance - 100 WHERE id = 123;

-- Check if sufficient funds
IF NOT FOUND THEN
    ROLLBACK TO my_savepoint;
    -- Handle insufficient funds
ELSE
    COMMIT;
END IF;
```

3. **Performance Optimization**
   - Use indexes effectively
   - Monitor lock contention
   - Consider batch processing for large operations

## Common Issues and Solutions

1. **Lost Updates**
   Problem: Two transactions updating the same row might lose changes.
   
   Solution:
   ```sql
   BEGIN;
   -- Use SELECT FOR UPDATE to lock the row
   SELECT balance FROM accounts 
   WHERE id = 123 FOR UPDATE;
   
   -- Now safe to update
   UPDATE accounts 
   SET balance = balance - 100 
   WHERE id = 123;
   COMMIT;
   ```

2. **Inconsistent Analysis**
   Problem: Long-running queries might see inconsistent state.
   
   Solution:
   ```sql
   -- Use snapshot export/import for consistent reads
   BEGIN;
   SET TRANSACTION SNAPSHOT transaction_snapshot;
   -- Run analysis queries
   COMMIT;
   ```

## Monitoring and Troubleshooting

1. **Check Transaction State**
```sql
SELECT pid, 
       usename, 
       application_name,
       state,
       query_start,
       wait_event_type
FROM pg_stat_activity
WHERE state = 'active';
```

2. **Monitor Lock Contention**
```sql
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype;
```

## Comparison with Other Isolation Levels

| Feature | Read Committed | Repeatable Read | Serializable |
|---------|---------------|-----------------|--------------|
| Dirty Reads | Prevented | Prevented | Prevented |
| Non-repeatable Reads | Possible | Prevented | Prevented |
| Phantom Reads | Possible | Possible | Prevented |
| Performance | Best | Good | Lowest |
| Concurrency | Highest | Medium | Lowest | 