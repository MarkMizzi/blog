---
title: "The Postgres Heap file | 1"
date: 2022-10-12T11:44:25+02:00
draft: false
---

Postgres stores relations in a single, unorganized file called a heap file. This format puts it at odds with other OLTP systems, which tend to store relations in an on-disk data structure organized by some high level constraint. For example, [MySQL (InnoDB) stores relations in a B+ tree](https://www.percona.com/blog/2017/04/10/innodb-page-merging-and-page-splitting/) where the search key is the primary key.

The atomic unit of disk access is an 8KB page. Each page has the following [structure](https://www.postgresql.org/docs/14/storage-page-layout.html):

![Postgres heap page](/postgres_heap_page.svg)

Line pointers (or item ids) are used by other parts of the system to identify the rows in the page. Row datums contain all the data for a particular tuple. 

Space for line pointers is reserved starting from the page header onwards, while space for datums is reserved from the end of the page backwards. When the two meet in the middle and the page can no longer fit a new tuple, the page is considered full. Rows are not allowed to cross page boundaries.

There are a lot of concerns that arise from this layout:

- How does the system keep track of the free space in each page?

- What happens when a row contains data larger than 8 KB?

- What if we want to update a row but the new version does not fit in the page?

- How does Postgres collect statistics from the heap file to run efficient queries?

Being a mature system, Postgres has solutions to all of these issues, and we will touch on them below.

It is also worth noting at this point that this layout puts Postgres somewhere between OLAP and OLTP-oriented systems. On the one hand, scanning a heap file is much faster than scanning a relation stored as a B+ tree (sequential v.s. random disk access). So Postgres computes aggregates much faster than MySQL. But the storage is still **row oriented**, so it is easily beaten by a columnar database. Unfortunately, the boost in query performance is a trade off with update performance. Postgres can suffer from write amplification, and maintaining a frequently updated heap file is a complex and arduous task for the system.

# A closer look at pages

What makes up page headers, line pointers, and the actual rows? Well [page headers are not very interesting](https://www.postgresql.org/docs/14/storage-page-layout.html#PAGE-TABLE). They contain some bookkeeping information related to WAL, integrity checking, versioning, etc...

Line pointers are 4 bytes that contain the offset and length of a row in the page (as well as two status flags). The pedantic reader can find all the gory details [here](https://github.com/postgres/postgres/blob/REL_14_STABLE/src/include/storage/itemid.h#L24).

Actual row data may be moved around **within the same page** in order to compact storage (e.g. during `VACUUM`ing). In contrast, line pointers remain at the same page offset for their entire lifetime. Since every line pointer is the same size, we can always fit new ones in the gaps, mitigating the need for compaction. For this reason, other data structures (indexes) generally refer to heap tuples via [`ItemPointer`](https://github.com/postgres/postgres/blob/REL_14_STABLE/src/include/storage/itemptr.h#L36)s, which are a pair of page number and line pointer offset.

Finally, the row itself consists of a 23 byte header, an (optional) bitmap, and then the user data itself.

Here are the fields in the header (copied from [the official documentation](https://www.postgresql.org/docs/14/storage-page-layout.html#STORAGE-TUPLE-LAYOUT)):

| Field         | Type            | Length  | Description                                           |
|---------------|-----------------|---------|-------------------------------------------------------|
| `t_xmin`      | TransactionId   | 4 bytes | insert XID stamp                                      |
| `t_xmax`      | TransactionId   | 4 bytes | delete XID stamp                                      |
| `t_cid`       | CommandId       | 4 bytes | insert and/or delete CID stamp (overlays with t_xvac) |
| `t_xvac`      | TransactionId   | 4 bytes | XID for VACUUM operation moving a row version         |
| `t_ctid`      | ItemPointerData | 6 bytes | current TID of this or newer row version              |
| `t_infomask2` | uint16          | 2 bytes | number of attributes, plus various flag bits          |
| `t_infomask`  | uint16          | 2 bytes | various flag bits                                     |
| `t_hoff`      | uint8           | 1 byte  | offset to user data                                   |

The first 5 fields used to keep track of the **current status** of the row. We'll meet them later on in the article.

`t_infomask` [contains the following fields](https://github.com/postgres/postgres/blob/REL_14_STABLE/src/include/access/htup_details.h#L189):

| Field                   | Mask   | Description                                                                      |
|-------------------------|--------|----------------------------------------------------------------------------------|
| `HEAP_HASNULL`          | 0x0001 | has null attribute(s)                                                            |
| `HEAP_HASVARWIDTH`      | 0x0002 | has variable-width attribute(s)                                                  |
| `HEAP_HASEXTERNAL`      | 0x0004 | has external stored attribute(s)                                                 |
| `HEAP_HASOID_OLD`       | 0x0008 | has an object-id field                                                           |
| `HEAP_XMAX_KEYSHR_LOCK` | 0x0010 | xmax is a key-shared locker                                                      |
| `HEAP_COMBOCID`         | 0x0020 | t_cid is a combo CID                                                             |
| `HEAP_XMAX_EXCL_LOCK`   | 0x0040 | xmax is exclusive locker                                                         |
| `HEAP_XMAX_LOCK_ONLY`   | 0x0080 | xmax, if valid, is only a locker                                                 |
| `HEAP_XMIN_COMMITTED`   | 0x0100 | t_xmin committed                                                                 |
| `HEAP_XMIN_INVALID`     | 0x0200 | t_xmin invalid/aborted                                                           |
| `HEAP_XMAX_COMMITTED`   | 0x0400 | t_xmax committed                                                                 |
| `HEAP_XMAX_INVALID`     | 0x0800 | t_xmax invalid/aborted                                                           |
| `HEAP_XMAX_IS_MULTI`    | 0x1000 | t_xmax is a MultiXactId                                                          |
| `HEAP_UPDATED`          | 0x2000 | this is UPDATEd version of row                                                   |
| `HEAP_MOVED_OFF`        | 0x4000 | moved to another place by pre-9.0 VACUUM FULL; kept for binary upgrade support   |
| `HEAP_MOVED_IN`         | 0x8000 | moved from another place by pre-9.0 VACUUM FULL; kept for binary upgrade support |

The first four are used to decode user data when fetching a row. `HEAP_HASNULL` is particularly interesting. If true, the page header is followed by a `NULL` bitmap, which contains enough bytes to fit one bit per column. Instead of explicitly storing `NULL`s, the system then simply resets the corresponding bit in the bitmap. (`NOT NULL` columns always have a set bit).

`t_infomask2` is [far less interesting](https://github.com/postgres/postgres/blob/REL_14_STABLE/src/include/access/htup_details.h#L276):

| Field               | Mask   | Description                                               |
|---------------------|--------|-----------------------------------------------------------|
| `HEAP_NATTS_MASK`   | 0x07FF | 11 bits for number of attributes                          |
| N/A                 | 0x1800 | Unused                                                    |
| `HEAP_KEYS_UPDATED` | 0x2000 | tuple was updated and key cols modified, or tuple deleted |
| `HEAP_HOT_UPDATED`  | 0x4000 | tuple was HOT-updated                                     |
| `HEAP_ONLY_TUPLE`   | 0x8000 | this is heap-only tuple                                   |
| `HEAP2_XACT_MASK`   | 0xE000 | visibility-related bits                                   |

To better understand the row header it's worth reading through [`src/include/access/htup_details.h`](https://github.com/postgres/postgres/blob/REL_14_STABLE/src/include/access/htup_details.h), which is peppered with helpful comments.

# Packing values

What about the user data itself? 

This starts `h_off` bytes after the start of the row header. Fixed size types are stored as is. Variable length types are the subject of the next part.

There is one more caveat to storage, which is padding.
