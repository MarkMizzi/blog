---
title: "The Postgres Heap file | 1"
date: 2022-10-12T11:44:25+02:00
draft: false
---

Postgres stores relations in a single, unorganized file called a heap file. This format puts it at odds with other OLTP systems, which tend to store relations in an on-disk data structure organized by some high level constraint. For example, [MySQL (InnoDB) stores relations in a B+ tree](https://www.percona.com/blog/2017/04/10/innodb-page-merging-and-page-splitting/) where the search key is the primary key.

The atomic unit of disk access is an 8KB page. Each page has the following [structure](https://www.postgresql.org/docs/14/storage-page-layout.html):

![Postgres heap page](/postgres_heap_page.png)

Line pointers (or item ids) are used by other parts of the system to identify the rows in the page. Row datums contain all the data for a particular tuple. 

Space for line pointers is reserved starting from the page header onwards, while space for datums is reserved from the end of the page backwards. When the two meet in the middle and the page can no longer fit a new tuple, the page is considered full. Rows are not allowed to cross page boundaries.

There are a lot of concerns that arise from this layout:

- How does the system keep track of the free space in each page?

- What happens when a row contains data larger than 8 KB?

- What if we want to update a row but the new version does not fit in the page?

- How does Postgres collect statistics from the heap file to run efficient queries?

Being a mature system, Postgres has solutions to all of these issues, and we will touch on them below.

It is also worth noting at this point that this layout puts Postgres somewhere between OLAP and OLTP-oriented systems. On the one hand, scanning a heap file is much faster than scanning a relation stored as a B+ tree (sequential v.s. random disk access). So Postgres computes aggregates much faster than MySQL. But the storage is still **row oriented**, so it is easily beaten by a columnar database. Unfortunately, the boost in query performance is a trade off with update performance. Postgres can suffer from write amplification, and maintaining a frequently updated heap file is a complex and arduous task for the system.

# (Not) Storing `NULL` values

# Free space management


