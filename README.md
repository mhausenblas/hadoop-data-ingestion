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
* One or more tables in a relational database such as PostgreSQL or MySQL
* Some kind of [constraint device](http://www.internet-of-things.eu/) such as a 
temperature sensor
* Application log files, for example from a [Web 
server](http://httpd.apache.org/docs/2.0/logs.html)
* Social media stream such as the [Twitter 
firehose](https://dev.twitter.com/docs/streaming-apis)
* The Hadoop filesystem HDFS

The `sink` in the context of this note is any system that offers 
[HDFS](http://hadoop.apache.org/docs/stable/hdfs_design.html) 
compatible access.

## From the local filesystem

If you have a file, such as a CSV file, in your local filesystem and want to
copy it to HDFS use `hadoop fs -copyFromLocal` like so:

	hadoop fs -copyFromLocal /user/michael/data.csv /mhausenblas/data.csv

[documentation][HC] | [Stack Overflow][S1]

----

If you prefer a RESTful interaction, there is the WebHDFS REST API available.

This API, available since Hadoop 0.20.205, must be enabled in other to use it.
If not yet enabled, edit `hdfs_site.xml` to change this:

	<property>
	 <name>dfs.webhdfs.enabled</name>
	 <value>true</value>
	</property>

Then, you can use for example `curl` to interaction with the WebHDFS API from
the shell:
	
	
	curl -i GET "http://localhost:50070/webhdfs/v1/michael?op=GETHOMEDIRECTORY"
	
	curl -i -X PUT "http://localhost:50070/webhdfs/v1/michael/data.csv?op=CREATE"

[documentation][HW] | [Stack Overflow][S2] | [example Python script][E1]


## From dynamic sources
Flume, Scribe, Kafka, MapR's NFS

## From relational databases
Sqoop

## From HDFS
distcp


## HDFS management

Configuration

	content of `core-site.xml` and `hdfs-site.xml` - TBD

To launch and shut down HDFS:

	`start-dfs.sh` and `stop-dfs.sh`

Check status with `jps`:

	$ `jps`
	5166 Jps
	5104 SecondaryNameNode
	5012 DataNode
	4922 NameNode

Logging:

	tail -f /Users/mhausenblas2/bin/hadoop-1.0.4/logs/hadoop-mhausenblas2-namenode-Michaels-MacBook-Pro-2.local.log
	tail -f /Users/mhausenblas2/bin/hadoop-1.0.4/logs/hadoop-mhausenblas2-datanode-Michaels-MacBook-Pro-2.local.log



[HC]:(http://hadoop.apache.org/docs/stable/file_system_shell.html#copyFromLocal) 
[S1]:(http://stackoverflow.com/questions/14704633/moving-data-to-hdfs-using-copyfromlocal-switch)
[HW]:(http://hadoop.apache.org/docs/stable/webhdfs.html) 
[E1]:(http://randomlydistributed.blogspot.ie/2011/12/webhdfs-py-simple-lean-hdfs-python.html)
[S2]:(http://stackoverflow.com/questions/11064229/hadoop-webhdfs-curl-create-file)