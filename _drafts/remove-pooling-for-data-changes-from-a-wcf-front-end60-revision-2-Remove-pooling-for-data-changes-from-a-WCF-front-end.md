---
id: 2006
title: Remove pooling for data changes from a WCF front end
date: 2011-06-20T08:44:11+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/06/20/60-revision-2/
permalink: /2011/06/20/60-revision-2/
---
The question was asked on the forums at <http://forums.microsoft.com/MSDN/ShowPost.aspx?PostID=2320807&SiteID=1>: How to enable a WCF based application to benefit from SQL Server 2005 Query Notifications? Basically the front end clients should be notified when data is changed on the back-end server tables as possible with least possible pooling.

Deploying SqlDependency in the WCF service itself to leverage the server&#8217;s Query Notification is a relatively easy process, simply follow the SqlDependency usage guidelines for any application: start SqlDependency, then attach a SqlDependency object instance to your SqlCommand objects, use this object&#8217;s OnChange event to react to changes, and subscribe again once notified.<!--more-->

  
This will provide the WCF service with a callback from the SqlDependency infrastructure when the database server tables have changed. The only thing challenging here is how to communicate from the WCF service back to the front end. The answer is a duplex channel and a callback that has to be supplied by the client. Here is how such a contract would be defined in WCF:

<code class="prettyprint lang-sql">&lt;br />
    [ServiceContract(&lt;br />
        Name="WCFQNTableSubscribe",&lt;br />
        Namespace="http://rusanu.com/Samples/WCFQN",&lt;br />
        CallbackContract=typeof(IWCFQNTableCallback),&lt;br />
        SessionMode=SessionMode.Required)]&lt;br />
    public interface IWCFQNTableSubscribe&lt;br />
    {&lt;br />
        [OperationContract(IsOneWay=true)]&lt;br />
        void Subscribe(string cookie, string sql);&lt;br />
    }&lt;br />
    public interface IWCFQNTableCallback&lt;br />
    {&lt;br />
        [OperationContract(IsOneWay = true)]&lt;br />
        void Callback(string cookie, DataSet data);&lt;br />
    }&lt;br />
</code>  
The important things here are the CallbackContract attribute specifying the contract the WCF service should use to callback on the client and the IsOneWay attribute on the two contracts to specify an asynchronous bidirectional contract. Since we’re going to callback the client each time the data is changed, I thought to actually send the new dataset with each callback. The client subscribes to the data and does not get back any data in the Subscribe call, but the WCF service will immediately call him back on the supplied callback, providing the dataset with the current state of the data. After that each time the data is changed the WCF service will callback again the client with a new dataset. In my example the SQL query that is being submitted to retrieve the data is actually supplied by the client, in a real world scenario of course the client would request some business data, not some arbitrary SQL.

The WCF service runs the query, fills a dataset with the result set and calls back the client passing back the resulted dataset:  
<code class="prettyprint lang-sql">&lt;br />
        public void SubmitDataRequest()&lt;br />
        {&lt;br />
            SqlConnectionStringBuilder scsb = new SqlConnectionStringBuilder (&lt;br />
Settings.Default.connectionString);&lt;br />
            scsb.Pooling = true;&lt;br />
            SqlConnection conn = new SqlConnection(scsb.ConnectionString);&lt;br />
            conn.Open();&lt;br />
            SqlCommand cmdGetTable = new SqlCommand(_sql, conn);&lt;br />
            // Create the associated SqlDependency&lt;br />
            SqlDependency dep = new SqlDependency(cmdGetTable);&lt;br />
            dep.OnChange += new OnChangeEventHandler(dep_OnChange);&lt;br />
            DataSet ds = new DataSet();&lt;br />
            SqlDataAdapter da = new SqlDataAdapter(cmdGetTable);&lt;br />
            da.Fill(ds);&lt;br />
           // Send the new dataset back to the client&lt;br />
            // on it's own callback&lt;br />
            _callback.Callback(_cookie, ds);&lt;br />
        }&lt;br />
</code>  
When the SqlDependency is notified, all it does it invokes again this very same method, causing a new dataset to be sent back to the client (and in the process subscribing again to be notified):  
<code class="prettyprint lang-sql">&lt;br />
     void dep_OnChange(object sender, SqlNotificationEventArgs e)&lt;br />
     {&lt;br />
         if (e.Type == SqlNotificationType.Change)&lt;br />
         {&lt;br />
            // Submit the SQL back to the server,&lt;br />
            SubmitDataRequest();&lt;br />
         };&lt;br />
     }&lt;br />
</code>  
The client reacts to the callback by loading the supplied dataset, in my case the client simply attaches the dataset to a DataGridView object displayed on the form:  
<code class="prettyprint lang-sql">&lt;br />
       #region WCFQNTableSubscribeCallback Members&lt;br />
        public void Callback(Callback request)&lt;br />
        {&lt;br />
            Invoke(new CallbackInvokeHandler(CallbackInvoke), request.data);&lt;br />
        }&lt;br />
       #endregion&lt;br />
       private void CallbackInvoke(DataSet ds)&lt;br />
        {&lt;br />
            grdViewer.DataSource = ds;&lt;br />
            grdViewer.DataMember = ds.Tables[0].TableName;&lt;br />
        }&lt;br />
        private delegate void CallbackInvokeHandler(DataSet ds);&lt;br />
</code>  
Because UI elements on the form have to be updated on the main form thread, the callback in my example is calling using the Invoke method to pass the received dataset to the CallbackInvoke method and this one, in turn, attaches the grid to the new dataset.  
The details are in how to get a WCF duplex contract up and running. Not every binding supports such channels, in my example I use a wsDualHTTPBinding:  
<code class="prettyprint lang-sql">&lt;br />
  &lt;system.serviceModel&gt;&lt;br />
    &lt;services&gt;&lt;br />
      &lt;service name="wcfService.WCFQNTableSubscription"&gt;&lt;br />
        &lt;endpoint contract="wcfService.IWCFQNTableSubscribe" binding="wsDualHttpBinding"/&gt;&lt;br />
      &lt;/service&gt;&lt;br />
    &lt;/services&gt;&lt;br />
  &lt;/system.serviceModel&gt;&lt;br />
</code>  
There are other binding that support duplex contracts, I based my example on the wsDualHTTPBinding just like the MSDN example does: <http://msdn2.microsoft.com/en-us/library/ms731184>.  
I have uploaded my demo project here: <http://rusanu.com/wp-content/uploads/2007/11/wcfqn.zip>