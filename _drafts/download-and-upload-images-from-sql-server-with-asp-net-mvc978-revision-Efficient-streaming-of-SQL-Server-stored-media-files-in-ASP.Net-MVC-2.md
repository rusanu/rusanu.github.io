---
id: 979
title: Efficient streaming of SQL Server stored media files in ASP.Net MVC 2
date: 2010-12-21T13:25:16+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/12/21/978-revision/
permalink: /2010/12/21/978-revision/
---
A frequent question that pops up on discussion forums is how to serve an image that is stored in a SQL Server table from an ASP.Net application. Unfortunately the answer is almost always wrong, as the prevalent solution involves copying the **entire** image file into memory before returning it to the client. This solution works fine when tested with a light load and returning few small images. But in production environment the memory consumption required by all those image files stored as byte arrays in memory causes serious performance degradation. A good solution must use streaming semantics, transferring the data in small chunks from the SQL Server to the HTTP returned result.

The SqlClient components do offer streaming semantics for large result sets, including large BLOB fields, but the client has to specifically ask for it. The &#8216;secret ingredient&#8217; is the passing in the <a hreh="http://msdn.microsoft.com/en-us/library/system.data.commandbehavior.aspx" target="_blank"><tt>CommandBehavior.SequentialAccess</tt></a> flag to the SqlCommand.ExecuteReader: