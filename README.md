# Hadoop Data Ingestion

A Hadoop cluster doesn't exist in a vacuum. Typically, the need to ingest data from various datasource exists.
This note collects and reviews options for data ingestion into an Hadoop cluster.

In general, the setup is as simple as the following figure suggests:

	+--------------+               +---------------+
	|              |               |               |
	|              |               |               |
	|    source    +-------------->|     sink      |
	|              |               |               |
	|              |               |               |
	+--------------+               +---------------+
  
The `source` can take on different forms, including but not limited to the following:

* A CSV file in the local filesystem
* One or more tables in a relational database such as PostgreSQL or MySQL
* Some kind of [constraint device](http://www.internet-of-things.eu/) such as a temperature sensor
* Application log files, for example from a [Web server](http://httpd.apache.org/docs/2.0/logs.html)
* Social media stream such as the [Twitter firehose](https://dev.twitter.com/docs/streaming-apis)
* The Hadoop filesystem HDFS

The `sink` in the context of this note is any system that offers [HDFS](http://hadoop.apache.org/docs/stable/hdfs_design.html) 
compatible access.
