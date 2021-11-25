---
id: 2000
title: Download and Upload images from SQL Server via ASP.Net MVC
date: 2010-12-29T16:41:16+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/12/29/978-revision-45/
permalink: /2010/12/29/978-revision-45/
---
A frequent question that pops up on discussion forums is how to serve an image that is stored in a SQL Server table from an ASP.Net application. Unfortunately the answer is almost always wrong, as the prevalent solution involves copying the **entire** image file into memory before returning it to the client. This solution works fine when tested with a light load and returning few small images. But in production environment the memory consumption required by all those image files stored as byte arrays in memory causes serious performance degradation. A good solution must use streaming semantics, transferring the data in small chunks from the SQL Server to the HTTP returned result.

The SqlClient components do offer streaming semantics for large result sets, including large BLOB fields, but the client has to specifically ask for it. The &#8216;secret ingredient&#8217; is the passing in the <a href="http://msdn.microsoft.com/en-us/library/system.data.commandbehavior.aspx" target="_blank"><tt>CommandBehavior.SequentialAccess</tt></a> flag to the SqlCommand.ExecuteReader:

> Provides a way for the DataReader to handle rows that contain columns with large binary values. Rather than loading the entire row, SequentialAccess enables the DataReader to load data as a stream. You can then use the GetBytes or GetChars method to specify a byte location to start the read operation, and a limited buffer size for the data being returned.

## An ASP.Net MVC virtual Media folder backed by SQL Server

Lets say we want to have a virtual Media folder in an ASP.Net MVC site, serving the files from a SQL Server database. A GET request for an URL like <tt>"http://site/Media/IMG0042.JPG"</tt> should return the content of the file named IMG0042.JPG from the database. A POST request to the URL <tt>"http://site/Media"</tt> which contains an embedded file should insert this new file in the database and redirect the response to the newly added file virtual path. This is how our upload HTML form looks like:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;form method="post" action="/Media" enctype="multipart/form-data"&gt;
&lt;input type="file" name="file" id="file"/&gt;
&lt;input type="submit" name="Submit" value="Submit"/&gt;
&lt;/form&gt;
&lt;/pre>
&lt;p></code>

We&#8217;ll start by adding a special route in Global.asax.cs that will act like a virtual folder for file download requests:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>            routes.MapRoute(
                "Media",
                "Media/{filename}",
                new { controller = "Media", action = "GetFile" },
                new { filename = @"[^/?*:;{}\\]+" });
&lt;/pre>
&lt;p></code>

Note that the upload will be handled by the default MVC route if we add an Index() method to our controller that will handle the POST.

For our GET requests we need a FileDownloadModel class to represent the requested file properties. for this sample, we don&#8217;t need a POST model since we&#8217;re only going to have have a single input field, the uploaded file. We&#8217;re going to use a Repository interface that declares two methods: GetFileByName returns from the repository a FileDownloadModel given a file name, and PutFile will accept an uploaded file and put it in the repository.

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
    public class FileDownloadModel
    {
        public string FileName {get; internal set;}
        public string ContentType { get; internal set; }
        public string ContentCoding { get; internal set; }
        public long ContentLength { get; internal set; }
        public Stream Content { get; internal set; }
    }

    public interface IMediaRepository
    {
        bool GetFileByName(
            string fileName,
            out FileDownloadModel file);

        void PostFile(
            HttpPostedFileBase file,
            out string fileName);
    }
&lt;/pre>
&lt;p></code>

With this Repository interface we can actually code our MediaController class:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
    public class MediaController : Controller
    {
        public IMediaRepository Repository { get; set; }

        public MediaController()
        {
            Repository = new SqlMediaRepository();
        }

        [HttpPost]
        public ActionResult Index()
        {
            string fileName;
            Repository.PostFile(Request.Files[0], out fileName);
            return new RedirectResult("/Media/" + fileName);
        }

        [HttpGet]
        public ActionResult GetFile(string fileName)
        {
            FileDownloadModel model;
            if (false == Repository.GetFileByName(
                fileName, 
                out model))
            {
                return new HttpNotFoundResult
                {
                    StatusDescription = String.Format(
                        "File {0} not found", 
                        fileName)
                };
            }

            if (null != model.ContentCoding)
            {
                Response.AddHeader(
                    "Content-Encoding", 
                    model.ContentCoding);
            }
            Response.AddHeader(
                "Content-Length", 
                model.ContentLength.ToString ());

            Response.BufferOutput = false;

            return new FileStreamResult(
                model.Content, 
                model.ContentType);
        }
    }
&lt;/pre>
&lt;p></code>

In have hard coded the Repository implementation to SqlMediaRepository, a class we&#8217;ll create shortly. A real project would probably use the Dependency Injection or Inversion of Control patterns, perhaps using <a href="http://stw.castleproject.org/" target="_blank">Castle Windsor</a> for example. For brevity, I will skip these details, there are plenty of blogs and articles describing how to do it.

Note the use of the <a href="http://msdn.microsoft.com/en-us/library/system.web.mvc.filestreamresult.aspx" target="_blank">FileStreamResult</a> return, which is an MVC supplied action for returning a download of an arbitrary Stream object. Which also brings us to the next point, we need to implement a Stream that read content from a SqlDataReader.

## A SqlDataReader based Stream

We now need an implementation of the abstract Stream class that can stream out a BLOB column from a SqlDataReader. Wait, you say, doesn&#8217;t SqlBytes already have a <a href="http://msdn.microsoft.com/en-us/library/system.data.sqltypes.sqlbytes.stream.aspx" target="_blank">Stream</a> property that reads a BLOB from a result as a Stream? Unfortunately this small remark makes this class useless for our purposes:

> Getting or setting the Stream property loads all the data into memory. Using it with large value data can cause an OutOfMemoryException.

So we&#8217;re left with implementing a Stream based on a SqlDataReader BLOB field, a Stream that returns the content of the BLOB using the proper GetBytes calls and does not load the entire BLOB in memory. Fortunately this is fairly simple as we only need to implement a handful of methods:

<pre>public class SqlReaderStream: Stream
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
</pre>

As you can see, we need only to return the proper responses to CanRead (yes), CanWrite (no) and CanSeek (also no), keep track of our current Position and we need implement Read to fetch more bytes from the reader, using GetReader.

We&#8217;re also overriding the <tt>Dispose(bool disposing)</tt> method. This is because we&#8217;re going to have to close the SqlDataReader when the content stream transfer is complete. If this is the first time you see this signature of the Dispose method, then you must read <a href="http://msdn.microsoft.com/en-us/library/fs2xkftw.aspx" target="_blank">Implementing a Dispose Method</a>

## Streaming Upload of BLOB data

Just like retrieving large BLOB data from SQL Server poses challenges to avoid creating full blown in-memory copies of the entire BLOB, similar issues arrive when attempting to insert a BLOB. The best solution is actually quite convoluted. It involves sending the data to the server in chunks, and using the in-place BLOB <a href="http://msdn.microsoft.com/en-us/library/ms177523.aspx" target="_blank">UPDATE</a> syntax. The MSDN has this to say in the Remarks section:

> Use the .WRITE (expression, @Offset, @Length) clause to perform a partial or full update of varchar(max), nvarchar(max), and varbinary(max) data types. For example, a partial update of a varchar(max) column might delete or modify only the first 200 characters of the column, whereas a full update would delete or modify all the data in the column. For best performance, we recommend that data be inserted or updated in chunk sizes that are multiples of 8040 bytes.

To implement such semantics, we will write a _second_ Stream implementation, this time for uploads:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
    public class SqlStreamUpload: Stream
    {
        public SqlCommand InsertCommand { get; set; }
        public SqlCommand UpdateCommand { get; set; }
        public SqlParameter InsertDataParam { get; set; }
        public SqlParameter UpdateDataParam { get; set; }

        public override bool CanRead
        {
            get { return false; }
        }

        public override bool CanSeek
        {
            get { return false; }
        }

        public override bool CanWrite
        {
            get { return true; }
        }

        public override void Flush()
        {
        }

        public override long Length
        {
            get { throw new NotImplementedException(); }
        }

        public override long Position
        {
            get; set;
        }

        public override int Read(byte[] buffer, int offset, int count)
        {
            throw new NotImplementedException();
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
            byte[] data = buffer;
            if (offset != 0 ||
                count != buffer.Length)
            {
                data = new byte[count];
                Array.Copy(buffer, offset, data, 0, count);
            }
            if (0 == Position &&
                null != InsertCommand)
            {
                InsertDataParam.Value = data;
                InsertCommand.ExecuteNonQuery();
            }
            else
            {
                UpdateDataParam.Value = data;
                UpdateCommand.ExecuteNonQuery();
            }
            Position += count;
        }
    }
&lt;/pre>
&lt;p></code>

This Stream implementation uses two SqlCommand objects: an InsertCommand to save the very first chunk, and an UpdateCommand to save the subsequent chunks. Note that the chunk sizes (the optimal 8040 bytes) is nowhere specified, that is easily achieved by wrapping the SqlStreamUpload in a <a href="http://msdn.microsoft.com/en-us/library/system.io.bufferedstream.aspx" target="_blank">BufferedStream</a> instance.

## The MEDIA table

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>create table media (
	[media_id] int not null identity(1,1),
	[file_name] varchar(256),
	[content_type] varchar(256),
	[content_coding] varchar(256),
	[content] varbinary(max),
        constraint pk_media_id primary key([media_id]),
	constraint unique_file_name unique ([file_name]));
&lt;/pre>
&lt;p></code> 

This table contains the downloadable media files. Files are identified by name, so the names have an unique constraint. I&#8217;ve added a IDENTITY primary key, because in a CMS these files are often references from other parts of the application and and INT is a shorter reference key than a file name. The <tt>content_type</tt> field is needed to know the type of file: &#8220;image/png&#8221;, &#8220;image/jpg&#8221; etc (see the officially registered <a href="http://www.iana.org/assignments/media-types/" target="_blank">IANA Media types</a>. The <tt>content_coding</tt> field is needed if the files are stored compressed and a <tt>Content-Encoding</tt> HTTP header with the tag &#8220;gzip&#8221; or &#8220;deflate&#8221; has to be added to the response. Note that most image types (JPG, PNG) don&#8217;t compress well, as these file formats already include a compression algorithm and when compressed again with the common HTTP transfer algorithms (gzip, deflate, compress) they usually _increase_ in size.

## The SqlMediaRepository

The final piece of the puzzle: an implementation of the IMediaRepository interface that uses a SQL Server back end for the files storage:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
    public class SqlMediaRepository: IMediaRepository
    {
        private SqlConnection GetConnection()
        {
            SqlConnectionStringBuilder scsb = new SqlConnectionStringBuilder(
                ConfigurationManager.ConnectionStrings["Images"].ConnectionString);
            scsb.Pooling = true;
            SqlConnection conn = new SqlConnection(scsb.ConnectionString);
            conn.Open();
            return conn;
        }

        public void PostFile(
            HttpPostedFileBase file,
            out string fileName)
        {
            fileName = Path.GetFileName(file.FileName);

            using (SqlConnection conn = GetConnection())
            {
                using (SqlTransaction trn = conn.BeginTransaction())
                {
                    SqlCommand cmdInsert = new SqlCommand(
                        @"INSERT INTO media (
                            file_name,
                            content_type,
                            content_coding,
                            content)
                        values (
                            @content_disposition,
                            @content_type,
                            @content_coding,
                            @data);", conn, trn);
                    cmdInsert.Parameters.Add("@data", SqlDbType.VarBinary, -1);
                    cmdInsert.Parameters.Add("@content_disposition", SqlDbType.VarChar, 256);
                    cmdInsert.Parameters["@content_disposition"].Value = fileName;
                    cmdInsert.Parameters.Add("@content_type", SqlDbType.VarChar, 256);
                    cmdInsert.Parameters["@content_type"].Value = file.ContentType;
                    cmdInsert.Parameters.Add("@content_coding", SqlDbType.VarChar, 256);
                    cmdInsert.Parameters["@content_coding"].Value = DBNull.Value;

                    SqlCommand cmdUpdate = new SqlCommand(
                            @"UPDATE media 
                            SET content.write (@data, NULL, NULL)
                            WHERE file_name = @content_disposition;", conn, trn);
                    cmdUpdate.Parameters.Add("@data", SqlDbType.VarBinary, -1);
                    cmdUpdate.Parameters.Add("@content_disposition", SqlDbType.VarChar, 256);
                    cmdUpdate.Parameters["@content_disposition"].Value = fileName;

                    using (Stream uploadStream = new BufferedStream(
                        new SqlStreamUpload
                        {
                            InsertCommand = cmdInsert,
                            UpdateCommand = cmdUpdate,
                            InsertDataParam = cmdInsert.Parameters["@data"],
                            UpdateDataParam = cmdUpdate.Parameters["@data"]
                        }, 8040))
                    {
                        file.InputStream.CopyTo(uploadStream);
                    }
                    trn.Commit();
                }
            }
        }

        public bool GetFileByName(string fileName, out FileDownloadModel file)
        {
            SqlConnection conn = GetConnection();
            try
            {
                SqlCommand cmd = new SqlCommand(
                    @"SELECT file_name,
	                    content_type,
                        content_coding,
                        DATALENGTH (content) as content_length,
	                    content
                    FROM media
                    WHERE file_name = @fileName;", conn);
                SqlParameter paramFilename = new SqlParameter(
                      @"fileName", SqlDbType.VarChar, 256);
                paramFilename.Value = fileName;
                cmd.Parameters.Add(paramFilename);
                SqlDataReader reader = cmd.ExecuteReader(
                    CommandBehavior.SequentialAccess |
                    CommandBehavior.SingleResult |
                    CommandBehavior.SingleRow |
                    CommandBehavior.CloseConnection);
                if (false == reader.Read())
                {
                    reader.Dispose();
                    conn = null;
                    file = null;
                    return false;
                }

                string contentDisposition = reader.GetString(0);
                string contentType = reader.GetString(1);
                string contentCoding = reader.IsDBNull(2) ? null : reader.GetString(2);
                long contentLength = reader.GetInt64(3);
                Stream content = new SqlReaderStream(reader, 4);

                file = new FileDownloadModel
                {
                    FileName = contentDisposition,
                    ContentCoding = contentCoding,
                    ContentType = contentType,
                    ContentLength = contentLength,
                    Content = content
                };
                conn = null; // ownership transfered to the reader/stream
                return true;
            }
            finally
            {
                if (null != conn)
                {
                    conn.Dispose();
                }
            }
        }
    }
&lt;/pre>
&lt;p></code>

Typically MVC applications tend to use a LINQ based Repository. In this case though I could not leverage LINQ because of the peculiar requirements of implementing efficient streaming for large BLOBs. So this Repository uses plain vanilla SqlClient code.

The GetFileByName method gets the one row from the <tt>media</tt> table and returns a FileDownloadModel with a SqlReaderStream object that is wrapping the SELECT command result. Note that I could not deploy the typical &#8220;using&#8221; pattern for the disposable SqlConnection because the connection must remain open until the command result SqlDataReader finishes streaming in the BLOB, and the streaming will be controlled by the MVC framework executing our ActionResult _after_ the GetFileByName is completed. The connection will be closed when the SqlReaderStream is disposed, because of the CommandBehavior.CloseConnection flag. The stream will be disposed by the <a href="http://aspnet.codeplex.com/SourceControl/changeset/view/61289#338558" target="_blank">FileStreamResult.WriteFile</a> method.

The PostFile method creates two SqlCommand statements, one to insert the first chunk of data along with the other relevant columns, and an update command that uses BLOB partial updates to write the subsequent chunks. A SqlStreamUpload object is then using these two SqlCommand to efficiently stream in the uploaded file. The intermediate BufferedStream is used to create upload chunks of the critical size of 8040 bytes (see above). If the content would have to be compressed, this is where it would happen, a <a href="http://msdn.microsoft.com/en-us/library/system.io.compression.gzipstream.aspx" target="_blank">GZipStream</a> would be placed in front of the Bufferedstream in order to compress the uploaded file, and an the &#8220;@content_coding&#8221; parameter would have to be set as to &#8220;gzip&#8221;.

## HTTP caching

I have left out from this implementation appropriate HTTP caching control. HTTP caching is extremely important in high volume traffic sites, there is no better way to optimize an HTTP request processing that to never receive said request at all and have the user-agent cache or some intermediate proxy serve a response instead. Our application would have to add appropriate ETag and Cache-Control HTTP headers and the MediaController would need to have an action for the HEAD requests. The IMediaRepository interface would need a new method to get all the file properties without the actual content. For the moment, I&#8217;ll leave these as an exercise for the reader&#8230;

## To BLOB or Not to BLOB

This article does not try to address the fundamental question whether you should store the images in the database to start with. Arguments can be made both for and against this. Russel Sears, Catherine van Ingen and Jim Gray have published a research paper in 2006 on their performance comparison analysis between storing files in the database and storing them in the file system: <a href="http://research.microsoft.com/pubs/64525/tr-2006-45.pdf" target="_blank">To BLOB or Not To BLOB: Large Object Storage in a Database or a Filesystem?</a>. They concluded that:

> The study indicates that if objects are larger than one megabyte on average, NTFS has a clear advantage over SQL Server. If the objects are under 256 kilobytes, the database has a clear advantage. Inside this range, it depends on how write intensive the workload is, and the storage age of a typical replica in the system.

However their study was not comparing how to serve the files as HTTP responses. The fact that the web server can efficiently serve a file directly from the file system without any code being run in the web application changes the equation quite a bit, and tilts the performance strongly in favor of the file system. But if you already have considered the pros and cons and decided that the advantages of a consistent backup/restore and strong referential integrity warrants database stored BLOBs, then I hope this article highlights an efficient way to return those BLOBs as HTTP responses.