---
id: 2392
title: Common misconceptions with SQL Server 2014 updateable columnstores
date: 2014-06-17T04:58:58+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/06/17/2390-revision-2/
permalink: /2014/06/17/2390-revision-2/
---
## Does Tuple Mover block INSERTS?

**No.** Both trickle inserts and bulk inserts into clustered columnstores can proceed while the Tuple Mover is compressing a rowgroup. You only need to think about the fact that the Tuple Mover only has business with CLOSED row groups and INSERTs are only concerned with OPEN row groups. There is no overlap so there is no reason for blocking.

## Are INSERTs serialized because of row group locks?

**No.** If a trickle INSERT requires an OPEN row group and it cannot place a lock on it _it will create a new one_. any number of INSERT statements can proceed in parallel, each locking its own row group. While this results in lower quality row groups (possible multiple small OPEN row groups) the decision was explicitly to favor concurrency. This applies as well to BULK INSERTs that fail to create the minimum row group size (~100k rows).

## Does Tuple Mover delete the rows in deltastore?

**No.** Surprised? Each OPEN/CLOSED row group has its own individual deltastore. When the Tuple Mover finishes compressing a row group the deltastore will be **deallocated**. This is done to avoid the explicit logging of individual deltastore row deletes. The deallocation may be a deffered one