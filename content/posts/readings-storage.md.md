---
title: "Essential Readings on Database Storage"
date: 2023-03-10T17:00:00+08:00
categories: ["Paper Reading"]
draft: false
---

## Disk I/O

- [Managing Non-Volatile Memory in Database Systems](https://db.in.tum.de/people/sites/vanrenen/papers/HyMem.pdf?lang=de), 2018, SIGMOD 
- [Design Tradeoffs of Data Access Methods](http://scholar.harvard.edu/files/stratos/files/rum-tutorial.pdf?m=1461167186), 2016, SIGMOD
- [Designing Access Methods: The RUM Conjecture](https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf), 2016, EDBT
- [The Five Minute Rule 20 Years Later and How Flash Memory Changes the Rules](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.227.3846&rep=rep1&type=pdf), 2008, ACM Queue
- [The 5 Minute Rule for Trading Memory for Disc Accesses and the 5 Byte Rule for Trading Memory for CPU Time](https://www.hpl.hp.com/techreports/tandem/TR-86.1.pdf), 1987, SIGMOD

## B Tree Families

- [Efficient Locking for Concurrent Operations on B-Trees](https://www.csd.uoc.gr/~hy460/pdf/p650-lehman.pdf), 1981, TODS
- [The Ubiquitous B-Tree](http://carlosproal.com/ir/papers/p121-comer.pdf), 1979

## Buffer Management

- [Virtual-Memory Assisted Buffer Management](https://www.cs.cit.tum.de/fileadmin/w00cfj/dis/_my_direct_uploads/vmcache.pdf), 2023, SIGMOD
- [Memory-Optimized Multi-Version Concurrency Control for Disk-Based Database Systems](https://db.in.tum.de/~freitag/papers/p2797-freitag.pdf), 2022, VLDB
- [Are You Sure You Want to Use MMAP in Your Database Management System?](https://db.cs.cmu.edu/papers/2022/cidr2022-p13-crotty.pdf), 2022, CIDR
- [Spitfire: A Three-Tier Buffer Manager for Volatile and Non-Volatile Memory](https://db.cs.cmu.edu/papers/2021/zhou-sigmod2021.pdf), 2021, SIGMOD
- [In-Memory Performance for Big Data](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43985.pdf), 2014, VLDB

## Log & Recover

- [Rethinking Logging, Checkpoints, and Recovery for High-Performance Storage Engines](https://db.in.tum.de/~leis/papers/rethinkingLogging.pdf), 2020, SIGMOD
- [FineLine: log-structured transactional storage and recovery](https://dbis.informatik.uni-kl.de/files/teaching/ws1819/seminar/protected/FineLine.pdf), 2018, VLDB
- [Scalable Logging through Emerging Non-Volatile Memory](http://www.vldb.org/pvldb/vol7/p865-wang.pdf), 2014, VLDB

## Codec & Compression

- [Mostly Order Preserving Dictionaries](https://15721.courses.cs.cmu.edu/spring2023/papers/05-compression/liu-icde2019.pdf "Mostly Order Preserving Dictionaries"), 2019, ICDE
- [Adaptive String Dictionary Compression in In-Memory Column-Store Database Systems](https://15721.courses.cs.cmu.edu/spring2023/papers/05-compression/muller-edbt2014.pdf "Adaptive String Dictionary Compression in In-Memory Column-Store Database Systems"), 2014, EDBT
- [Dictionary-based Order-preserving String Compression for Main Memory Column Stores](https://15721.courses.cs.cmu.edu/spring2023/papers/05-compression/p283-binnig.pdf "Dictionary-based Order-preserving String Compression for Main Memory Column Stores"), 2009, SIGMOD
- [Integrating Compression and Execution in Column-Oriented Database Systems](https://15721.courses.cs.cmu.edu/spring2023/papers/05-compression/abadi-sigmod2006.pdf "Integrating Compression and Execution in Column-Oriented Database Systems"), 2006, SIGMOD
- [How to Wring a Table Dry: Entropy Compression of Relations and Querying of Compressed Relations](https://15721.courses.cs.cmu.edu/spring2023/papers/05-compression/p858-raman.pdf "How to Wring a Table Dry: Entropy Compression of Relations and Querying of Compressed Relations"), 2006, VLDB 

## LSM Tree & Its Variants

- [Revisiting the Design of LSM-tree Based OLTP Storage Engine with Persistent Memory](http://vldb.org/pvldb/vol14/p1872-yan.pdf), 2021, VLDB
- [LSM-based Storage Techniques: A Survey](https://arxiv.org/pdf/1812.07527.pdf), 2019, VLDBJ
- [WiscKey: Separating Keys from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf), 2016, FAST
- [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf), 1996

### Learned Index

- [From WiscKey to Bourbon: A Learned Index for Log-Structured Merge Trees](http://pages.cs.wisc.edu/~yifann/bourbon-osdi20.pdf), 2020
- [Learning Multi-dimensional Indexes](https://arxiv.org/pdf/1912.01668.pdf), 2019
- [The Case for Learned Index Structures](https://www.cl.cam.ac.uk/~ey204/teaching/ACS/R244_2018_2019/papers/Kraska_SIGMOD_2018.pdf), 2018

### Compaction Optimizations

TODO

## Column Store

- [Column Sketches: A Scan Accelerator for Rapid and Robust Predicate Evaluation](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/hentschel-sigmod18.pdf "Column Sketches: A Scan Accelerator for Rapid and Robust Predicate Evaluation"), 2018, SIGMOD
- [BitWeaving: Fast Scans for Main Memory Data Processing](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/li-sigmod2013.pdf "BitWeaving: Fast Scans for Main Memory Data Processing"), 2013, SIGMOD 
- [Column Imprints: A Secondary Index Structure](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/p893-sidirourgos.pdf "Column Imprints: A Secondary Index Structure"), 2013, in SIGMOD
- [SQL Server Column Store Indexes](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/p1177-larson.pdf "SQL Server Column Store Indexes"), 2011, SIGMOD
- [Cache Conscious Indexing for Decision-Support in Main Memory](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/rao-vldb97.pdf "Cache Conscious Indexing for Decision-Support in Main Memory"), 1999, VLDB
- [Bitmap Index Design and Evaluation](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/p355-chan.pdf "Bitmap Index Design and Evaluation"), 1998, SIGMOD

## Distributed Storage

### Replication & Consistency

- [Ark: A Real-World Consensus Implementation](https://arxiv.org/pdf/1407.4765.pdf), 2014, CoRR

### Replica Placement

- [Adaptive HTAP through Elastic Resource Scheduling](https://dl.acm.org/doi/pdf/10.1145/3318464.3389783), 2020, SIGMOD
- [MorphoSys: Automatic Physical Design Metamorphosis for Distributed Database Systems](http://www.vldb.org/pvldb/vol13/p3573-abebe.pdf), 2020, VLDB
- [Autoscaling Tiered Cloud Storage in Anna](https://dl.acm.org/doi/pdf/10.14778/3311880.3311881), 2019, VLDB
- [Automated Demand-driven Resource Scaling in Relational Database-as-a-Service](http://www.audentia-gestion.fr/MICROSOFT/p883-das.pdf), 2016, SIGMOD

## Transaction & Concurrenct Conctrol

- [Scalable and Robust Snapshot Isolation for High-Performance Storage Engines](https://www.vldb.org/pvldb/vol16/p1426-alhomssi.pdf), 2022, VLDB
- [Memory-Optimized Multi-Version Concurrency Control for Disk-Based Database Systems](https://db.in.tum.de/~freitag/papers/p2797-freitag.pdf), 2022, VLDB
- [An Empirical Evaluation of In-Memory Multi-Version Concurrency Control](https://www.vldb.org/pvldb/vol10/p781-Wu.pdf), 2017, VLDB
- [Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf), 2012, VLDB
- [Serializable Isolation for Snapshot Databases](https://courses.cs.washington.edu/courses/cse444/08au/544M/READING-LIST/fekete-sigmod2008.pdf), 2009, TODS
- [Making Snapshot Isolation Serializable](https://dsf.berkeley.edu/cs286/papers/ssi-tods2005.pdf), 2005, TODS
- [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf), 1995, SIGMOD

## Systems

Leanstore:
- [LeanStore: In-Memory Data Management Beyond Main Memory](https://db.in.tum.de/~leis/papers/leanstore.pdf), 2018, ICDE

Umbra:
- [Umbra: A Disk-Based System with In-Memory Performance](http://cidrdb.org/cidr2020/papers/p29-neumann-cidr20.pdf), 2020, CIDR

Distributed NoSQL Systems:
- [PNUTS to Sherpa: Lessons from Yahoo!’s Cloud Database](http://www.vldb.org/pvldb/vol12/p2300-cooper.pdf), 2019, VLDB
- [Cassandra - A Decentralized Structured Storage System](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf), 2010, SOSP
- [PNUTS: Yahoo!’s Hosted Data Serving Platform](https://sites.cs.ucsb.edu/~agrawal/fall2009/PNUTS.pdf), 2008, VLDB
- [Dynamo: Amazon’s Highly Available Key-value Store](https://sites.cs.ucsb.edu/~agrawal/fall2009/dynamo.pdf), 2007, SOSP
- [Bigtable: A Distributed Storage System for Structured Data](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf), 2006, OSDI