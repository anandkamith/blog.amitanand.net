---
title: Understanding Hive HA
date: 2016-07-16 14:08:48
comments: true
categories: 
 - Hive
tags:
 - Hive
 - SQL
 - Hiveserver2
 - HS2
---

In my blog {% link Enable HA for HiveServer2 http://blog.amitanand.net/2016/07/09/hive-ha/ %}, I talked about configuration that is needed to enable **High Availability (HA)** for **HiveServer2 (HS2)**. In this blog I want to take a deep dive into the architecture behind HA of HS2. Let’s start with understanding the core components involved.
<!--more-->

1. JDBC Client – A client application/user that uses a JDBC connection URL to talk to ZooKeeper to get connection information about one of the HS2. The connection string is in the form of
```
jdbc:hive2://<ZKS:PORT>/<DB>;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;
```
Where
* _ZKS_ is list of zookeeper servers separated by a comma
* _PORT_ is zookeeper client port
* _DB_ is the **Hive** Database you want to connect to

2. ZooKeeper – A state store that registers multiple instances of HS2 and returns one of them when a client requests for it. The **Zookeeper Znode** upon query will show something like
```bash
ubuntu@bcpc-vm1:~$ zookeeper-client -server f-bcpc-vm1.bcpc.example.com:2181 ls /HiveServer2
Connecting to f-bcpc-vm1.bcpc.example.com:2181

[serverUri=f-bcpc-vm2.bcpc.example.com:10000;version=1.2.1.2.3.4.0-3485;sequence=0000000007]
```

3. HiveServer2 – More than once instance of HS2 running on either same machine using different port number or running multiple machines using same/different port number

4. Hadoop Cluster – A client reads/writes data from after connecting to HS2.

Below is the step by step explanation of how a client session gets established with **HS2**

{% asset_img HiveHADesign.png %}

* Step 1
A JDBC client connects to ZooKeeper ensemble to discover currently registered HS2 instances using URL akin to
```
jdbc:hive2://<ZKS:PORT>/<DB>;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;
```


* Step 2 
ZK retrieves connection information for one of the registered HS2 and returns it to the client

* Step 3 
Upon receiving connection information from ZK, a client attempts to connect to HS2

* Step 4 
After making a connection to HS2, client continues to read/write data from Hadoop Cluster

### Unregister a HiveServer2 instance from Zookeeper

There are two ways to remove a registered **HS2** instance from **ZK**

##### Remove Znode from ZK
Launch the Zookeeper command line interface:
```
Zookeeper-client -server f-bcpc-vm1.bcpc.example.com:2181
```
Run the following command to list all registered instances
```
ls /HiveServer2
```
You should see something similar to:
```
[serverUri=f-bcpc-vm2.bcpc.example.com:10000;version=1.2.1.2.3.4.0-3485;sequence=0000000007]
[serverUri=f-bcpc-vm3.bcpc.example.com:10000;version=1.2.1.2.3.4.0-3485;sequence=0000000008]
```
To remove a particular registered instance, in ZK command line interface, execute following command:
```
delete /HiveServer2/serverUri=f-bcpc-vm2.bcpc.example.com:10000;version=1.2.1.2.3.4.0-3485;sequence=0000000007
```
##### Use HS2 deregister command
To remove all registered instances, execute the following command from the command line:
```
hive --service hiveserverf2 --deregister <version number>
```
Example:
```
hive --service hiveserver2 --deregister 1.2.1.2.3.4.0-3485
```

**Please note that after deregistering the HS2 from ZK, the deregistered HS2 will not be returned for any new client connections. However, any active client sessions are not impacted by this.**

