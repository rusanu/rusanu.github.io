---
id: 1003
title: Retrieving images from SQL Server with ASP.Net MVC
date: 2010-12-26T19:36:43+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/12/26/978-revision-24/
permalink: /2010/12/26/978-revision-24/
---
A frequent question that pops up on discussion forums is how to serve an image that is stored in a SQL Server table from an ASP.Net application. Unfortunately the answer is almost always wrong, as the prevalent solution involves copying the **entire** image file into memory before returning it to the client. This solution works fine when tested with a light load and returning few small images. But in production environment the memory consumption required by all those image files stored as byte arrays in memory causes serious performance degradation. A good solution must use streaming semantics, transferring the data in small chunks from the SQL Server to the HTTP returned result.

The SqlClient components do offer streaming semantics for large result sets, including large BLOB fields, but the client has to specifically ask for it. The &#8216;secret ingredient&#8217; is the passing in the <a hreh="http://msdn.microsoft.com/en-us/library/system.data.commandbehavior.aspx" target="_blank"><tt>CommandBehavior.SequentialAccess</tt></a> flag to the SqlCommand.ExecuteReader:

> Provides a way for the DataReader to handle rows that contain columns with large binary values. Rather than loading the entire row, SequentialAccess enables the DataReader to load data as a stream. You can then use the GetBytes or GetChars method to specify a byte location to start the read operation, and a limited buffer size for the data being returned.

To leverage this streaming semantics in practice, we need a bit of plumbing in place. Lets say that out ASP.Net MVC solution exposes a virtual path where media files are located. A requests like <tt>"http://site/Media/IMG0042.JPG"</tt> needs to return the content of the file IMG0042.JPG from our database. First, we&#8217;ll add a special route in Global.asax.cs:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
            routes.MapRoute(
                "Media",
                "Media/{filename}",
                new { controller = "Media", action = "GetFile" },
                new { filename = @"[^/?*:;{}\\]+" });
&lt;/pre>
&lt;p></code>

Next we need to the <tt>MediaController</tt> that handles the request for files in the <tt>GetFile</tt> action. Since we want to return the file as a streaming response, the MediaController.GetFile method should return an ActionResult that returns the content of the requested file. ASP.Net MVC already has an ActionResult implmenetation for an arbitrary stream, the <a href="http://msdn.microsoft.com/en-us/library/system.web.mvc.filestreamresult.aspx" target="_blank"><tt>FileStreamResult</tt></a>.  
<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
    public class MediaController : Controller
    {
        public ActionResult GetFile(string filename)
        {
            FileStreamResult result = new FileStreamResult(...);
            ...
            Response.BufferOutput = false;
            return result;
        }
    }
&lt;/pre>
&lt;p></code>

Note that we disable the output buffering. Since our goal is to return large files and save the extra overhead of keeping an in-memory copy, having ASP.Net buffer the output would invalidate any solution we come up with. The FileStreamResult class constructor takes two parameters:

FileStream
:   This is the Stream that will be returned as the content of the HTTP response.

ContentType
:   This will be sent as the &#8220;content-type&#8221; HTTP header and should correspond to a type that the user agent (the Web browser) can use, like &#8220;image/jpg&#8221; for instance for a .JPG file. See <a href="http://en.wikipedia.org/wiki/Internet_media_type" target="_blank">Internet Media Types</a> for more details.

One can also set the optional FileDownloadName property. If set this will be sent as the &#8220;content-disposition&#8221; HTTP header and it is the proposed file name the user agent is offering the save the file as when the user chooses to save the return. Because the &#8220;attachment&#8221; tag is added to this header the browsers will always propose to _save_ the returned image, not dow display it. For more details on the <tt>content-disposition</tt> header, see <a href="http://www.ietf.org/rfc/rfc2183.txt" target="_blank">RFC2183</a>.

Now that the ASP.Net MVC plumbing is in place, all we need to do is actually stream out the image file from the database.

## A SqlDataReader based Stream

We now need an implementation of the abstract Stream class that can stream out a BLOB column from a SqlDataReader. Wait, you say, doesn&#8217;t SqlBytes already have a <a href="http://msdn.microsoft.com/en-us/library/system.data.sqltypes.sqlbytes.stream.aspx" target="_blank">Stream</a> property that reads a BLOB from a result as a Stream? Unfortunately this small remark makes this class useless for our purposes:

> Getting or setting the Stream property loads all the data into memory. Using it with large value data can cause an OutOfMemoryException.

So we&#8217;re left with implementing a Stream based on a SqlDataReader BLOB field, a Stream that returns the content of the BLOB using the proper GetBytes calls and does not load the entire BLOB in memory. Fortunately, this is fairly, as we really only need to implement a handfull of methods:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
    public class SqlReaderStream: Stream
    {
        private SqlDataReader reader;
        private int columnIndex;
        private long position;

        public SqlReaderStream(
            SqlDataReader reader,
            int columnIndex)
        {
            this.reader = reader;
            this.columnIndex = columnIndex;
        }

        public override long Position
        {
            get { return position; }
            set { throw new NotImplementedException(); }
        }

        public override int Read(byte[] buffer, int offset, int count)
        {
            long bytesRead = reader.GetBytes(columnIndex, position, buffer, offset, count);
            position += bytesRead;
            return (int)bytesRead;
        }

        public override bool CanRead
        {
            get { return true; }
        }

        public override bool CanSeek
        {
            get { return false; }
        }

        public override bool CanWrite
        {
            get { return false; }
        }

        public override void Flush()
        {
            throw new NotImplementedException();
        }

        public override long Length
        {
            get { throw new NotImplementedException(); }
        }

        public override long Seek(long offset, SeekOrigin origin)
        {
            throw new NotImplementedException();
        }

        public override void SetLength(long value)
        {
            throw new NotImplementedException();
        }

        public override void Write(byte[] buffer, int offset, int count)
        {
            throw new NotImplementedException();
        }

        protected override void Dispose(bool disposing)
        {
            if (disposing && null != reader)
            {
                reader.Dispose();
                reader = null;
            }
            base.Dispose(disposing);
        }
    }
&lt;/pre>
&lt;p></code>

As you can see, we need only to return the proper responses to CanRead (yes), CanWrite (no) and CanSeek (also no), keep track of our current Position and we need implement Read to fetch more bytes from the reader, using GetReader.

We&#8217;re also overriding the <tt>Dispose(bool disposing)</tt> method. This is because we&#8217;re going to have to close the SqlDataReader when the content stream transfer is complete. If this is the first time you see this signature of the Dispose method, then you must read <a href="http://msdn.microsoft.com/en-us/library/fs2xkftw.aspx" target="_blank">Implementing a Dispose Method</a>

## The MEDIA table

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
create table media (
	[media_id] int not null identity(1,1),
	[file_name] varchar(256),
	[content_type] varchar(256),
	[content_coding] varchar(256),
	[content] varbinary(max),
        constraint pk_media_id primary key([media_id]),
	constraint unique_file_name unique ([file_name]));
&lt;/pre>
&lt;p></code>

This table contains the downloadable media files. Files are identified by name, so the names have an unique constraint. I&#8217;ve added a IDENTITY primary key, because in a CMS these files are often references from other parts of the application and and INt is a shorter reference key than a file name. The <tt>content_type</tt> field is needed to know the type of file: &#8220;image/png&#8221;, &#8220;image/jpg&#8221; etc (see the officially registered <a href="http://www.iana.org/assignments/media-types/" target="_blank">IANA Media types</a>. The <tt>content_coding</tt> field is needed if the files are stored compressed and a <tt>Content-Encoding</tt> HTTP header with the tag &#8220;gzip&#8221; or &#8220;deflate&#8221; has to be added to the response. Note that most image types don&#8217;t compress well, as the 

## To BLOB or Not to BLOB

This article does not try to address the fundamental question whether you should store the images in the database to start with. Arguments can be made both for and against this. Russel Sears, Catherine van Ingen and Jim Gray have published a research paper in 2006 on their performance comparison analysis between storing files in the database and storing them in the file system: <a href="http://research.microsoft.com/pubs/64525/tr-2006-45.pdf" target="_blank">To BLOB or Not To BLOB: Large Object Storage in a Database or a Filesystem?</a>. They concluded that:

> The study indicates that if objects are larger than one megabyte on average, NTFS has a clear advantage over SQL Server. If the objects are under 256 kilobytes, the database has a clear advantage. Inside this range, it depends on how write intensive the workload is, and the storage age of a typical replica in the system.

However their study was not comparing how to serve the files as HTTP responses. The fact that the web server can efficiently serve a file directly from the file system without any code being run in the web application changes the equation quite a bit, and tilts the performance strongly in favor of the file system. But if you already have considered the pros and cons and decided that the advantages of a consistent backup/restore and strong referential integrity warrants database stored BLOBs, then I hope this article highlights an efficient way to return those BLOBs as HTTP responses.