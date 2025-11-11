---
id: 60
title: Remove pooling for data changes from a WCF front end
date: 2007-11-01T18:37:13+00:00
author: remus
layout: post
guid: /2007/11/01/remove-pooling-for-data-changes-from-a-wcf-front-end/
permalink: /2007/11/01/remove-pooling-for-data-changes-from-a-wcf-front-end/
categories:
  - Samples
---
The question was asked on the forums at <http://forums.microsoft.com/MSDN/ShowPost.aspx?PostID=2320807&SiteID=1>: How to enable a WCF based application to benefit from SQL Server 2005 Query Notifications? Basically the front end clients should be notified when data is changed on the back-end server tables as possible with least possible pooling.

Deploying SqlDependency in the WCF service itself to leverage the server&#8217;s Query Notification is a relatively easy process, simply follow the SqlDependency usage guidelines for any application: start SqlDependency, then attach a SqlDependency object instance to your SqlCommand objects, use this object&#8217;s OnChange event to react to changes, and subscribe again once notified.<!--more-->

  
This will provide the WCF service with a callback from the SqlDependency infrastructure when the database server tables have changed. The only thing challenging here is how to communicate from the WCF service back to the front end. The answer is a duplex channel and a callback that has to be supplied by the client. Here is how such a contract would be defined in WCF:

<pre><code class="prettyprint linenums">
    [ServiceContract(
        Name="WCFQNTableSubscribe",
        Namespace="/Samples/WCFQN",
        CallbackContract=typeof(IWCFQNTableCallback),
        SessionMode=SessionMode.Required)]
    public interface IWCFQNTableSubscribe
    {
        [OperationContract(IsOneWay=true)]
        void Subscribe(string cookie, string sql);
    }
    public interface IWCFQNTableCallback
    {
        [OperationContract(IsOneWay = true)]
        void Callback(string cookie, DataSet data);
    }
</code>
</pre>

The important things here are the CallbackContract attribute specifying the contract the WCF service should use to callback on the client and the IsOneWay attribute on the two contracts to specify an asynchronous bidirectional contract. Since we’re going to callback the client each time the data is changed, I thought to actually send the new dataset with each callback. The client subscribes to the data and does not get back any data in the Subscribe call, but the WCF service will immediately call him back on the supplied callback, providing the dataset with the current state of the data. After that each time the data is changed the WCF service will callback again the client with a new dataset. In my example the SQL query that is being submitted to retrieve the data is actually supplied by the client, in a real world scenario of course the client would request some business data, not some arbitrary SQL.

The WCF service runs the query, fills a dataset with the result set and calls back the client passing back the resulted dataset:

<pre><code class="prettyprint linenums">
        public void SubmitDataRequest()
        {
            SqlConnectionStringBuilder scsb = new SqlConnectionStringBuilder (
Settings.Default.connectionString);
            scsb.Pooling = true;
            SqlConnection conn = new SqlConnection(scsb.ConnectionString);
            conn.Open();
            SqlCommand cmdGetTable = new SqlCommand(_sql, conn);
            // Create the associated SqlDependency
            SqlDependency dep = new SqlDependency(cmdGetTable);
            dep.OnChange += new OnChangeEventHandler(dep_OnChange);
            DataSet ds = new DataSet();
            SqlDataAdapter da = new SqlDataAdapter(cmdGetTable);
            da.Fill(ds);
           // Send the new dataset back to the client
            // on it's own callback
            _callback.Callback(_cookie, ds);
        }
</code>
</pre>

When the SqlDependency is notified, all it does it invokes again this very same method, causing a new dataset to be sent back to the client (and in the process subscribing again to be notified):

<pre><code class="prettyprint linenums">
     void dep_OnChange(object sender, SqlNotificationEventArgs e)
     {
         if (e.Type == SqlNotificationType.Change)
         {
            // Submit the SQL back to the server,
            SubmitDataRequest();
         };
     }
</code>
</pre>

The client reacts to the callback by loading the supplied dataset, in my case the client simply attaches the dataset to a DataGridView object displayed on the form:

<pre><code class="prettyprint lang-sql">
       #region WCFQNTableSubscribeCallback Members
        public void Callback(Callback request)
        {
            Invoke(new CallbackInvokeHandler(CallbackInvoke), request.data);
        }
       #endregion
       private void CallbackInvoke(DataSet ds)
        {
            grdViewer.DataSource = ds;
            grdViewer.DataMember = ds.Tables[0].TableName;
        }
        private delegate void CallbackInvokeHandler(DataSet ds);
</code>
</pre>

Because UI elements on the form have to be updated on the main form thread, the callback in my example is calling using the Invoke method to pass the received dataset to the CallbackInvoke method and this one, in turn, attaches the grid to the new dataset.  
The details are in how to get a WCF duplex contract up and running. Not every binding supports such channels, in my example I use a wsDualHTTPBinding:

<pre><code class="prettyprint linenums">
  &lt;system.serviceModel&gt;
    &lt;services&gt;
      &lt;service name="wcfService.WCFQNTableSubscription"&gt;
        &lt;endpoint contract="wcfService.IWCFQNTableSubscribe" binding="wsDualHttpBinding"/&gt;
      &lt;/service&gt;
    &lt;/services&gt;
  &lt;/system.serviceModel&gt;
</code>
</pre>

There are other binding that support duplex contracts, I based my example on the wsDualHTTPBinding just like the MSDN example does: <http://msdn2.microsoft.com/en-us/library/ms731184>.  
I have uploaded my demo project here: </wp-content/uploads/2007/11/wcfqn.zip>.  
Update: I posted this on [github.com/rusanu/WCFQN](https://github.com/rusanu/WCFQN).