# Hadoop Data Ingestion

A Hadoop cluster doesn't exist in a vacuum. Typically, the need to ingest data 
from various datasources into the Hadoop cluster exists. This note collects and
reviews options for data ingestion into an Hadoop cluster.

In general, the setup is as simple as the following figure suggests:

	+--------------+               +---------------+
	|              |               |               |
	|              |               |               |
	|    source    +-------------->|     sink      |
	|              |               |               |
	|              |               |               |
	+--------------+               +---------------+
  
The `source` can take on different forms, including but not limited to the 
following:

* A [CSV file](http://tools.ietf.org/html/rfc4180) in the local filesystem
* Some kind of [constraint device](http://www.internet-of-things.eu/) such as a 
temperature sensor
* Application log files, for example from a [Web 
server](http://httpd.apache.org/docs/2.0/logs.html)
* Social media streams, such as the [Twitter 
firehose](https://dev.twitter.com/docs/streaming-apis)
* One or more tables in a relational database such as PostgreSQL or MySQL
* The Hadoop filesystem HDFS

The `sink` in the context of this note is any system that offers 
[HDFS](http://hadoop.apache.org/docs/stable/hdfs_design.html) 
compatible access.

## Table of Contents

###### 1. From [local filesystem](https://github.com/mhausenblas/hadoop-data-ingestion#from-local-filesystem)
* Command Line Interface - `hadoop fs -copyFromLocal`
* WebHDFS REST API
* Network File System

###### 2. From [dynamic sources](https://github.com/mhausenblas/hadoop-data-ingestion#from-dynamic-sources)
* Apache Flume
* Apache Kafka
* Facebook's Scribe

###### 3. From [relational databases](https://github.com/mhausenblas/hadoop-data-ingestion#from-relational-databases)
* Apache Sqoop

###### 4. From [HDFS](https://github.com/mhausenblas/hadoop-data-ingestion#from-hdfs)
* Command Line Interface - `distcp`

###### 5. [HDFS management](https://github.com/mhausenblas/hadoop-data-ingestion#hdfs-management)
* Configuration
* Common Commands
* Further Reading
 





----


## From local filesystem

### Command Line Interface (CLI)

If you have a file, such as a CSV file, in your local filesystem and want to
copy it to HDFS use `hadoop fs -copyFromLocal` like so:

	$ hadoop fs -copyFromLocal data.csv hdfs://localhost/michael/cli-data.csv

[documentation](http://hadoop.apache.org/docs/stable/file_system_shell.html#copyFromLocal) | 
[Stack Overflow](http://stackoverflow.com/questions/14704633/moving-data-to-hdfs-using-copyfromlocal-switch)

----

### WebHDFS REST API
If you prefer a RESTful interaction, there is the WebHDFS REST API available.
This API, available since Hadoop 0.20.205, must be enabled in other to use it. 

First, if WebHDFS is not yet enabled, edit `hdfs_site.xml` to change this:

	<property>
	 <name>dfs.webhdfs.enabled</name>
	 <value>true</value>
	</property>

Then, you can use for example `curl` to interaction with the WebHDFS API from
the shell. Let's first check if the CSV file we've uploaded in the previous
step using the CLI actually is available (the `-s` tells `curl` to be silent):
	
	$ curl -s GET "http://localhost:50070/webhdfs/v1/michael/cli-data.csv?op=GETFILESTATUS"
	{
	    "FileStatus": {
	        "accessTime": 1371472189412,
	        "blockSize": 67108864,
	        "group": "supergroup",
	        "length": 52,
	        "modificationTime": 1371472189412,
	        "owner": "mhausenblas2",
	        "pathSuffix": "",
	        "permission": "644",
	        "replication": 1,
	        "type": "FILE"
	    }
	}

OK, this looks fine. Now let's create a second copy of the CSV file in HDFS,
this time we call it `rest-data.csv` (note the `-v` here, we ask `curl` to
be chatty this time as we want to see exactly what is happening):
	
	$ curl -v -X PUT "http://localhost:50070/webhdfs/v1/michael/rest-data.csv?op=CREATE"
	* About to connect() to localhost port 50070 (#0)
	*   Trying ::1...
	* connected
	* Connected to localhost (::1) port 50070 (#0)
	> PUT /webhdfs/v1/michael/rest-data.csv?op=CREATE HTTP/1.1
	> User-Agent: curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8x zlib/1.2.5
	> Host: localhost:50070
	> Accept: */*
	>
	< HTTP/1.1 307 TEMPORARY_REDIRECT
	< Content-Type: application/octet-stream
	< Location: http://10.196.2.36:50075/webhdfs/v1/michael/rest-data.csv?op=CREATE&overwrite=false
	< Content-Length: 0
	< Server: Jetty(6.1.26)
	<
	* Connection #0 to host localhost left intact
	* Closing connection #0

So, the namenode confirms our attempt to create the file and internally makes
sure the metadata is provisioned. Then, the API redirects us - the 
[307 response code](http://tools.ietf.org/html/rfc2616#section-10.3.8) -
to a datanode where the file is to be written. So, in a second step we can now 
actually write the file to the location specified in the previous HTTP response:

	$ curl -v -X PUT -T data.csv "http://10.196.2.36:50075/webhdfs/v1/michael/rest-data.csv?op=CREATE&user.name=mhausenblas2"
	* About to connect() to 10.196.2.36 port 50075 (#0)
	*   Trying 10.196.2.36...
	* connected
	* Connected to 10.196.2.36 (10.196.2.36) port 50075 (#0)
	> PUT /webhdfs/v1/michael/rest-data.csv?op=CREATE&user.name=mhausenblas2 HTTP/1.1
	> User-Agent: curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8x zlib/1.2.5
	> Host: 10.196.2.36:50075
	> Accept: */*
	> Content-Length: 52
	> Expect: 100-continue
	>
	< HTTP/1.1 100 Continue
	* We are completely uploaded and fine
	< HTTP/1.1 201 Created
	< Content-Type: application/octet-stream
	< Location: webhdfs://0.0.0.0:50070/michael/rest-data.csv
	< Content-Length: 0
	< Server: Jetty(6.1.26)
	<
	* Connection #0 to host 10.196.2.36 left intact
	* Closing connection #0

Note I had to append `user.name=mhausenblas2` in the request URL. This is 
necessary due to the fact that in the absence of any authentication
mechanism (such as Kerberos), the default user privileges on HDFS are 
mirrored from the local user. 

And just to confirm that the operation actually was successful let's use the
CLI to verify this:

	$ hadoop fs -ls /michael
	Found 2 items
	-rw-r--r--   1 mhausenblas2 supergroup         52 2013-06-17 13:29 /michael/cli-data.csv
	-rw-r--r--   1 mhausenblas2 supergroup         52 2013-06-17 13:50 /michael/rest-data.csv

[documentation](http://hadoop.apache.org/docs/stable/webhdfs.html)  | 
[Stack Overflow](http://stackoverflow.com/questions/11064229/hadoop-webhdfs-curl-create-file) | 
[example - Python](http://randomlydistributed.blogspot.ie/2011/12/webhdfs-py-simple-lean-hdfs-python.html)

----

### Network File System (NFS)

The [MapR Hadoop distribution](http://www.mapr.com/) lets you mount the
Hadoop cluster itself via Network File System ([NFS](http://tools.ietf.org/html/rfc1813) v3),
enabling your applications to directly write the data.


[documentation](http://www.mapr.com/doc/display/MapR/Accessing+Data+with+NFS) |
[example - Java](http://www.mapr.com/blog/leverage-existing-file-based-applications-with-hadoop)

----

## From dynamic sources

If you're dealing with data from dynamic sources such as:

* **constraint devices** in the [IoT](http://www.internet-of-things.eu/), for example
a temperature sensor, a light sensor, or a people counter,
* data streams from **governmental agencies** on topics like traffic or
[weather](http://data.gov.uk/dataset/metoffice_uklocs3hr_fc), 
* **log files** that are continuously updated, such as [Web 
server](http://httpd.apache.org/docs/2.0/logs.html) logs,
* **social media streams**, like the [Twitter 
firehose](https://dev.twitter.com/docs/streaming-apis) or Wikipedia edits via 
[IRC](http://meta.wikimedia.org/wiki/IRC/Channels#Raw_feeds)

then you have a number of options as listed below.

### Apache Flume

From the homepage:

> Flume is a distributed, reliable, and available service for efficiently 
> collecting, aggregating, and moving large amounts of log data. 
> It has a simple and flexible architecture based on streaming data flows. 
> It is robust and fault tolerant with tunable reliability mechanisms and 
> many failover and recovery mechanisms. 

[documentation](http://flume.apache.org/FlumeUserGuide.html) | 
[Dr. Dobbs article](http://www.drdobbs.com/database/acquiring-big-data-using-apache-flume/240155029)

----

### Apache Kafka

From the homepage:

> Apache Kafka is a distributed pub-sub messaging system designed for throughput.
> It is designed to support the following:
>  (i) persistent messaging with O(1) disk structures that provide constant 
>  time performance even with many TB of stored messages; 
>  (ii) high-throughput: even with very modest hardware Kafka can support hundreds
>  of thousands of messages per second; 
>  (iii) explicit support for partitioning messages over Kafka servers and
>  distributing consumption over a cluster of consumer machines while 
>  maintaining per-partition ordering semantics; 
>  (iv) support for parallel data load into Hadoop.


[documentation](https://cwiki.apache.org/confluence/display/KAFKA/Operations) | 
[MediaWiki infrastructure article](http://www.mediawiki.org/wiki/Analytics/Kraken/Request_Logging#Kafka)


----

### Facebook's Scribe

From the homepage:

> Scribe is a server for aggregating log data streamed in real time from a large
> number of servers. It is designed to be scalable, extensible without client-side
> modification, and robust to failure of the network or any specific machine.

[documentation](https://github.com/facebook/scribe/wiki) | 
[Stack Overflow](http://stackoverflow.com/questions/2241247/logging-data-with-scribe)

----

## From relational databases

### Apache Sqoop

From the homepage:

> Apache Sqoop is a tool designed for efficiently transferring bulk data 
> between Apache Hadoop and structured datastores such as relational databases.

See also (Sqoop connectors/plug-ins):

* Oracle's [MySQL Applier](https://blogs.oracle.com/MySQL/entry/announcing_the_mysql_hadoop_applier) for Apache Hadoop
* Quest (R) [Data Connector for Oracle and Hadoop](https://github.com/QuestSoftwareTCD/OracleSQOOPconnector)

[documentation](http://sqoop.apache.org/docs/1.4.3/index.html) | 
[Apache blog post](https://blogs.apache.org/sqoop/entry/apache_sqoop_overview)

----

## From HDFS

Sometimes, the data might originally reside in HDFS itself, for example, in case
of copying data form an old cluster into a new cluster. So, if you want
to ingest data from HDFS into HDFS, use `distcp`.

[documentation](http://hadoop.apache.org/docs/stable/distcp.html) | 
[Stack Overflow](http://stackoverflow.com/questions/15532575/does-hadoop-distcp-copy-replicas)

----

## HDFS management

### Configuration

The content of my `$HADOOP_HOME/conf/core-site.xml` for the operations 
shown above was:

	<configuration>
	 <property>
	  <name>fs.default.name</name>
	  <value>hdfs://localhost/</value>
	 </property>
	</configuration>

The content of my `$HADOOP_HOME/conf/hdfs-site.xml` for the operations 
shown above was:

	<configuration>
	 <property>
	  <name>dfs.name.dir</name>
	  <value>/Users/mhausenblas2/opt/hadoop/name/</value>
	 </property>
	 <property>
	  <name>dfs.data.dir</name>
	  <value>/Users/mhausenblas2/opt/hadoop/data/</value>
	 </property>
	 <property>
	  <name>dfs.replication</name>
	  <value>1</value>
	 </property>
	 <property>
	  <name>dfs.webhdfs.enabled</name>
	  <value>true</value>
	 </property>
	</configuration>

Note that this puts HDFS into [pseudo-distributed mode](http://hadoop.apache.org/docs/stable/single_node_setup.html#PseudoDistributed).

### Common Commands

To launch and shut down HDFS use `start-dfs.sh` and `stop-dfs.sh`.

To check the status of the HDFS daemons use `jps` (note: 
[HotSpot](http://docs.oracle.com/javase/6/docs/technotes/tools/share/jps.html)-specific):

	$ jps
	5166 Jps
	5104 SecondaryNameNode
	5012 DataNode
	4922 NameNode

To view the relevant logs live, use `tail -f`:

	tail -f /Users/mhausenblas2/bin/hadoop-1.0.4/logs/hadoop-mhausenblas2-namenode-Michaels-MacBook-Pro-2.local.log
	tail -f /Users/mhausenblas2/bin/hadoop-1.0.4/logs/hadoop-mhausenblas2-datanode-Michaels-MacBook-Pro-2.local.log

To view the status and health of the HDFS, point your browser to
[http://localhost:50070/dfshealth.jsp](http://localhost:50070/dfshealth.jsp)

Some common HDFS commands used throughout: 

* Initially, for a fresh install, run once `hadoop namenode -format`
* To list the contents of an HDFS directory run	`hadoop fs -ls /michael`

### Further Reading

Useful resources such as tutorials, introductions, background and deep-dives on HDFS:

* Yahoo! Developer Network (HTML) - [Module 2: The Hadoop Distributed File System](http://developer.yahoo.com/hadoop/tutorial/module2.html)
* YouTube (video, 33min) - [Hadoop Tutorial: Intro to HDFS](http://youtu.be/ziqx2hJY8Hg)
* Apache Hadoop (PDF) - [HDFS Architecture Guide](http://hadoop.apache.org/docs/stable/hdfs_design.pdf)
* Usenix 2010 article by Konstantin V. Shvachko (PDF) - [HDFS scalability: the limits to growth](http://static.usenix.org/publications/login/2010-04/openpdfs/shvachko.pdf)