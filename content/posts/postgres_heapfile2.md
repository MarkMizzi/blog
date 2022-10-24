---
title: "The Postgres Heap file | 2 TOASTing"
date: 2022-10-23T11:44:25+02:00
draft: false
---

Postgres uses a technique called [TOAST](https://www.postgresql.org/docs/14/storage-TOAST.html) (The Oversized-Attribute Storage Technique) to deal with tuples that can't fit in a page. Essentially TOASTing allows individual elements in a tuple to be compressed and/or stored "out of line" in another table called the relation's TOAST table. The process of compressing or storing out of line is called TOASTing, while the reverse is called de-TOASTing (duh).

Only variable length data types can be TOASTed. After all, why compress an integer that is always 4 bytes in size? Variable length data type are stored in pedestrian fashion: with an integer containing the byte length (including the bytes for the length itself), followed by payload bytes. The TOAST system uses up two bits of the length (higher order on big endian, lower order on little endian) to denote the state of the element (whether it has been compressed, stored out of line, etc.)

A TOASTable tuple element can be in one of 4 states:

| TOAST bits | Name              | Description                                                | Maximum size (inline)           |
|------------|-------------------|------------------------------------------------------------|---------------------------------|
| 00         | Regular           | Uncompressed, inline storage with 4-byte length.           | ~8000 bytes, due to page size.  |
| 1(0/1)     | Short form        | Unaligned, uncompressed, inline storage with 7-bit length. | 127 bytes, due to 7-bit length. |
| 10(000000) | Out of line       | Out of line storage.                                       | N/A                             |
| 01         | Inline compressed | Compressed, inline storage with 4-byte length              | ~8000 bytes, due to page size.  |

Elements stored in short form are not aligned to 4-byte boundaries. The length consists of the remaining 7 bits in the byte containing TOAST bits.

When the byte containing the TOAST bits is `10000000`, the inline datum is not a short form element, but instead a TOAST pointer to the actual data, which is stored out of line in TOAST tables. This does not clash with short form format, since a self-inclusive length can never be 0.

The next few bytes after the first byte contain the type and size of the TOAST pointer, followed by the pointer itself. Out of line values may be compressed. Compression information is stored in the TOAST pointer.

# Making the most out of plain TOAST

Postgres uses two configuration parameters to trigger the TOAST system: `TOAST_TUPLE_THRESHOLD`, which (annoyingly) is a compile-time constant set to 2KB by default, and `TOAST_TUPLE_TARGET`.

The TOAST system is triggered when a row being inserted into the heap file has size larger than `TOAST_TUPLE_THRESHOLD`. At this point the system tries to shrink the row to a size smaller than `TOAST_TUPLE_TARGET` by compressing TOASTable columns. If this fails, the columns are stored out of line in a TOAST table.

`TOAST_TUPLE_TARGET` is 2KB by default, but can be set during table creation or using an `ALTER TABLE` statement:

```sql
CREATE TABLE unary (
    a INT PRIMARY KEY
    
-- set TOAST tuple target
-- typically valid values are in the range [128, 8160] bytes
) WITH (toast_tuple_target = 4080);
```

```sql
-- Rows present in unary table are not affected immediately,
--      instead we have to rewrite the table (e.g. with VACUUM FULL).
ALTER TABLE unary
    SET (toast_tuple_target = 4080);
```

Postgres also offers more fine-grained control:

```sql
CREATE TABLE t (a VARCHAR, b INT[], c BYTEA, d JSONB);

------ Per-column TOAST configuration

-- Disable compression, out of line storage and short form storage.
ALTER TABLE t
    ALTER COLUMN a SET STORAGE PLAIN;

-- Allow compression and out of line storage.
-- Typically this is the default.
ALTER TABLE t
    ALTER COLUMN b SET STORAGE EXTENDED;

-- Allow out of line storage, but not compression.
ALTER TABLE t
    ALTER COLUMN c SET STORAGE EXTERNAL;

-- Allow compression, but not out of line storage.
ALTER TABLE t
    ALTER COLUMN d SET STORAGE MAIN;
```

What happens if we disable out of line storage for some column, but the row (even after compression) does not fit in a page? In this case the system ignores our bad advice, and stores the column element out of line anyway.

Postgres allows us to choose the compression algorithm it uses. There's a connection parameter called [default_toast_compression](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-DEFAULT-TOAST-COMPRESSION) that is consulted when a client inserts TOASTable elements. This can also be overridden for specific columns in the `CREATE TABLE` statement:

```sql
CREATE TABLE unary (
    a VARCHAR COMPRESSION lz4
);
```

or by running an `ALTER TABLE`:

```sql
ALTER TABLE unary
    ALTER COLUMN a SET COMPRESSION pglz;
```

The compression algorithm that was used to compress a particular element can be found using the [`pg_column_compression`](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-DBOBJECT) function.

The two algorithms supported by default are `pglz` and `lz4` (since Postgres 14).

It's worth noting that since Postgres compresses each element separately, the space savings tend to be lower than columnar databases which compress entire columns.

Let's see an example of TOAST in practice. As an example we will use this simple table:

```sql
CREATE TABLE unary(a VARCHAR);
```

In order to see out of line storage at work, we need a function that generates uncompressable strings:

```sql
CREATE OR REPLACE FUNCTION random_str(len INT, seed INT)
    RETURNS VARCHAR
    AS
$$
    SELECT string_agg(md5((seed+i)::VARCHAR), '')
    FROM generate_series(1, len/octet_length(md5(seed::VARCHAR))) AS i
$$ LANGUAGE 'sql';
```

Let's populate the table with one row:

```sql
INSERT INTO unary
SELECT random_str(5000, extract(epoch FROM now())::INTEGER);
```

Each character in a `VARCHAR` [takes up at least one byte](https://www.postgresql.org/docs/current/multibyte.html#:~:text=The%20character%20set%20support%20in,8,%20and%20Mule%20internal%20code.), making this row significantly larger than `TOAST_TUPLE_THRESHOLD`. 

Since the row element can't be compressed effectively, its data is stored out of line in the table's **TOAST table**, in the `pg_toast` schema. The following query gives us the TOAST table's name:

```sql
SELECT relname
FROM pg_class
WHERE oid = (
    SELECT reltoastrelid
    FROM pg_class
    WHERE relname = 'unary'
);
```

On my system this returns `pg_toast_16384`. Selecting all rows from this table:

```sql
SELECT * FROM pg_toast.pg_toast_16384;
```

yields

| chunk_id | chunk_seq | chunk_data                                         |
|----------|-----------|----------------------------------------------------|
| 16390    | 0         | c4ca4238a0b923820dcc509a6f75849bc81e728d9d4c2f6... |
| 16390    | 1         | 29b125f8597834fa83a4ea5d2f1c4608232e07d3aa3d998... |
| 16390    | 2         | a77393dd069059b7ef840f0c74a814ec9237b6ecec5decc... |

So the inserted row element was split into chunks. `chunk_id` identifies the element being chunked, `chunk_seq` is a serial number assigned to chunks, and `chunk_data` contains the actual data itself.

The contents of the `unary` table are:

| a                                                  |
|----------------------------------------------------|
| c4ca4238a0b923820dcc509a6f75849bc81e728d9d4c2f6... |

so as expected, the row is not compressed before chunking. The system realizes that compression is ineffective.

Compression by itself is much harder to observe. In order to see it in action, let's disable out of line storage:

```sql
ALTER TABLE unary
    ALTER COLUMN a SET STORAGE MAIN;
```

and insert a compressable row:

```sql
TRUNCATE unary;
INSERT INTO unary
SELECT repeat('abcd', 2100/4);
```

The following query gives us the on-disk size of each `a` element in `unary`:

```sql
SELECT unary.a, pg_column_size(unary.a)
from unary;
```

and returns

| a                      | pg_column_size |
|------------------------|----------------|
| abcdabcdabcdabcdabc... | 38             |

on my system. Wait, just 38 bytes? This is much smaller than the expected >2100 bytes, indicating that the row has been compressed and stored inline. (Sceptics can check the TOAST table to confirm that it is empty).

Now what happens if we insert a row that can't be compressed, or that even when compressed does not fit in an 8KB page?

Running

```sql
TRUNCATE unary;

INSERT INTO unary
SELECT random_str(8200, extract(epoch from now())::integer);

-- number of chunks used to store the inserted row out of line
-- 0 means that the row was stored inline.
SELECT count(*) FROM pg_toast.pg_toast_16384;
```

yields `5`. The system could not fit the inserted row inline, and resorted to out of line storage despite our instruction. 

Running the same sequence of commands with `random_str(8100,...)` gives us `0`, indicating that (on my system, at least) the threshold for fitting a row in a page is somewhere between `8104` and `8204` bytes. This number is subject to change between Postgres releases, due to modifications to the page header or line pointer structure.

# How fast is TOAST?

The performance enthusiasts among you may be shaking their heads in dismay. For every page we scan we may have to fetch many more pages containing chunks of out of line row elements: read amplification! Even worse, these pages are stored in the TOAST table, which is likely to be stored "far away" from the original relation on disk: random disk I/O!

Clearly the worst case scenario here is a full scan of the heap file. How badly does TOAST affect performance in this case?

As a litmus test, let's reconsider the `unary` table.
