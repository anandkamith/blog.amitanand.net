---
title: Enable HA for HiveServer2
categories: 
 - Hive
tags:
 - Hive
 - HIVEQL
 - SQL
 - Hiveserver2
 - HS2
comments: true
---
In this blog I am going to show you how to enable **High Availability** for **HiveServer2**. The **HiveServer2**, like **NameNode** that at present only allow 2 nodes, does not have any limit on number of servers that can be added to **HA** configuration. Also it doesn't provide any fail-over mechanism. A client will have to reconnect, in case a connection to current **HiveServer2** is lost. Below are the requirements to enable **High Availability** for **HiveServer2**.
<!--more-->

#### ZooKeeper
Zookeeper is used to store information about each **HS2** instance that is launched and registered with Zookeeper. A client, upon connection request, gets one of the registered **HS2** instance.

#### Multiple HiveServer2 instances 
To enable HA more than one instance of **HS2** is required. Multiple instances can run either on the same server with different port for each instance or on two different machines using same port number. If one of the instance fails, Zookeeper will return the next active **HS2** instance. Running multiple instances provides following benefits:
- High Availability
- Load Balancing
- Rolling Upgrade
> _It is also possible to have  **HS2** instances configured with different authentication scheme. For example, four instances can be configured with first two instances running on port 10000 using **Kerberos** authentication and other two running on port 10001 using **LDAP** authentication._

## Configuration

Let's look at the configuration that is needed for running multiple instances of **HS2**

| Parameter     | Description |
| ------------- |-------------|
| hive.zookeeper.quorum | The list of Zookeeper servers to talk to. |
| hive.zookeeper.session.timeout | Zookeeper client's session timeout. The client is disconnected, and as a result, all locks released, if a heartbeat is not sent in the timeout |
| hive.zookeeper.namespace | The parent node under which all Zookeeper nodes are created. |
| hive.server2.support.dynamic.service.discovery | Set to true to enable HiveServer2 dynamic service discovery for its clients |

##### Sample configuration:

Below configuration is deployed on two different hosts. HiveServer2 will be running on default port of 10000.

```XML
<property>
  <name>hive.zookeeper.quorum</name>
  <value>f-bcpc-vm1.bcpc.example.com:2181,f-bcpc-vm2.bcpc.example.com:2181,f-bcpc-vm3.bcpc.example.com:2181</value>
</property>
<property>
  <name>hive.zookeeper.session.timeout</name>
  <value>600000</value>
</property>
<property>
  <name>hive.zookeeper.namespace</name>
  <value>hiveserver2</value>
</property>
<property>
  <name>hive.server2.support.dynamic.service.discovery</name>
  <value>true</value>
</property>
```
At this stage after adding the configuration restart **HiveServer2** instance. Please do take a moment to look at **Znodes** created under `/hiveserver2` in **Zookeeper**.

After enabling HA one can connect to **HS** using syntax given below:
```bash
ubuntu@bcpc-vm8:~$ beeline -u  "jdbc:hive2://f-bcpc-vm1.bcpc.example.com:2181,f-bcpc-vm2.bcpc.example.com:2181,f-bcpc-vm3.bcpc.example.com:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;principal=hive/f-
bcpc-vm3.bcpc.example.com@BCPC.EXAMPLE.COM"
Connecting to jdbc:hive2://f-bcpc-vm1.bcpc.example.com:2181,f-bcpc-vm2.bcpc.example.com:2181,f-bcpc-vm3.bcpc.example.com:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;principal=hive/f-bcpc-vm3.bcpc.exam
ple.com@BCPC.EXAMPLE.COM
Connected to: Apache Hive (version 1.2.1.2.3.4.0-3485)
Driver: Hive JDBC (version 1.2.1.2.3.4.0-3485)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 1.2.1.2.3.4.0-3485 by Apache Hive
0: jdbc:hive2://f-bcpc-vm1.bcpc.example.com:2> select * from test limit 10;
+----------+-------------+--+
| test.id  | test.idval  |
+----------+-------------+--+
| id0001   | value0001   |
| id0002   | value0002   |
| id0003   | value0003   |
| id0004   | value0004   |
| id0005   | value0005   |
| id0006   | value0006   |
| id0007   | value0007   |
| id0008   | value0008   |
| id0009   | value0009   |
| id0010   | value0010   |
+----------+-------------+--+
10 rows selected (2.102 seconds)
```

In my next blog I will explain the architecture behind **Hive HA**.
