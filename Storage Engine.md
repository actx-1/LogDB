The first revision of this file type is to be text-friendly.

# Data Files
Root storage file:
```
database.db
```
The fact of the design of LogDB is that the database will not need a separate file for transaction logging or journalling, as this all takes place in one file.
This may however lead to some form of fragmentation, especially if individual durable writes are needed (writing 4 KB pages each time). Hence, compactions must be performed.
Compactions should not lock the database to reads and writes, so a second file is needed:
```
database.db-compcation
```
It may be acceptable for the database to delete a file with a matching file name on startup, as it is unlikely to exist unrelated by coincidence.

# File Structure
The intention is for database files to be split into `N` pages, where the size of each page is consistent throughout the file. The first page is the “metadata” page, and for starters, the full schema and all metadata should fit in one page. For that reason, 512 byte pages are not yet supported.
**Structure**:


+------------------+
|      Header      |
+------------------+
|   Remainder of   |
|   metadata page  |
+------------------+
+------------------+
|                  |
|    Data Page     |
|                  |
+------------------+
+------------------+
|                  |
|More Data Pages...|
|                  |
....................

**Header**


The header is to be confirmed, but should contain the following text information, encoded in utf-8:
```
00LogDB
```
And it should contain the following additional information:
- Version integer for the database file.
- Page Size (changes in page size require a rewrite)
The first 64 bytes are reserved for the header.

**Metadata Page**

The first data page (excluding the header) is the first metadata page, hence its size is: `PAGE_SIZE - 64 [bytes]`. However, the metadata page itself is subjected to the page size.
As with the rest of the database file, the metadata page should primarily be a JSON object. However, the addition of arbitrary fields to the JSON object is discouraged, for temporary size concerns. Hence, the metadata page will look something like this:
```
00000000|checksum|[PAGE HEADER]
\n0\n
{"Database Name": "File Folding","Tables": {"0": {"Name": "Metadata"},"1": {"name": "Files"},"2": {"name": "Folders"},"3": {"name": "Logs","ttl": 86400},"4": {"deleted": true}},"Compaction Status": "None"}
```
Which should keep track of the following:
- Database name
- Database schema (table IDs and names) (Table IDs cannot be reused)
- Table default TTL values
And more data... (more to consider in future).

The `[PAGE HEADER]` shown is a page header, and effectively just contains the checksum for the remainder of the page. The first number is an 8-long text-contained HEX number, and is the page number, for a range of `0` to `4294967295` pages (~`16` TiB for a page size of 4K) – the maximum number of pages. The default location for the metadata page is `00000000`, so the first data page is `00000001`, and the last possible data page is `FFFFFFFF`. During future upgrade to a binary format, the page number ID will likely become a binary integer, however it's unclear if the page number is actually necessary, given that the position of the correct page can be identified by position in the file.
**Deferred Metadata**

In the metadata page description, the second part of the metadata page is: `\n0\n`: two newline characters, for separation, and a `0` text zero. The byte represents where the metadata page is stored: `0` says that the metadata page is indeed in the metadata page (the `64` to `PAGE_SIZE - 64` first bytes of the file). However, a value of `1` represents that the metadata page is stored elsewhere in the database: the JSON value should be replaced by the base 10 value of the page ID locating the current schema, and is to be recorded in text (length should not be a concern). This is for additional immutability of the schema, but also so that the schema may in future span multiple pages, if it gets too large. A value for this number of anything other than `0` or `1` is undefined.

**Data Pages**

Data pages are expected to contain the majority of the data in the file. Each page should have a page ID, a header, and then text rows corresponding to the actual data: pages are not separated into tables, but instead for convenient checksumming and planned future features. Furthermore, as data pages may be sentenced as padded pages (for write durability), the header should specify the start position for the text data:
```
00000001|[PAGE CHECKSUM]|[PAGE TYPE]|[HEADER LENGTH]
[0][00][0000]{"key": {"ID": "Text","Number": 6}}(16906788888888)
[0][01][0001]{"key2": {"Path": "/path/to/file","Size": 4096}}(16906788888891)
..........
[PAGE HEADER]
```

`[PAGE TYPE]` is simply `p` for data **p**ages, `d` for **d**onor pages, and `s` for **s**chema donor pages. Just so that the application knows what to read when going sequentially.
The page header is recorded back-to-start at the end of the page, so that new rows can be appended to the start of the page, and new header entries can be appended to the end of the page.

Between the square brackets, each number (text based HEX as above) has its own meaning: the first, `[0]`, specifies the insertion type. `0` refers to data rows, and other values are: `1` for padding, `2` for transaction starts, `3` for transaction commits, `4` for transaction cancellations, but all are to be confirmed. To be discussed later.
The `(16906788888888)` values are unix epoch timestamps, in a format yet to be confirmed but probably base 16 text HEX, representing the time at which each row was inserted, for point in time scanning during compactions or backups.

The second set, `[00]`, refers to the table where the data is to be inserted: tables do not own pages, but all rows of a table are marked as belonging to that table, by table ID, as defined in the schema. Any rows belonging to a deleted table are hence considered deleted.

The third set, `[0000]`, represents the ID of the transaction. The value `0000` is reserved, and means that the modification is not part of a transaction. Any other number is to be taken as a transaction ID, and is to be handled in-memory at the time: the database being closed while transactions are open is deemed to be sufficient to assume the transactions to be aborted, and seeing a transaction start, followed by data modifications, followed by another transaction start for the same transaction ID is assumed to mean that the database was closed before the first transaction was committed, and hence only the modifications following the second transaction start are to be kept. 4 hex values implies that only 65535 transactions will be possible before suffering transaction wrap-around, however it is also assumed that if the transaction ID can wrap around before an earlier transaction is committed, then the earlier transaction can be omitted.

The actual data should be newline-escaped JSON, and the position of the entire row (square bracket data + timestamp included) in the page and length should be recorded in the header.

Additionally, it is possible that a row may take up more space than just one page. For this, the header should include a specific flag to indicate that the very next `N` pages should be handled as part of this page: the individual pages all checksum independently, but the next pages will have no headers, and will effectively function as a part of the previous page, sharing its header. If the page header of the previous page is unable to expand to accommodate additional rows on the next pages, then it is acceptable to pass all the free space to get to the next independent page.

The header should hence look as follows (read back to front):
```
00001D,000024|000000,00001B|02
```
The values are all separated by vertical slashes (`|`). The first value is `00001D,000024`, and regards the second row: `00001D` is the position in the page (having accounted for Page ID and Page Checksum), and `000024` is the length of the row.
The next value pairs are hence for the first row, and finally, the last value in the header is `02`, which specifies that the next two pages effectively function as data for the current page.
When determining whether to create a new page or allocate more pages to the current page: rows projected to be smaller than the data remaining in the page should be written on the page, and rows projected to be larger than the current free space but larger than the default free space in an empty page should be given a whole new page. Only rows which are calculated to be larger than the free space in an empty page are to be strewn across pages like that.


Donor page layout:
```
00000002|[CHECKSUM]|0000
[0][00][0000]{data}(ts)
```
# Compaction
Compaction is a hefty task: it does not require the database to be locked (unlike with SQLite or others) when the files are swapped, but compaction relies on the imutability of the database:
- The timestamp of the latest insertion in the database is determined
- The metadata page of the primary database is updated to include `Compaction Status = Compacting`
- The compactor creates the new file: `database.db-compaction`
- The database metadata page is written according to the old one, including: `Compaction Status = Output`, to show that it is the target of a compaction
- The database is read into memory, with a few exceptions: transaction begin and finish records are not recorded. Any rows within a successful transaction are recorded, but ones in an unsuccessful transaction are dropped. Rows whose table IDs do not exist should not be recorded. Only the latest version of any key is recorded, and any trailing `null` values are dropped, along with the key. Timestamps are included.
- All tables should have their IDs reduced to the minimum possible, to mitigate the impact of “table holes” caused by deleting tables.
- The database should be written down into, trying to put the schema and any schema donor pages at the start, then all data afterwards, and transaction IDs should be set to `0000`. Table IDs should be maintained but adjusted to the new table IDs.
- The database should continue from the first row whose TS value is larger than the recorded timestamp.
- The database should temporarily be locked (shouldn't be for long).
- The compaction should continue from the recorded TS, right up until everything that has been written during the compaction is included.
- The old database schema should be checked against the new one for changes, and the new database schema should be updated if necessary. Additionally, `Compaction Status` should be set to `None`, to represent that the compaction is finised.
- The new database should be renamed into the old database to overwrite it.
- All open handles on the database should refresh, in order to be connected to the new database file.
- Operations can continue as usual.
