---
id: 2538
title: 'SQL Injection:  cast of Unicode homoglyphs can introduce additional single quotes'
date: 2017-11-24T05:39:03+00:00
author: remus
layout: revision
guid: http://rusanu.com/2017/11/24/2534-revision-v1/
permalink: /2017/11/24/2534-revision-v1/
---
<pre><code class="prettyprint lang-sql">
declare @inject nvarchar(4000) = NCHAR(0x02bc) + N'; select 1/0; select ' + nchar(0x02bc);
declare @safe nvarchar(4000) = REPLACE(@inject, N'''', N'''''');
declare @sql varchar(4000) = N'SELECT ''' + @safe + N'''';

print @inject;
print @safe;
print @sql;

exec(@sql);
go

declare @inject nvarchar(4000) = NCHAR(0x02bc) + N'; select 1/0; select ' + nchar(0x02bc);
declare @safe nvarchar(4000) = QUOTENAME(@inject, N'''');
declare @sql varchar(8000) = N'SELECT ' + @safe;

print @inject;
print @safe;
print @sql;

exec(@sql);
go
</code></pre>

<pre><code style="color:red">
Msg 8134, Level 16, State 1, Line 1
Divide by zero error encountered.
</code></pre>

In both examples above the SQL executed apparently _should_ had been safe from SQL injection, but it isn&#8217;t. Neither <tt>REPLACE</tt> nor <tt>QUOTENAME</tt> were able to properly protect and the injected division by zero was executed. The problem is the [Unicode MODIFIER LETTER APOSTROPHE](http://www.fileformat.info/info/unicode/char/2bc/index.htm) (<tt>NCHAR(0x02bc)</tt>) character that I used in constructing the NVARCHAR value, which is then implicitly cast to VARCHAR. This cast is converting the special &#8216;modifier letter apostrophe&#8217; character to a plain single quote. This introduces new single quotes in the value, after the supposedly safe escaping occurred. The result is plain old SQL Injection.

In order for this problem to occur, there has to exist an implicit or explicit conversion (a cast) from Unicode to ASCII (ie. NVARCHAR to VARCHAR) and this conversion must occur _after_ the single quote escaping occurred.