**USING CONSENSUS TO BUILD DISTRIBUTED SYSTEMS: Part 1**

[TOC]

# Overview #
The goals of this assignment are to learn how to use consensus to build distributed systems by working with:

1. Zookeeper,  a consensus-based object-storage service widely used in industry;

2. Gigapaxos, a home-brew system that makes the replicated state machine (RSM) abstraction easy to use for third-party â€œblackboxâ€ services;

This assignment will require Linux and Java. The assignment is largely organized as step-by-step tutorials with almost no code to write except â€œhello worldâ€ code to verify that the system and its programming APIs work as expected. An overview of Zookeeper and GigaPaxos follows. 

***

#  Background #

### Zookeeper overview ###
ZooKeeper allows distributed applications to use a replicated server backend as a logically centralized datastore to create, read, write, or watch shared objects so that they can easily coordinate their actions. The replicated servers use a consensus protocol called ZAB (for ZooKeeper atomic broadcast) to replicate objects created by clients for improving fault-tolerance. ZAB differs from other consensus protocols like Paxos in two ways: First, it guarantees that servers via their primary-order consistency semantics preserve FIFO ordering for each client, i.e., requests issued by a single client (that is equivalent to a TCP connection) are executed by the servers in the order issued by the client. Amidst server or client failures, it is possible that servers only execute a prefix of requests issued by any given client, but re-ordering or â€œholesâ€ (or skipped requests) are guaranteed to not happen. Second, ZAB is a primary-backup protocol, i.e., only a single elected replica, the primary or the leader, executes a write and then propagates the state changes to the backup replicas. If the primary crashes, the backups by consensus elect a new leader that determines the most recent point up to which the most recent leader executed writes and continues from there onwards. The underlying mechanisms and the protocol overhead per request is roughly similar to Paxos (i.e., `n` ACCEPTs, `n` ACCEPT-ACKs, and `n` COMMITs per request at the servers with a replica group of size `n`) under graceful execution, but the details are quite different. Refer to the [ZooKeeper](https://www.usenix.org/conference/usenix-atc-10/zookeeper-wait-free-coordination-internet-scale-systems) and [ZAB](https://marcoserafini.github.io/papers/zab.pdf) papers for more details. 

The `watch` API is an event-driven mechanism for clients to be notified of the existence of or a change to some object in which they are interested. Issuing an object read with `watch=true` ensures that the client will be notified if it tries to later read that object and that object has since been modified. Furthermore, as different clients may be connected to different server replicas, and despite consensus, some replicas may be behind others, ZooKeeper also provides a `sync` API to flush all outstanding writes before processing a read. Otherwise, without the `sync` option, read operations by default are local, i.e., a client simply reads the current (possibly stale) value of the object from the server replica to which it is connected.

### GigaPaxos overview ###
GigaPaxos is a scalable fine-grained replicated state machine (RSM) system, i.e., it allows applications to easily create and manage a large number of separate RSMs. Clients can associate each service with a separate RSM of its own from a subset of a pool of server machines. Thus, different services may be replicated on different sets of machine in accordance with their fault tolerance or performance requirements. The underlying consensus protocol for each RSM is still Paxos, however it is carefully engineered so as to be extremely lightweight and fast. For example, each RSM uses only a few hundred bytes of memory when it is idle (i.e., not actively processing requests), so commodity machines can participate in millions of different RSMs. When actively processing requests, the overhead per request is similar to Paxos, but automatic batching of requests and Paxos messages significantly improves the throughput by reducing message overhead, especially when the number of different RSM groups is small. The lightweight API for creating and interacting with different RSMs allows applications to â€œcarelesslyâ€ create consensus groups on the fly for even small shared objects, e.g. a simple counter. GigaPaxos also has extensive support for reconfiguration, i.e., the membership of different RSMs can be programmatically by applications by writing their own policies.

GigaPaxos has a simple Replicable wrapper API that any black-box application can implement in order for it to be automatically be replicated and reconfigured as needed by GigaPaxos. This API requires three methods to be implemented: `execute(name, request), checkpoint(name)`, and `restore(name, state)`, to respectively execute a request, obtain a state checkpoint, or to roll back the state of a service called `name`, and additionally defines methods to serialize and deserialize application requests. For further details, refer to the [GigaPaxos](https://people.cs.umass.edu/~arun/papers/gigapaxos.pdf) paper.

### Summary of differences ###
Note that ZooKeeper does NOT enable a replicated state machine, i.e., applications can only use the ZooKeeper API to manage shared objects, and use its API to implemented several primitives (such as leader election, locks, barriers, etc.) easily, but Zookeeper is not designed to make a general-purpose stateful application (e.g., a database, file system, e-commerce application etc.) fault-tolerant. In order to use Zookeeper, or for that matter any object storage system in order to make a general-purpose stateful application fault-tolerant, the application developers would need to (1) cleanly isolate the safety-critical state of an application from the application logic itself; (2) represent that state as Zookeper's hierarchically organized data objects; (3) commit state changes upon write (or state-changing) to the Zookeeper ensemble and, depending on the application-desired consistency semantics, sychronize state as necessary upon read (or state-preserving) requests; and (4) upon recovering from a crash, have the application replica recover to a consistent copy of the state maintained by the Zookeeper ennsemble. A system enabling an RSM abstraction relieves application developers of the above burdens.

GigaPaxos allows arbitrary application services, essentially any application that can implement the `Replicable` API, to be deployed as an RSM. However, being a Paxos-based system, it does not preserve FIFO client request ordering like Zookeeper. With `k`-fold replication, it also incurs `k` times the execution cost of executing each request as any RSM does. GigaPaxos does not directly provide client APIs like `watch` or `sync`, however the `sync` functionality is implicit in GigaPaxos as it can be configured to linearize all requests or just a subset of requests (like writes in Zookeeper), and the `Replicable` API allows application servers and clients to define arbitrary request types as well as easily implement event-driven notification functionality as needed on top of GigaPaxos. GigaPaxos is agnostic to the representation of application state or application logic treating the application itself as a "black-box" and only relies on the application to implement the `Replicable` API for execution, checkpointing, restoration, and serialization/de-serialization.

*** 

# Activity 1: Test-driving Zookeeper #
There is a wealth of information on the [ZooKeeper Apache project website](http://zookeeper.apache.org/). We will follow the instructions at the [Getting Started](http://zookeeper.apache.org/doc/current/zookeeperStarted.html) page.

** Step-by-Step Activities: **

0. Download the latest stable version of Zookeeper as instructed on the page. You only need the binary, not the source for this assignment.

1. Follow the steps for the â€œ_Standalone Operation_â€ mode on localhost to familiarize yourself with unreplicated Zookeeper.

2. Next, install ZooKeeper on 5 different EC2 instances by following the â€œ_Running Replicated ZooKeeper_â€ instructions. 
	* It is also acceptable to do this step by running all 5 instances on `localhost` (instead of EC2) by modifying the `zoo.cfg` file to run the different zookeeper instances on different client-facing ports and designating different `dataDir` locations to them.
	
3. In the replicated mode when all servers are running, first verify that you can perform the command-line operations for `create, set, get`, and `delete` successfully, and further verify that
	
	a. when one or two servers crash, the operations continue to be successful; 
	
	b. when 3 or more servers crash, the command-line operations fail. 
	
4. **Capture screenshots of the server logs and client logs to show the two observations 3(a) and 3(b) noted above.**

5. Familiarize yourself with the Java ZooKeeper client by following the link in the â€œJava Exampleâ€ link in the menu on the left side of the page. The bottom of the page has the complete source listings (Executor.java and DataMonitor.java) for the code snippets explained on that page. Download these files and make sure that you have the required jar files `zookeeper-*.*.*.jar` and `log4j-*.*.*.jar` in your class path in order to be able to compile the above two source files. For the next step, ensure that your class path includes the `zookeeper` and `log4j` binaries as well as the `Executor` and `DataMonitor` classes. You can either export your `CLASSPATH` shell variable or explicitly supply the classpath using the `-cp` JVM argument option or set them up in your favorite IDE.

6. In two separate windows, run `Executor` respectively with the following arguments (assuming that you have previously created a znode called `/zk_test` in step 3 above). The first command below should print the value of `/zk_test` that will also be stored in the local file myfile, and the second will print nothing at this point.

	`java Executor host:port /zk_test myfile cat myfile`

	`java Executor host:port /watch_test1 test_file1 cat test_file1`

7. In another window, run the ZooKeeper client using `bin/zkCli.sh` and connect it to a server replica different from the server in step 5, and create a znode `/watch_test1` with the value `"test_value1"`. At this point, the second command in step 6 above will print `â€œtest_value1â€` and this value will also be stored in the local file `test_file1`. Now delete the znode `/watch_test1` and observe the output.

8. **Capture screenshots showing the output after step 7 in the two windows in step 6.**

9. Next, we will programmatically perform the `create, set`, and `delete` operations in step 7 above. For this, we only need to modify the `Executor` class and donâ€™t need `DataMonitor` anymore. We will use the field `Executor.zk` of type ZooKeeper to invoke its `create` method as shown below:  

	
            executor.zk.create("/watch_test1", "some_gibberish".getBytes(),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT,
                    new AsyncCallback.StringCallback() {

                @Override
                public void processResult(int rc, String path, Object ctx,
                                          String name) {
                    System.out.println("creation_result:[rc=" + rc + ", " +
                            "path=" + path + ", name=" + name + "]");
                }
            }, null);

10. Can you now use the `zk.setData, zk.getData`, and `zk.delete` respectively to write to, read from, and delete the znode `/watch_test1` and reproduce your observations in the manual step 7 above? 
11. **Include your Executor code using the above three functions in your submission.**


***



# Activity 2: Test-driving GigaPaxos #

** Step-by-Step GigaPaxos Activities **

1. Follow Tutorial 1 instructions at [gigapaxosâ€™ github page](https://github.com/MobilityFirst/gigapaxos) to run a simple â€œno-opâ€ GigaPaxos application. Note that both the localhost version and distributed version use replicated server mode with 3 servers as default.

	a. Modify the number of servers to 5 and verify, as above for ZooKeeper, that the system is live with one or two failures but not with 3 or more failures.

2. Complete Tutorials 2 and 3 with 5 servers (instead of the default 3). For tutorial 3, report the latency measured over at least 100 requests after ignoring at least the first 100 requests as warm-up.

3. **Include screenshots of the client output from each of the three tutorials showing that you have completed them**. Note that all 3 tutorials are being run on `localhost` (but they will also run as-is on physically distributed machines provided the server `host:port` addresses are set up correctly in the configuration properties file.



# Submission Instructions #

1. Submit 

	1. All the screenshots and the answer to the question at the end of Tutorial 3 in Gigapaxos as a single PDF file named TutorialActivities.pdf;  
 
	2. A single Zookeeper code file Executor.java from step 11 above. 

2. Do not impose any directory or package structure on the above two files.
