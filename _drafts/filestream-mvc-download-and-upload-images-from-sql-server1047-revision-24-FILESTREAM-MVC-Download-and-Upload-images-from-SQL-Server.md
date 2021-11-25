---
id: 2002
title: 'FILESTREAM MVC: Download and Upload images from SQL Server'
date: 2011-02-12T14:11:31+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/02/12/1047-revision-24/
permalink: /2011/02/12/1047-revision-24/
---
In a previous article I have shown how it is possible to use efficient streaming semantics when [Download and Upload images from SQL Server via ASP.Net MVC](http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/). In this article I will go over an alternative approach that relies on the <a href="http://technet.microsoft.com/en-us/library/bb933993.aspx" target="_blank"><tt>FILESTREAM</tt></a> column types introduced in SQL Server 2008.

### What is FILESTREAM?

<tt>FILESTREAM</tt> storage is a new option available in SQL Server 2008 and later that allows for BLOB columns to be stored directly on the file system as individual files. As files, the data is accessible through the Win32 file access API like <a href="http://msdn.microsoft.com/en-us/library/aa365467%28v=vs.85%29.aspx" target="_blank"><tt>ReadFile</tt></a> and <a href="http://msdn.microsoft.com/en-us/library/aa365747%28v=vs.85%29.aspx" target="_blank"><tt>WriteFile</tt></a>. But at the same time the same data is available through the normal T-SQL operations like <tt>SELECT</tt> or <tt>UPDATE</tt>. Not only that, but the data _is_ contained logically in the database so it will be contained in a database backup, it is subject to ordinary transaction commit and rollback behavior, it is searched by SQL Server FullText indexes and it follows the normal SQL Server security access rules: if you are granted SELECT permission on the table, then you can open the file to read. There are some restrictions, eg. a database with FILESTREAM cannot be mirrored. For a full list of restrictions and limitations, see <a href="http://technet.microsoft.com/en-us/library/bb895334.aspx" target="_blank">Using FILESTREAM with Other SQL Server Features</a>. Note that SQL Server Express edition _does_ support FILESTREAM storage.

Another attribute of the FILESTREAM storage is the size limitation: normal BLOB column values have a maximum size of 2Gb. FILESTREAM columns are limited only by the volume size limit of the file system. 2Gb may seem like a large value, but consider that a media file like an HD movie stream can easily go up to a size of 5Gb.

## Using FILESTREAM

One way to use FILESTREAM columns is to treat them as ordinary BLOB values and manipulate them through T-SQL. The one restriction in place is that the efficient partial update syntax for BLOBs is not supported. That is, one cannot issue <tt>UPDATE table SET column.WRITE(...) WHERE ... </tt> on a FILESTREAM column. But where FILESTREAM storage begins to shine is when accessed through the system file API. This allows the application to efficiently read, write and seek in a large BLOB value, just as it would in a file. In fact, the application _does_ read, write and seek in a file :).

Native Win32 applications use a new API function <a href="http://msdn.microsoft.com/en-us/library/bb933972.aspx" target="_blank"><tt>OpenSqlFilestream</tt></a> that opens a <tt>HANDLE</tt> that can be then used with the file IO API functions. Managed applications use the new <a href="http://msdn.microsoft.com/en-us/library/system.data.sqltypes.sqlfilestream.aspx" target="_blank"><tt>SqlFileStream</tt></a> class that exposes a Stream based on the underlying FILESTREAM value. Both the native and the manged API require as input two special values, a <a href="http://technet.microsoft.com/en-us/library/bb895239.aspx" target="_blank"><tt>PathName</tt></a> and a <a href="http://technet.microsoft.com/en-us/library/bb934014.aspx" target="_blank"><tt>Transaction Context</tt></a> that must be obtained previously from the SQL Server using T-SQL.

The requirement to provide a Transaction Context when manipulating a FILESTREAM value through the file IO API highlights another aspect of working with this new type: a T-SQL transaction must be started and must be kept open while the file is manipulated, and then it must be committed. If you think about it such a requirement is to be expected, since we said that FILESTREAM columns are subject to normal T-SQL transaction commit and rollback semantics, including when they are manipulated through the Windows file read/write API.

## FILESTREAM based images table

In order to have FILESTREAM column, we must have a special filegroup in our database, a FILESTREAM filegroup. You can either add a new filegroup to your existing database, or you can create the database with a FILESTREAM filegroup from scratch. Also the FILESTREAM feature must be enabled on the SQL Server instance. For all the details, see <a href="http://technet.microsoft.com/en-us/library/bb933995.aspx" target="_blank">Getting Started with FILESTREAM Storage</a>. For my project, I&#8217;m simply going to create a new database with a FILESTREAM filegroup:

<pre><code class="prettyprint lang-sql">
create database images 
	on (name='images_data', filename='c:\temp\images.mdf')
	, filegroup FS contains FILESTREAM 
              (name = 'images_files', filename='c:\temp\images_files')
	log on (name='images_log', filename='c:\temp\images.ldf');
go
</code>
</pre>

The media table used by our MVC project is going to be similar to the one used in the previous article but the <tt>content</tt> column will have the FILESTREAM attribute added. A table that has FILESTREAM columns is required to have a <a href="http://msdn.microsoft.com/en-us/library/aa933115%28v=sql.80%29.aspx" target="_blank"><tt>ROWGUIDCOL</tt></a> column, so we&#8217;re going to add one of those as well:

<pre><code class="prettyprint lang-sql">
create table media (
	[media_id] int not null identity(1,1),
	[file_name] varchar(256),
	[content_type] varchar(256),
	[content_coding] varchar(256),
	[media_rowguid] uniqueidentifier not null 
               ROWGUIDCOL UNIQUE default newsequentialid() ,
	[content] varbinary(max) filestream,
        constraint pk_media_id primary key([media_id]),
	constraint unique_file_name unique ([file_name]));
go
</code>
</pre>

## FILESTREAM based <span style="text-transform: none">IMediaRepository</span>

If you haven&#8217;t read the previous article [Download and Upload images from SQL Server via ASP.Net MVC](http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/) yet, now is a good time to do it. I am going to reuse the same code and simply provide a new implementation for the <tt>IMediaRepository</tt> interface, an implementation that works with FILESTREAM storage:

<pre><code class="prettyprint lang-sql">
    /// &lt;summary&gt;
    /// SQL Server FILESTREAM based implementation of the IMediaRepository
    /// &lt;/summary&gt;
    public class FileStreamMediaRepository: IMediaRepository
    {
        /// &lt;summary&gt;
        /// Gets an open connection to the SQL Server back end
        /// &lt;/summary&gt;
        /// &lt;returns&gt;the SqlConneciton object, ready to use&lt;/returns&gt;
        private SqlConnection GetConnection()
        {
            SqlConnectionStringBuilder scsb = new SqlConnectionStringBuilder(
                ConfigurationManager.ConnectionStrings["Images"].ConnectionString);
            scsb.Pooling = true;
            scsb.AsynchronousProcessing = true;
            SqlConnection conn = new SqlConnection(scsb.ConnectionString);
            conn.Open();
            return conn;
        }

        /// &lt;summary&gt;
        /// Gets a file from the SQL repository
        /// &lt;/summary&gt;
        /// &lt;param name="fileName"&gt;filename to retrieve&lt;/param&gt;
        /// &lt;param name="file"&gt;Output, the model for the file if found&lt;/param&gt;
        /// &lt;returns&gt;True if the file is found, False if not&lt;/returns&gt;
        public bool GetFileByName(string fileName, out FileDownloadModel file)
        {
            SqlConnection conn = GetConnection();
            SqlTransaction trn = conn.BeginTransaction();
            try
            {
                SqlCommand cmd = new SqlCommand(
                    @"SELECT file_name,
	                    content_type,
                        content_coding,
                        DATALENGTH (content) as content_length,
	                content.PathName() as path,
                        GET_FILESTREAM_TRANSACTION_CONTEXT ()
                    FROM media
                    WHERE file_name = @fileName;", conn, trn);
                SqlParameter paramFilename = new SqlParameter(
                            @"fileName", SqlDbType.VarChar, 256);
                paramFilename.Value = fileName;
                cmd.Parameters.Add(paramFilename);
                using (SqlDataReader reader = cmd.ExecuteReader())
                {
                    if (false == reader.Read())
                    {
                        reader.Close();
                        trn.Dispose();
                        conn.Dispose();
                        trn = null;
                        conn = null;
                        file = null;
                        return false;
                    }

                    string contentDisposition = reader.GetString(0);
                    string contentType = reader.GetString(1);
                    string contentCoding = reader.IsDBNull(2) ? null : reader.GetString(2);
                    long contentLength = reader.GetInt64(3);
                    string path = reader.GetString(4);
                    byte[] context = reader.GetSqlBytes(5).Buffer;

                    file = new FileDownloadModel
                    {
                        FileName = contentDisposition,
                        ContentCoding = contentCoding,
                        ContentType = contentType,
                        ContentLength = contentLength,
                        Content = new MvcResultSqlFileStream
                        {
                            SqlStream = new SqlFileStream(path, context, FileAccess.Read),
                            Connection = conn,
                            Transaction = trn
                        }
                    };
                    conn = null; // ownership transfered to the stream
                    trn = null;
                    return true;
                }
            }
            finally
            {
                if (null != trn)
                {
                    trn.Dispose();
                }
                if (null != conn)
                {
                    conn.Dispose();
                }
            }
        }

        /// &lt;summary&gt;
        /// Adds a file to the SQL repository
        /// &lt;/summary&gt;
        /// &lt;param name="file"&gt;POST-ed file to be added&lt;/param&gt;
        /// &lt;param name="fileName"&gt;The filename part of the uploaded file path&lt;/param&gt;
        public void PostFile(HttpPostedFileBase file, out string fileName)
        {
            fileName = Path.GetFileName(file.FileName);

            using (SqlConnection conn = GetConnection ())
            {
                using (SqlTransaction trn = conn.BeginTransaction ())
                {
                    SqlCommand cmdInsert = new SqlCommand(
@"insert into media 
    (file_name, content_type, content_coding, content)
output 
	INSERTED.content.PathName(),    
    GET_FILESTREAM_TRANSACTION_CONTEXT ()
values 
    (@content_disposition, @content_type, @content_coding, 0x)", conn, trn);
                    cmdInsert.Parameters.Add("@content_disposition", SqlDbType.VarChar, 256);
                    cmdInsert.Parameters["@content_disposition"].Value = fileName;
                    cmdInsert.Parameters.Add("@content_type", SqlDbType.VarChar, 256);
                    cmdInsert.Parameters["@content_type"].Value = file.ContentType;
                    cmdInsert.Parameters.Add("@content_coding", SqlDbType.VarChar, 256);
                    cmdInsert.Parameters["@content_coding"].Value = DBNull.Value;

                    string path = null;
                    byte[] context = null;

                    // cmdInsert is an INSERT command that uses the OUTPUT clause
                    // Thus we use the ExecuteReader to get the 
                    // result set from the output columns
                    //
                    using (SqlDataReader rdr = cmdInsert.ExecuteReader())
                    {
                        rdr.Read();
                        path = rdr.GetString(0);
                        context = rdr.GetSqlBytes(1).Buffer;
                    }

                    using (SqlFileStream sfs = new SqlFileStream(
                                     path, context, FileAccess.Write))
                    {
                        file.InputStream.CopyTo(sfs);
                    }
                    trn.Commit ();
                }
            }
        }
    }
</code>
</pre>

</code>

The implementation is pretty straight forward. The PostFile methods starts a transaction, inserts a row in the images table and obtains the PahtName and FILESTREAM transaction context to the newly inserted row, and then opens a SqlFileStream and copies in the uploaded file directly into the file that backs the FILESTREAM column. The GetFileByName method obtains the PathName and FILESTREAM transaction context of the desired file and then returns a SqlFileStream that accesses directly the file in which the FILESTREAM value is stored.

The only thing requiring explanation is the MvcResultSqlFileStream class. Because the SqlFileStream uses a transaction context, it is required to keep the SqlConnection and the SqlTransaction open until the SqlFileStream finishes accessing the content. In the PostFile method this requirement is achieved easily because the entire usage of the SqlFileStream complete within the stack frame, but the GetFileByName method has to return a Stream that in turn is passed into a FileStreamResult by the MediaController class and there is no control over how the MVC framework will use this returned ActionResult. My solution is to wrap the SqlFileStream in a new Stream derived class (since SqlFileStream is sealed, it cannot be inherited&#8230;) and then handle the resource management in this derived class&#8217; Dispose:

<pre><code class="prettyprint lang-sql">
    /// &lt;summary&gt;
    /// Implementation of System.IO.Stream based on a SqlFileStream
    /// Disposes the connection, transaction and SqlFileStream objects
    /// &lt;/summary&gt;
    public class MvcResultSqlFileStream: Stream
    {
        public SqlFileStream SqlStream { get; set; }
        public SqlConnection Connection { get; set; }
        public SqlTransaction Transaction { get; set; }
        public override bool CanRead
        {
            get { return SqlStream.CanRead; }
        }

        public override bool CanSeek
        {
            get { return SqlStream.CanSeek; }
        }

        public override bool CanWrite
        {
            get { return SqlStream.CanWrite; }
        }

        public override void Flush()
        {
            SqlStream.Flush ();
        }

        public override long Length
        {
            get { return SqlStream.Length; }
        }

        public override long Position
        {
            get
            {
                return SqlStream.Position;
            }
            set
            {
                SqlStream.Position = value;
            }
        }

        public override int Read(byte[] buffer, int offset, int count)
        {
            return SqlStream.Read (buffer, offset, count);
        }

        public override long Seek(long offset, SeekOrigin origin)
        {
            return SqlStream.Seek(offset, origin);
        }

        public override void SetLength(long value)
        {
            SqlStream.SetLength(value);
        }

        public override void Write(byte[] buffer, int offset, int count)
        {
            SqlStream.Write(buffer, offset, count);
        }

        protected override void Dispose(bool disposing)
        {
            if (disposing)
            {
                SqlStream.Dispose();
                Transaction.Dispose();
                Connection.Dispose();
            }
            base.Dispose(disposing);
        }
    }
</code>
</pre>

This extra class feels rather unnecessary, but unfortunately is required because the MVC framework has no proper handling of resources passed out from the controller. If you ask me ActionResult should implement IDisposable, but then of course nobody asks me 😉

You can download the source for the entire MVC project from [here](http://rusanu.com/wp-content/uploads/2011/02/MvcMedia.Filestream.zip).

## Conclusion

Handling media content stored in the database using FILESTREAM storage makes for a nicer developer experience. The FILESTREAM content can be accessed using streaming semantics based on the SqlFileStream class, or using the Win32 file IO API for native applications. By comparison the previous article that relied on straight T-SQL to manipulate the BLOB values had to do some very unintuitive tricks in order to achieve streaming semantics, specially for uploading a file. FILESTREAM storage simplifies development and is supported in every SQL Server edition, including Express.

If you intend to deploy FILESTREAM in your environment I recommend going over some Knowledge Base and white paper articles first:

  * <a href="http://msdn.microsoft.com/en-us/library/cc949109%28v=sql.100%29.aspx" target="_blank">FILESTREAM Storage in SQL Server 2008</a>
  * <a href="http://msdn.microsoft.com/en-us/library/dd283098.aspx" target="_blank">How to: Use FILESTREAM in a Firewall-Protected Environment</a>
  * <a href="http://support.microsoft.com/kb/2160002" target="_blank">Recommendations and guidelines for improving SQL Server FileStream performance</a>