# Operations
Operations are defined as what the database can do: CRUD operations (INSERT, SELECT, UPDATE, DELETE) are a good starting point, however the structure of the database system (the storage engine) leads to the need for UPSERT operations to be used in place of INSERT and UPDATE: last-write-wins semantics are employed for all writes.
