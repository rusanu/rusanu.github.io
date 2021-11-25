---
id: 988
title: Retrieving images from SQL Server database with ASP.Net MVC
date: 2010-12-24T21:51:22+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/12/24/978-revision-9/
permalink: /2010/12/24/978-revision-9/
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

Next we need to the <tt>MediaController</tt> that handles the request for files in the <tt>GetFile</tt> action. Since we want to return the file as a streaming response, the MediaController.GetFile method should return the content of the requested file. ASP.Net MVC already has an ActionResult implmenetation for an arbitrary stream, the <a href="http://msdn.microsoft.com/en-us/library/system.web.mvc.filestreamresult.aspx" target="_blank"><tt>FileStreamResult</tt></a>:  
<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
    public class MediaController : Controller
    {
        public ActionResult GetFile(string filename)
        {
            FileStreamResult result = new FileStreamResult();
            ...
            return result;
        }
    }
&lt;/pre>
&lt;p></code>

With this, the ASP.Net MVC plumbing is in place, now all we need to do is actually stream out the image file from the database. The FileStreamResult class requires three parameters:

ContentType
:   This will be sent as the &#8220;content-type&#8221; HTTP header and should correspond to a type that the user agent (the Web browser) can use, like &#8220;image/jpg&#8221; for instance for a .JPG file. See <a href="http://en.wikipedia.org/wiki/Internet_media_type" target="_blank">Internet Media Types</a> for more details.

FileDownloadName
:   Optional, if set this will be sent as the &#8220;content-disposition&#8221; HTTP header and it is the proposed file name the user agent is offering the save the file as when the user chooses to save the return. Unfortunately the MVC framework internally uses an <a href="http://msdn.microsoft.com/en-us/library/system.net.mime.contentdisposition.contentdisposition.aspx" target="_blank">ContentDisposition</a> instance but it does not expose it so one ends up with the default behavior, which means that the &#8220;attachment&#8221; tag is added and the net result is that browsers will always propose to _save_ the returned image. For more details on the content-disposition header, see <a href="http://www.ietf.org/rfc/rfc2183.txt" target="_blank">RFC2183</a>.

FileStream
:   This is the Stream that will be returned as the content of the HTTP response.