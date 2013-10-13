FreeEedCollect
==============

Distributed collection of data for the purposes of eDiscovery and forensics.

Two modules are planned: 

1. MapReduce job for Hadoop: FreeEedCollectMR.
2. System for distributed data collection: FreeEedCollect.

First let us describe FreeEedCollectMR.

This module is based on the use of Apache ZooKeeper.

The MR job is given three parameters: URL of the ZooKeeper quorum, the top directory for data collection, and the 
desired output chunk size.

Each MapReduce Mapper does the following.

a) Grab a node in ZooKeeper with the path corresponding to the root. For namespace separation between the jobs, 
we use the JobID as the root for each MR job.

b) Set the state of the node to "In process". The first mapper that succeeds in grabbing this node continues the work,
while all others register to wait until the node becomes available.

c) The mapper that is active, having been able to grab a directory for processing and register this in ZooKeeper,
begins to traverse the director structure recursively (order to be determined later). When it comes to the next
directory that it is ready to evaluate, it grabs that one is ZooKeeper and releases the one is has been holding 
(we need to check for deadlock condition here). 

d) In each directory, the mapper quickly finds out the size (either the number of files or the total number of bytes, TBD).
Once it finds a directory that fits within the job's chunk size limit, it begins to process. For first pass, this means
determining and recording the important metadata parameters of each file. For staging, it means reading the file's
content (streaming, not all in memory), and writing it to HDFS in zipped format.

e) Having completed the directory, the mapper marks it in ZooKeeper as "completed" and is ready for the next piece of
work. That is, it begins traversal from the top again, following the rules (i) if the directory is marked is
ZooKeeper as "completed", continue; (ii) if the directory is "in processing" register to listen for when it becomes
available. If all are processed, it exits.

f) The job is finished when all mappers exit. 

Questions to be answered.

a) Is the algorithm complete? Do we have any deadlock or race conditions?
b) How do we verify that no files are lost in such processing?
