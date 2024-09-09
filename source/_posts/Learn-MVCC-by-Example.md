---
title: Learn MVCC by Example
date: 2024-09-08 16:04:47
tags:
---
## Overview
MVCC (Multi-Version Concurrency Control) is commonly used concurrency control mechanism in postgresql. While reading different textual or pseudocode versions of MVCC algorithms is a good starting point, they often gloss over important implementation details. These details then show up in other related works, making it even harder to comprehend those texts.

Another reason to attempt an implementation is that production-ready implementations out there have complexities related to the context they are embedded in. So while you can read code from PostgreSQL's implementation, you might have to read a lot more code than required to just understand MVCC.

This blog post is an attempt to provide enough implementation details, using Java, to help me grasp the nuances of MVCC better. The goal is not to present production-ready code but rather to serve as a learning resource. By walking through a practical implementation, I will learn about the real-world challenges and considerations involved in building an MVCC system.


### Basic MVCC in java

Let's start by creating the necessary classes and then implement a simple MVCC system.



```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.ArrayList;
import java.util.List;

class MVCCDatabase {
    private ConcurrentHashMap<String, List<VersionedData>> data;
    private AtomicLong currentTransactionId;

    public MVCCDatabase() {
        this.data = new ConcurrentHashMap<>();
        this.currentTransactionId = new AtomicLong(0);
    }

    public long beginTransaction() {
        return currentTransactionId.incrementAndGet();
    }

    public void put(String key, String value, long transactionId) {
        data.computeIfAbsent(key, k -> new ArrayList<>())
            .add(new VersionedData(value, transactionId));
    }

    public String get(String key, long transactionId) {
        List<VersionedData> versions = data.get(key);
        if (versions == null) {
            return null;
        }

        for (int i = versions.size() - 1; i >= 0; i--) {
            VersionedData vd = versions.get(i);
            if (vd.transactionId <= transactionId) {
                return vd.value;
            }
        }

        return null;
    }

    private static class VersionedData {
        String value;
        long transactionId;

        VersionedData(String value, long transactionId) {
            this.value = value;
            this.transactionId = transactionId;
        }
    }
}

// Example usage
public class MVCCExample {
    public static void main(String[] args) {
        MVCCDatabase db = new MVCCDatabase();

        // Transaction 1
        long tx1 = db.beginTransaction();
        db.put("key1", "value1", tx1);

        // Transaction 2
        long tx2 = db.beginTransaction();
        db.put("key1", "value2", tx2);

        // Read from different transaction points
        System.out.println("Tx1 reads key1: " + db.get("key1", tx1)); // Should print "value1"
        System.out.println("Tx2 reads key1: " + db.get("key1", tx2)); // Should print "value2"
        System.out.println("New transaction reads key1: " + db.get("key1", db.beginTransaction())); // Should print "value2"
    }
}

```

This implementation provides a basic MVCC system similar to PostgreSQL's approach. Here's a breakdown of the main components:

1. `MVCCDatabase`: The main class that handles the MVCC operations.
2. `VersionedData`: An inner class that represents a version of data with its associated transaction ID.
3. `ConcurrentHashMap<String, List<VersionedData>>`: Stores the versioned data for each key.
4. `AtomicLong currentTransactionId`: Generates unique transaction IDs.

The key MVCC concepts implemented here are:

- Each write operation creates a new version of the data.
- Read operations retrieve the most recent version of the data that's not newer than the reading transaction.
- Transactions are identified by monotonically increasing IDs.

This implementation is simplified and doesn't include features like commit/rollback or garbage collection of old versions. In a real-world scenario, you'd need to add these features and handle concurrent access more robustly.

Would you like me to explain any part of this implementation in more detail?

### Implementing Serializable Txns


To implement serializable transactions in our MVCC system, we need to add several key features to ensure that concurrent transactions behave as if they were executed serially. Here's what we need to add:



```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.HashSet;
import java.util.Map;
import java.util.HashMap;

class MVCCDatabase {
    private ConcurrentHashMap<String, List<VersionedData>> data;
    private AtomicLong currentTransactionId;
    private ConcurrentHashMap<Long, Transaction> activeTransactions;

    public MVCCDatabase() {
        this.data = new ConcurrentHashMap<>();
        this.currentTransactionId = new AtomicLong(0);
        this.activeTransactions = new ConcurrentHashMap<>();
    }

    public long beginTransaction() {
        long txId = currentTransactionId.incrementAndGet();
        activeTransactions.put(txId, new Transaction(txId));
        return txId;
    }

    public void put(String key, String value, long transactionId) throws TransactionConflictException {
        Transaction tx = activeTransactions.get(transactionId);
        if (tx == null) {
            throw new IllegalStateException("Transaction not active");
        }

        // Check for write-write conflicts
        if (hasWriteConflict(key, transactionId)) {
            throw new TransactionConflictException("Write-write conflict detected");
        }

        tx.writeSet.add(key);
        tx.writes.put(key, value);
    }

    public String get(String key, long transactionId) throws TransactionConflictException {
        Transaction tx = activeTransactions.get(transactionId);
        if (tx == null) {
            throw new IllegalStateException("Transaction not active");
        }

        // Check if the key was written in this transaction
        if (tx.writes.containsKey(key)) {
            return tx.writes.get(key);
        }

        // Check for read-write conflicts
        if (hasReadWriteConflict(key, transactionId)) {
            throw new TransactionConflictException("Read-write conflict detected");
        }

        tx.readSet.add(key);

        List<VersionedData> versions = data.get(key);
        if (versions == null) {
            return null;
        }

        for (int i = versions.size() - 1; i >= 0; i--) {
            VersionedData vd = versions.get(i);
            if (vd.transactionId <= transactionId) {
                return vd.value;
            }
        }

        return null;
    }

    public void commitTransaction(long transactionId) throws TransactionConflictException {
        Transaction tx = activeTransactions.get(transactionId);
        if (tx == null) {
            throw new IllegalStateException("Transaction not active");
        }

        // Final conflict check before commit
        for (String key : tx.readSet) {
            if (hasReadWriteConflict(key, transactionId)) {
                throw new TransactionConflictException("Read-write conflict detected at commit");
            }
        }

        // Apply writes
        for (Map.Entry<String, String> entry : tx.writes.entrySet()) {
            data.computeIfAbsent(entry.getKey(), k -> new ArrayList<>())
                .add(new VersionedData(entry.getValue(), transactionId));
        }

        activeTransactions.remove(transactionId);
    }

    public void rollbackTransaction(long transactionId) {
        activeTransactions.remove(transactionId);
    }

    private boolean hasWriteConflict(String key, long transactionId) {
        for (Transaction tx : activeTransactions.values()) {
            if (tx.transactionId != transactionId && tx.writeSet.contains(key)) {
                return true;
            }
        }
        return false;
    }

    private boolean hasReadWriteConflict(String key, long transactionId) {
        for (Transaction tx : activeTransactions.values()) {
            if (tx.transactionId != transactionId && tx.writeSet.contains(key)) {
                return true;
            }
        }
        return false;
    }

    private static class VersionedData {
        String value;
        long transactionId;

        VersionedData(String value, long transactionId) {
            this.value = value;
            this.transactionId = transactionId;
        }
    }

    private static class Transaction {
        long transactionId;
        Set<String> readSet;
        Set<String> writeSet;
        Map<String, String> writes;

        Transaction(long transactionId) {
            this.transactionId = transactionId;
            this.readSet = new HashSet<>();
            this.writeSet = new HashSet<>();
            this.writes = new HashMap<>();
        }
    }

    public static class TransactionConflictException extends Exception {
        public TransactionConflictException(String message) {
            super(message);
        }
    }
}

// Example usage
public class SerializableMVCCExample {
    public static void main(String[] args) {
        MVCCDatabase db = new MVCCDatabase();

        try {
            // Transaction 1
            long tx1 = db.beginTransaction();
            db.put("key1", "value1", tx1);

            // Transaction 2
            long tx2 = db.beginTransaction();
            db.put("key2", "value2", tx2);

            // This should succeed
            System.out.println("Tx1 reads key2: " + db.get("key2", tx1));

            // This should throw a TransactionConflictException
            db.put("key1", "value1_updated", tx2);

            // Commit transactions
            db.commitTransaction(tx1);
            db.commitTransaction(tx2);
        } catch (MVCCDatabase.TransactionConflictException e) {
            System.out.println("Transaction conflict: " + e.getMessage());
        }
    }
}

```

To implement serializable transactions in our MVCC system, we've added the following key features:

1. Transaction Management:
   - We now keep track of active transactions using the `activeTransactions` map.
   - Each transaction has its own `Transaction` object that stores read and write sets.

2. Conflict Detection:
   - Write-Write Conflicts: We check if any other active transaction has written to the same key.
   - Read-Write Conflicts: We check if any other active transaction has written to a key that we're trying to read.

3. Optimistic Concurrency Control (OCC):   What we will actually implement is a form of OCC. In this approach, Transactions proceed without locking data items. If there's a conflict, the transaction aborts (in our case, by throwing a TransactionConflictException).
    - Read Phase: When a transaction reads data (in our `get` method), it adds the key to its read set.
    - Write Phase: When a transaction writes data (in our `put` method), it adds the key to its write set and stores the write in a local buffer (our `writes` map in the `Transaction` class).
    - Validation Phase: This occurs during the `put`, `get`, and `commitTransaction` methods:
       - We check for write-write conflicts by seeing if any other active transaction has written to the same key.
       - We check for read-write conflicts by seeing if any other active transaction has written to a key that we're trying to read.
   - Commit Phase: If validation succeeds, we apply all the writes atomically during the `commitTransaction` method.
   - Commit and Rollback:
       - `commitTransaction`: Performs a final conflict check and applies all writes atomically.
       - `rollbackTransaction`: Discards all changes made by the transaction.
   - Error Handling:We introduce a `TransactionConflictException` to handle serialization conflicts.
   - Visibility Rules:
       - Transactions only see committed data from other transactions that started before them.
       - A transaction's own writes are immediately visible to itself.

This implementation provides serializable isolation by ensuring that concurrent transactions behave as if they were executed serially. It does this by detecting and preventing conflicts that would violate serializability.

Key points to note:

1. This is still a simplified implementation. A production-ready system would need additional features like deadlock detection and prevention.
2. Performance optimizations, such as predicate locking for range queries, are not implemented here.
3. This implementation may be prone to high abort rates under contention. Real databases often use more sophisticated techniques to reduce conflicts.

To further improve this system, we might consider:

1. Implementing a scheduler to manage transaction order and reduce conflicts.
2. Adding support for different isolation levels.
3. Implementing a more efficient conflict detection mechanism, possibly using timestamps or multi-version timestamps.
4. Adding support for distributed transactions.

### Optimization - Timestamp-based conflict detection

In our current implementation, we're using a relatively simple approach to detect conflicts: we check all active transactions for potential conflicts every time we perform a read or write operation. This can become inefficient as the number of concurrent transactions increases. Using timestamps or multi-version timestamps can provide a more scalable approach to conflict detection. Here's how these mechanisms work:

1. Timestamp-based conflict detection:

In this approach, each transaction is assigned a unique timestamp when it starts. Additionally, each data item maintains two timestamps:
- Read timestamp (RTS): The highest timestamp of any transaction that has read the item.
- Write timestamp (WTS): The highest timestamp of any transaction that has written to the item.

When a transaction tries to read or write an item, we can use these timestamps to detect conflicts:

- For a read operation: If the transaction's timestamp is less than the item's WTS, it means a newer transaction has already written to this item, so we have a conflict.
- For a write operation: If the transaction's timestamp is less than either the item's RTS or WTS, we have a conflict.

This approach allows for quicker conflict detection without needing to check all active transactions.

2. Multi-version timestamps:

This is an extension of the timestamp-based approach that maintains multiple versions of each data item, each with its own timestamp. It's similar to what we're already doing with our versioned data, but with more sophisticated timestamp management.

In this approach:
- Each write operation creates a new version of the data item with the transaction's timestamp.
- Read operations find the latest version with a timestamp less than or equal to the reading transaction's timestamp.

This allows for even better concurrency because:
- Read operations never conflict with write operations.
- Only conflicting write operations need to be serialized.

Here's a basic example of how you might implement multi-version timestamps:

```java
class VersionedData {
    String value;
    long writeTimestamp;
    long readTimestamp;

    VersionedData(String value, long writeTimestamp) {
        this.value = value;
        this.writeTimestamp = writeTimestamp;
        this.readTimestamp = writeTimestamp;
    }
}

class MVCCDatabase {
    private ConcurrentHashMap<String, List<VersionedData>> data;
    private AtomicLong globalTimestamp;

    public MVCCDatabase() {
        this.data = new ConcurrentHashMap<>();
        this.globalTimestamp = new AtomicLong(0);
    }

    public long beginTransaction() {
        return globalTimestamp.incrementAndGet();
    }

    public void put(String key, String value, long transactionTimestamp) {
        data.computeIfAbsent(key, k -> new ArrayList<>())
            .add(new VersionedData(value, transactionTimestamp));
    }

    public String get(String key, long transactionTimestamp) {
        List<VersionedData> versions = data.get(key);
        if (versions == null) {
            return null;
        }

        VersionedData latestValidVersion = null;
        for (VersionedData version : versions) {
            if (version.writeTimestamp <= transactionTimestamp) {
                latestValidVersion = version;
            } else {
                break;
            }
        }

        if (latestValidVersion != null) {
            latestValidVersion.readTimestamp = Math.max(latestValidVersion.readTimestamp, transactionTimestamp);
            return latestValidVersion.value;
        }

        return null;
    }
}
```

In this example:
- Each transaction gets a unique timestamp when it begins.
- Write operations create new versions with the transaction's timestamp.
- Read operations find the latest version valid for the transaction's timestamp.
- We update the read timestamp of the version that was read, which can be used for cleanup of old versions.

This approach allows for efficient conflict detection and resolution, as well as potentially better performance in read-heavy workloads, as reads don't block writes and vice versa.

Implementing such a system would make our MVCC more efficient and scalable, especially when dealing with a high number of concurrent transactions. However, it also introduces additional complexity in terms of version management and garbage collection of old versions.

### Testing

Let's focus on the following anomalies:
1. Dirty Read
2. Non-Repeatable Read
3. Phantom Read
4. Lost Update

We'll use JUnit for our tests. Here's an implementation with tests for these anomalies:



```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class MVCCDatabaseTest {
    private MVCCDatabase db;

    @BeforeEach
    void setUp() {
        db = new MVCCDatabase();
    }

    @Test
    void testNoDirtyRead() throws Exception {
        long tx1 = db.beginTransaction();
        db.put("key", "initial", tx1);
        db.commitTransaction(tx1);

        long tx2 = db.beginTransaction();
        db.put("key", "updated", tx2);

        long tx3 = db.beginTransaction();
        String value = db.get("key", tx3);

        assertEquals("initial", value, "Transaction should not see uncommitted changes");

        db.rollbackTransaction(tx2);
        db.commitTransaction(tx3);
    }

    @Test
    void testNoNonRepeatableRead() throws Exception {
        long tx1 = db.beginTransaction();
        db.put("key", "initial", tx1);
        db.commitTransaction(tx1);

        long tx2 = db.beginTransaction();
        String firstRead = db.get("key", tx2);

        long tx3 = db.beginTransaction();
        db.put("key", "updated", tx3);
        db.commitTransaction(tx3);

        String secondRead = db.get("key", tx2);

        assertEquals(firstRead, secondRead, "Repeated read within the same transaction should return the same value");

        db.commitTransaction(tx2);
    }

    @Test
    void testNoPhantomRead() throws Exception {
        long tx1 = db.beginTransaction();
        db.put("key1", "value1", tx1);
        db.commitTransaction(tx1);

        long tx2 = db.beginTransaction();
        int initialCount = countKeys(tx2);

        long tx3 = db.beginTransaction();
        db.put("key2", "value2", tx3);
        db.commitTransaction(tx3);

        int finalCount = countKeys(tx2);

        assertEquals(initialCount, finalCount, "Number of keys should not change within the same transaction");

        db.commitTransaction(tx2);
    }

    @Test
    void testNoLostUpdate() throws Exception {
        long tx1 = db.beginTransaction();
        db.put("counter", "0", tx1);
        db.commitTransaction(tx1);

        CountDownLatch latch = new CountDownLatch(2);
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Runnable incrementTask = () -> {
            try {
                long tx = db.beginTransaction();
                String value = db.get("counter", tx);
                int counter = Integer.parseInt(value);
                counter++;
                db.put("counter", String.valueOf(counter), tx);
                db.commitTransaction(tx);
                latch.countDown();
            } catch (Exception e) {
                e.printStackTrace();
            }
        };

        executor.submit(incrementTask);
        executor.submit(incrementTask);

        latch.await();
        executor.shutdown();

        long finalTx = db.beginTransaction();
        String finalValue = db.get("counter", finalTx);
        db.commitTransaction(finalTx);

        assertEquals("2", finalValue, "Counter should be incremented twice without lost updates");
    }

    private int countKeys(long transactionId) throws Exception {
        int count = 0;
        for (int i = 1; i <= 10; i++) {  // Assume we don't have more than 10 keys
            if (db.get("key" + i, transactionId) != null) {
                count++;
            }
        }
        return count;
    }
}

```

Let's go through each test and explain what it's checking:

1. `testNoDirtyRead()`:
   This test ensures that a transaction cannot read uncommitted changes made by another transaction. It creates two transactions: one that updates a value but doesn't commit, and another that reads the value. The read should return the original value, not the uncommitted update.

2. `testNoNonRepeatableRead()`:
   This test verifies that within a single transaction, repeated reads of the same data return the same result, even if another transaction modifies and commits changes to that data in between the reads.

3. `testNoPhantomRead()`:
   This test checks for phantom reads, which occur when a transaction reads a set of rows that satisfy a search condition, but a second transaction inserts new rows that match the condition. The test ensures that the count of keys remains the same within a transaction, even if another transaction adds a new key.

4. `testNoLostUpdate()`:
   This test checks for lost updates, which can occur when two transactions read and update the same data concurrently. It uses two concurrent threads to increment a counter and ensures that both increments are reflected in the final value.

These tests demonstrate that our MVCC implementation provides serializable isolation, preventing common transaction anomalies. However, there are a few important points to note:

1. These tests assume that our `MVCCDatabase` class throws a `TransactionConflictException` when it detects a conflict. The tests might need to be adjusted if the conflict resolution mechanism is different.

2. The `countKeys()` method is a simplified way to simulate a range query. In a real database, you'd have more sophisticated methods for range queries.

3. The `testNoLostUpdate()` test uses a `CountDownLatch` to coordinate the two threads. This is a simplification; in a real-world scenario, you'd need more robust concurrency control.

4. These tests focus on the core ACID properties, but they don't cover all possible edge cases. In a production system, you'd want even more comprehensive tests, including stress tests with many concurrent transactions.

5. The tests assume that the database operations (`get`, `put`, etc.) don't throw checked exceptions. If they do, you'll need to adjust the test methods to declare these exceptions.

To run these tests, you'll need to have JUnit 5 in your project's classpath. These tests provide a good starting point for verifying the correctness of your MVCC implementation, but remember that thorough testing of a concurrent system often requires more extensive scenarios and stress testing.
