# Design
The structure of the database management system takes inspiration from SQLite: a self-contained executable, potentially making use of IPC (shared memory/mmap) to communicate file access between instances. The database paradigm is that of a key-value or document store.
The general idea is that the application can call on LogDB to write a dictionary/JSON object to the database. The write requests will then be persistent, and can be reloaded later during a full-file scan. Upserts and deletion values for any prior-written key are accepted. Anything which is read from the database will be in the format of a JSON object, or a dictionary.

(TBC) For the testing period, the application is currently just an indepentend python module which can be called from other python scripts.
The first versions of the application and file format are to be written in python, for ease of development purposes. However, the intention is for the production-ready version to be written in Go.
Furthermore, unlike with SQL-like database management system querying languages, the intent is for the application to rely on specific LogDB functions.

# Use
Most databases provide a method for applications to provide the database with data, and can then later retrieve that data at an arbtrary point. Many databases make use of write-ahead logs (WALs) to ensure the durability of write operations.
The purpose of LogDB is quite different, and significantly more simple: LogDB merely acts as a WAL for the application directly: the application retains its own internal copy of whatever it wanted to make persistent, and LogDB writes it to disk in a fashion guaranteed to be persistent. Upon start-up following a shut-down or a crash, the application can reload the entire data set, and store it however it likes.
This access pattern may prove useful in a few ways: fast, direct write transactions that effectively write a copy of a native variable allow applications to guarantee durability and crash resistance of their data to varying degrees, without needing to write to a backend database, which may be useful in a few of the following use cases, for example:
- Autosaving game saves, where new information is to be written to disk constantly, but where the game application itself does not need to dynamically load save data values from disk
- Desktop applications which save new data frequently
- Persistent and durable logging
- Durable write caches
- Dirty cache persistence
- Durable incrementing counters
Further examples can be found in the `Examples` section. 

LogDB should also make the configuration of a single logger more simple than configuring an entire database management system. However, as a caveat, the database is designed to only be accessible through one thread.

# Features

- Transactions
- Persistent schema
- Persistence levels
- Tables
- Compaction

# Operations
Operations are defined as what the database can do: interaction with the database management system.

CRUD operations (INSERT, SELECT, UPDATE, DELETE) are a good starting point, however the structure of the database system (the storage engine) leads to the need for UPSERT operations to be used in place of INSERT and UPDATE: last-write-wins semantics are employed for all writes.

- INSERT/UPSERT: Inserting data to the database should be done in a key-value manner, where the key is a single supplied data point, and the value is a dictionary/JSON object. Inserting will overwrite any prior instances of the key if it exists.
- UPDATE: Update is simply an alias for an `INSERT` operation.
- DELETE: Delete is a special case for the `UPDATE` operation, and the value which it writes is a `null` value. Deletion markers will be removed during compaction.
- SELECT: Selecting can be done for an entire table, or the whole database, in order to get a complete copy of everything stored in the database. The database has no readable page cache, as its function is that of a WAL, and the intended use is to read all data required at once. Operations will be included to find a particular key, but such operations will equate to a full database scan anyway, and so should not be used unless necessary. Reading a value which does not exist is equivalent to reading a value of `null`; there is no distinction between keys with `null` values and keys which do not exist.
- Transactions: Session-specific transactions can be used to guarantee the atomicity and durability of a group of operations. Operations performed within a transaction will be all-or-nothing, and read operations performed within a transaction will apply to the data visible up until that point. However, of the ACID principles, consistency and isolation are not guaranteed, so write operations in a transaction, while only being confirmed to have persisted at the end of a transaction, will later become visible at the time they were made after the transaction is committed, so be careful using transactions.
- COMPACT: Over time, especially with update-heavy or delete-heavy workloads, it is expected that the data file will become fragmented. Furthermore, if durable writes are to be enabled, then the space amplification of the file will be palable. Hence, it is desirable for the file to be able to “compact” itself, anolagous to a VACUUM for relational database management systems, or COMPACT for wide column stores. It is hoped that you will only need enough free memory for the active data in the file in order to load the file, however compactions will likely take double the file's total size. I'm not sure what can be done with a database which must be compacted but is too large to be compacted in memory – certainly let me know if that ever happens.


# Examples
Some examples of real-world use cases where LogDB may find use.

**Incrementing a Counter**
