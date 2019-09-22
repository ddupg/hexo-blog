---
layout:     post
title:      "「译」深入了解HBase体系结构"
subtitle:   ""
date:       2019-09-15 21:00:00
author:     "Hardbird"
header-img: "img/HBaseArchitecture/home-bg-o.jpg"
catalog: true
tags:
    - HBase
    - translation
---

[原文链接](https://mapr.com/blog/in-depth-look-hbase-architecture/)

> In this blog post, I’ll give you an in-depth look at the HBase architecture and its main benefits over NoSQL data store solutions. Be sure and read the first blog post in this series, titled
> “HBase and MapR Database: Designed for Distribution, Scale, and Speed.”

在这篇文章中，将深入介绍HBase体系结构，和它相较于NoSQL数据存储方案的主要优点。务必要阅读本系列的第一篇文章：[HBase and MapR Database: Designed for Distribution, Scale, and Speed.](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/#.VcKFNflVhBc)

## HBase体系结构

> Physically, HBase is composed of three types of servers in a master slave type of architecture. Region servers serve data for reads and writes. When accessing data, clients communicate with HBase RegionServers directly. Region assignment, DDL (create, delete tables) operations are handled by the HBase Master process. Zookeeper, which is part of HDFS, maintains a live cluster state.

HBase是由主从式体系结构中的三种服务器组成。RegionServer提供数据读写。当访问数据的时候，客户端直接与RegionServer交互。Region分配，DDL（表的创建删除）操作由HBase Master执行。Zookeeper维护集群状态。

> The Hadoop DataNode stores the data that the Region Server is managing. All HBase data is stored in HDFS files. Region Servers are collocated with the HDFS DataNodes, which enable data locality (putting the data close to where it is needed) for the data served by the RegionServers. HBase data is local when it is written, but when a region is moved, it is not local until compaction.

> Hadoop DataNode存储Region服务器管理的数据。所有的HBase数据存储在HDFS。RegionServer和管理HDFS的DataNode可以部署在同一批机器上（数据被就近存储）。HBase数据写入时在同一批机器，但如果region被迁移，要等到下一次compaction才会再在同一批机器上。

> The NameNode maintains metadata information for all the physical data blocks that comprise the files.

NameNode主要负责记录所有由文件组成的物理数据块的元信息。


![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig1.png)


## Regions
> HBase Tables are divided horizontally by row key range into “Regions.” A region contains all rows in the table between the region’s start key and end key. Regions are assigned to the nodes in the cluster, called “Region Servers,” and these serve data for reads and writes. A region server can serve about 1,000 regions.

Hbase表按rowkey水平切分形成Region。一个region包含表从startKey到endKey的所有行。Region被分配给集群中的RegionServer，提供该region的数据读写。一个RegionServer可以支持大约1000个region。

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig2.png)

## HBase HMaster
> Region assignment, DDL (create, delete tables) operations are handled by the HBase Master.

HBase Master负责region的分配，DDL（数据定义语言，表的增删）操作。

> A master is responsible for:
> Coordinating the region servers
> - Assigning regions on startup , re-assigning regions for recovery or load balancing
> - Monitoring all RegionServer instances in the cluster (listens for notifications from zookeeper)
> Admin functions
> - Interface for creating, deleting, updating tables

Master负责以下操作：

协调所有RegionServer
- 在启动时分配region，为恢复或负载均衡重新分配region
- 监控集群中所有RegionServer实例（监听zookeeper的通知）

管理功能
- 提供创建、删除、更新表的接口

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig3.png)

## ZooKeeper: The Coordinator
HBase uses ZooKeeper as a distributed coordination service to maintain server state in the cluster. Zookeeper maintains which servers are alive and available, and provides server failure notification. Zookeeper uses consensus to guarantee common shared state. Note that there should be three or five machines for consensus.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig4.png)

## How the Components Work Together
Zookeeper is used to coordinate shared state information for members of distributed systems. Region servers and the active HMaster connect with a session to ZooKeeper. The ZooKeeper maintains ephemeral nodes for active sessions via heartbeats.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig5.png)

Each Region Server creates an ephemeral node. The HMaster monitors these nodes to discover available region servers, and it also monitors these nodes for server failures. HMasters vie to create an ephemeral node. Zookeeper determines the first one and uses it to make sure that only one master is active. The active HMaster sends heartbeats to Zookeeper, and the inactive HMaster listens for notifications of the active HMaster failure.

If a region server or the active HMaster fails to send a heartbeat, the session is expired and the corresponding ephemeral node is deleted. Listeners for updates will be notified of the deleted nodes. The active HMaster listens for region servers, and will recover region servers on failure. The Inactive HMaster listens for active HMaster failure, and if an active HMaster fails, the inactive HMaster becomes active.

## HBase First Read or Write
There is a special HBase Catalog table called the META table, which holds the location of the regions in the cluster. ZooKeeper stores the location of the META table.

This is what happens the first time a client reads or writes to HBase:

The client gets the Region server that hosts the META table from ZooKeeper.
The client will query the .META. server to get the region server corresponding to the row key it wants to access. The client caches this information along with the META table location.
It will get the Row from the corresponding Region Server.
For future reads, the client uses the cache to retrieve the META location and previously read row keys. Over time, it does not need to query the META table, unless there is a miss because a region has moved; then it will re-query and update the cache.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig6.png)

## HBase Meta Table
This META table is an HBase table that keeps a list of all regions in the system.
The .META. table is like a b tree.
The .META. table structure is as follows:

- Key: region start key,region id
- Values: RegionServer

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig7.png)

## Region Server Components
A Region Server runs on an HDFS data node and has the following components:

WAL: Write Ahead Log is a file on the distributed file system. The WAL is used to store new data that hasn't yet been persisted to permanent storage; it is used for recovery in the case of failure.
BlockCache: is the read cache. It stores frequently read data in memory. Least Recently Used data is evicted when full.
MemStore: is the write cache. It stores new data which has not yet been written to disk. It is sorted before writing to disk. There is one MemStore per column family per region.
Hfiles store the rows as sorted KeyValues on disk.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig8.png)

## HBase Write Steps (1)
When the client issues a Put request, the first step is to write the data to the write-ahead log, the WAL:

- Edits are appended to the end of the WAL file that is stored on disk.
- The WAL is used to recover not-yet-persisted data in case a server crashes.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig9.png)

## HBase Write Steps (2)
Once the data is written to the WAL, it is placed in the MemStore. Then, the put request acknowledgement returns to the client.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig10.png)

## HBase MemStore
The MemStore stores updates in memory as sorted KeyValues, the same as it would be stored in an HFile. There is one MemStore per column family. The updates are sorted per column family.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig11.png)

## HBase Region Flush
When the MemStore accumulates enough data, the entire sorted set is written to a new HFile in HDFS. HBase uses multiple HFiles per column family, which contain the actual cells, or KeyValue instances. These files are created over time as KeyValue edits sorted in the MemStores are flushed as files to disk.

Note that this is one reason why there is a limit to the number of column families in HBase. There is one MemStore per CF; when one is full, they all flush. It also saves the last written sequence number so the system knows what was persisted so far.

The highest sequence number is stored as a meta field in each HFile, to reflect where persisting has ended and where to continue. On region startup, the sequence number is read, and the highest is used as the sequence number for new edits.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig12.png)

## HBase HFile
Data is stored in an HFile which contains sorted key/values. When the MemStore accumulates enough data, the entire sorted KeyValue set is written to a new HFile in HDFS. This is a sequential write. It is very fast, as it avoids moving the disk drive head.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig13.png)

## HBase HFile Structure
An HFile contains a multi-layered index which allows HBase to seek to the data without having to read the whole file. The multi-level index is like a b+tree:

Key value pairs are stored in increasing order
Indexes point by row key to the key value data in 64KB “blocks”
Each block has its own leaf-index
The last key of each block is put in the intermediate index
The root index points to the intermediate index
The trailer points to the meta blocks, and is written at the end of persisting the data to the file. The trailer also has information like bloom filters and time range info. Bloom filters help to skip files that do not contain a certain row key. The time range info is useful for skipping the file if it is not in the time range the read is looking for.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig14.png)

## HFile Index
The index, which we just discussed, is loaded when the HFile is opened and kept in memory. This allows lookups to be performed with a single disk seek.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig15.png)

## HBase Read Merge
We have seen that the KeyValue cells corresponding to one row can be in multiple places, row cells already persisted are in Hfiles, recently updated cells are in the MemStore, and recently read cells are in the Block cache. So when you read a row, how does the system get the corresponding cells to return? A Read merges Key Values from the block cache, MemStore, and HFiles in the following steps:

1. First, the scanner looks for the Row cells in the Block cache - the read cache. Recently Read Key Values are cached here, and Least Recently Used are evicted when memory is needed.
2. Next, the scanner looks in the MemStore, the write cache in memory containing the most recent writes.
3. If the scanner does not find all of the row cells in the MemStore and Block Cache, then HBase will use the Block Cache indexes and bloom filters to load HFiles into memory, which may contain the target row cells.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig16.png)

## HBase Read Merge
As discussed earlier, there may be many HFiles per MemStore, which means for a read, multiple files may have to be examined, which can affect the performance. This is called read amplification.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig17.png)

## HBase Minor Compaction
HBase will automatically pick some smaller HFiles and rewrite them into fewer bigger Hfiles. This process is called minor compaction. Minor compaction reduces the number of storage files by rewriting smaller files into fewer but larger ones, performing a merge sort.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig18.png)

## HBase Major Compaction
Major compaction merges and rewrites all the HFiles in a region to one HFile per column family, and in the process, drops deleted or expired cells. This improves read performance; however, since major compaction rewrites all of the files, lots of disk I/O and network traffic might occur during the process. This is called write amplification.

Major compactions can be scheduled to run automatically. Due to write amplification, major compactions are usually scheduled for weekends or evenings. Note that MapR Database has made improvements and does not need to do compactions. A major compaction also makes any data files that were remote, due to server failure or load balancing, local to the region server.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig19.png)

## Region = Contiguous Keys
Let’s do a quick review of regions:

A table can be divided horizontally into one or more regions. A region contains a contiguous, sorted range of rows between a start key and an end key
Each region is 1GB in size (default)
A region of a table is served to the client by a RegionServer
A region server can serve about 1,000 regions (which may belong to the same table or different tables)

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig20.png)

## Region Split
Initially there is one region per table. When a region grows too large, it splits into two child regions. Both child regions, representing one-half of the original region, are opened in parallel on the same Region server, and then the split is reported to the HMaster. For load balancing reasons, the HMaster may schedule for new regions to be moved off to other servers.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig21.png)

## Read Load Balancing
Splitting happens initially on the same region server, but for load balancing reasons, the HMaster may schedule for new regions to be moved off to other servers. This results in the new Region server serving data from a remote HDFS node until a major compaction moves the data files to the Regions server’s local node. HBase data is local when it is written, but when a region is moved (for load balancing or recovery), it is not local until major compaction.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig22.png)

## HDFS Data Replication
All writes and Reads are to/from the primary node. HDFS replicates the WAL and HFile blocks. HFile block replication happens automatically. HBase relies on HDFS to provide the data safety as it stores its files. When data is written in HDFS, one copy is written locally, and then it is replicated to a secondary node, and a third copy is written to a tertiary node.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig23.png)

## HDFS Data Replication (2)
The WAL file and the Hfiles are persisted on disk and replicated, so how does HBase recover the MemStore updates not persisted to HFiles? See the next section for the answer.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig24.png)

## HBase Crash Recovery
When a RegionServer fails, Crashed Regions are unavailable until detection and recovery steps have happened. Zookeeper will determine Node failure when it loses region server heart beats. The HMaster will then be notified that the Region Server has failed.

When the HMaster detects that a region server has crashed, the HMaster reassigns the regions from the crashed server to active Region servers. In order to recover the crashed region server’s memstore edits that were not flushed to disk. The HMaster splits the WAL belonging to the crashed region server into separate files and stores these file in the new region servers’ data nodes. Each Region Server then replays the WAL from the respective split WAL, to rebuild the memstore for that region.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig25.png)

## Data Recovery
WAL files contain a list of edits, with one edit representing a single put or delete. Edits are written chronologically, so, for persistence, additions are appended to the end of the WAL file that is stored on disk.

What happens if there is a failure when the data is still in memory and not persisted to an HFile? The WAL is replayed. Replaying a WAL is done by reading the WAL, adding and sorting the contained edits to the current MemStore. At the end, the MemStore is flush to write changes to an HFile.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig26.png)

## Apache HBase Architecture Benefits
HBase provides the following benefits:

Strong consistency model

- When a write returns, all readers will see same value
Scales automatically

- Regions split when data grows too large
- Uses HDFS to spread and replicate data
Built-in recovery

- Using Write Ahead Log (similar to journaling on file system)
Integrated with Hadoop

- MapReduce on HBase is straightforward
## Apache HBase Has Problems Too…
Business continuity reliability:

- WAL replay slow
- Slow complex crash recovery
- Major Compaction I/O storms
## MapR Database with MapR XD does not have these problems
The diagram below compares the application stacks for Apache HBase on top of HDFS on the left, Apache HBase on top of MapR's read/write file system MapR XD in the middle, and MapR Database and MapR XD in a Unified Storage Layer on the right.

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig27.png)

MapR Database exposes the same HBase API and the Data model for MapR Database is the same as for Apache HBase. However the MapR Database implementation integrates table storage into the MapR Distributed File and Object Store, eliminating all JVM layers and interacting directly with disks for both file and table storage.

MapR Academy
Apache HBase Data Model and Architecture
Explore the FREE Course
If you need to learn more about the HBase data model, we have 4 lessons that will help you. Module 2 of the FREE course, DEV 320 - Apache HBase Data Model and Architecture will walk you through everything you need to know.

MapR Database offers many benefits over HBase, while maintaining the virtues of the HBase API and the idea of data being sorted according to primary key. MapR Database provides operational benefits such as no compaction delays and automated region splits that do not impact the performance of the database. The tables in MapR Database can also be isolated to certain machines in a cluster by utilizing the topology feature of MapR. The final differentiator is that MapR Database is just plain fast, due primarily to the fact that it is tightly integrated into the MapR Distributed File and Object Store itself, rather than being layered on top of a distributed file system that is layered on top of a conventional file system.

## Key differences between MapR Database and Apache HBase

![pic](/img/HBaseArchitecture/HBaseArchitecture-Blog-Fig28.png)

- Tables part of the MapR Read/Write File system
    - Guaranteed data locality
- Smarter load balancing
    - Uses container Replicas
- Smarter fail over
    - Uses container replicas
- Multiple small WALs
    - Faster recovery
- Memstore Flushes Merged into Read/Write File System
    - No compaction !
You can take this free On Demand training to learn more about MapR XD and MapR Database

In this blog post, you learned more about the HBase architecture and its main benefits over NoSQL data store solutions. If you have any questions about HBase, please ask them in the comments section below.

## Want to learn more?
- [MapR provides Complete HBase Certification Curriculum as part of Free Hadoop On-Demand Training](https://mapr.com/training/courses/)
- [Installing HBase on MapR](http://doc.mapr.com/display/MapR/HBase?_ga=2.128978508.1380057432.1568551937-1506200283.1568551937)
- [Getting Started with HBase on MapR](http://doc.mapr.com/display/MapR/Working+with+HBase?_ga=2.133615310.1380057432.1568551937-1506200283.1568551937)
- [Release notes for HBase on MapR](http://doc.mapr.com/display/components/HBase+Release+Notes?_ga=2.133615310.1380057432.1568551937-1506200283.1568551937)
- [Apache HBase docs](https://issues.apache.org/jira/browse/HBASE/?report=com.atlassian.jira.jira-projects-plugin:roadmap-panel&selectedTab=com.atlassian.jira.jira-projects-plugin:summary-panel)
- [HBase: The Definitive Guide, by Lars George](http://shop.oreilly.com/product/0636920014348.do)
