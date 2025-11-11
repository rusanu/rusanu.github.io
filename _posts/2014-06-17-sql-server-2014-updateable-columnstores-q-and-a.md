---
id: 2390
title: SQL Server 2014 updateable columnstores Q and A
date: 2014-06-17T05:26:24+00:00
author: remus
layout: post
guid: /?p=2390
permalink: /2014/06/17/sql-server-2014-updateable-columnstores-q-and-a/
categories:
  - Columnstore
  - SQL 2014
  - Troubleshooting
---
## Are INSERTs serialized because of row group locks?

**No.** If a trickle INSERT requires an OPEN row group and it cannot place a lock on it _it will create a new one_. any number of INSERT statements can proceed in parallel, each locking its own row group. While this results in lower quality row groups (possible multiple small OPEN row groups) the decision was explicitly to favor concurrency. This applies as well to BULK INSERTs that fail to create the minimum row group size (~100k rows).

## Does Tuple Mover delete the rows in deltastore?

**No.** Surprised? Each OPEN/CLOSED row group has its own individual deltastore. When the Tuple Mover finishes compressing a row group the entire deltastore _for that row group_ will be **deallocated**. This is done to avoid the explicit logging of each individual row delete from deltastores. This is the same difference as between TRUNCATE and DELETE.

## Does Tuple Mover block reads?

**No.** No way. Absolutely no.

## Does Tuple Mover block INSERTS?

**No.** Both trickle inserts and bulk inserts into clustered columnstores can proceed while the Tuple Mover is compressing a rowgroup. You only need to think about the fact that the Tuple Mover only has business with CLOSED row groups and INSERTs are only concerned with OPEN row groups. There is no overlap so there is no reason for blocking.

## Does Tuple Mover block UPDATE, DELETE, MERGE?

**Yes**. Spooled scans for update (or delete) cannot re-acquire the row if the row storage changes between the scan and the update. So the Tuple Mover is mutually exclusive with any UPDATE, DELETE or MERGE statement on the columnstore. This exclusion is achieved by a special object level lock that is acquired by UPDATE/DELETE/MERGE in shared mode and by the Tuple Mover in exclusive mode.

## Can Tuple Mover, or REORGANIZE, shift rows between row groups?

**No.** The Tuple Mover (and REORGANIZE) can only compress a row group but it cannot shift rows between row groups. Particularly it cannot &#8216;stich&#8217; several small deltastores into one compressed row group. REBUILD may appear that it can shift or move rows, but REBUILD is doing exactly what the name implies: a full rebuild. It reads the existing columnstore and builds a new one. The organization (row group numbers and size) of the new (rebuilt) columnstore has basically no relation with the original organization of the old columnstore.