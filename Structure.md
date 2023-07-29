# Design
The structure of the database management system takes inspiration from SQLite: a self-contained executable, potentially making use of IPC (shared memory/mmap) to communicate file access between instances.
(TBC) For the testing period, the application is currently just an indepentend python module which can be called from other python scripts.

# Use
Most databases provide a method for applications to provide the database with data, and can then later retrieve that data at an arbtrary point. Many databases make use of write-ahead logs (WALs) to ensure the durability of write operations.
The purpose of LogDB is quite different, and significantly more simple: LogDB merely acts as a WAL for the application directly.



# Operations
Operations are defined as what the database can do: CRUD operations (INSERT, SELECT, UPDATE, DELETE) are a good starting point, however the structure of the database system (the storage engine) leads to the need for UPSERT operations to be used in place of INSERT and UPDATE: last-write-wins semantics are employed for all writes.
