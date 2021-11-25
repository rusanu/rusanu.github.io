---
id: 2393
title: Common misconceptions with SQL Server 2014 updateable columnstores
date: 2014-06-17T05:08:20+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/06/17/2390-revision-3/
permalink: /2014/06/17/2390-revision-3/
---
## Are INSERTs serialized because of row group locks?

**No.** If a trickle INSERT requires an OPEN row group and it cannot place a lock on it _it will create a new one_. any number of INSERT statements can proceed in parallel, each locking its own row group. While this results in lower quality row groups (possible multiple small OPEN row groups) the decision was explicitly to favor concurrency. This applies as well to BULK INSERTs that fail to create the minimum row group size (~100k rows).

## Does Tuple Mover delete the rows in deltastore?

**No.** Surprised? Each OPEN/CLOSED row group has its own individual deltastore. When the Tuple Mover finishes compressing a row group the entire deltastore _for that row group_ will be **deallocated**. This is done to avoid the explicit logging of individual deltastore row deletes. The deallocation may be deffered and occur only after pending scanners finish since it would be rude the deallocate the rows under a scanner that reads them right now&#8230; Consider that Tuple Mover does no block reads (scans) in any way.

## Does Tuple Mover block INSERTS?

**No.** Both trickle inserts and bulk inserts into clustered columnstores can proceed while the Tuple Mover is compressing a rowgroup. You only need to think about the fact that the Tuple Mover only has business with CLOSED row groups and INSERTs are only concerned with OPEN row groups. There is no overlap so there is no reason for blocking.

## Does Tuple Mover block UPDATEs?

**Yes**. Spooled scans for update cannot reacquire the row if the row storage changes between the scan and the update. So the Tuple Mover is mutually exclusive with any UPDATE statement on the columnstore.