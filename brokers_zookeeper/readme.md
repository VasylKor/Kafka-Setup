

# Kafka Setup (Linux, On-Prem, No Docker, Multi-node Zookeeper and Brokers) 



## Objective

Deploying a high availability, fault tolerant Kafka Cluster where ZooKeeper cannot represent a single point of failure. 

[image] 

We will have a Broker and a ZooKeeper instance running on each machine to ensure fault tolerance and high availability. 



## Requirements (what I had at least)

- Raspberry Pi 4 (4GB) x3 with Raspbian OS
- Confluent Kafka version (check https://www.confluent.io for download)
- In LAN static IP for Pis
- Java11 (8 or higher is fine) installed on each Pi (`sudo apt install openjdk-11-jdk`)



## Setting up Pis

### 1.

After downloading Confluent files, for convenience set a `$KAFKA_HOME` env variable containing the path to the confluent folder containing `/bin, /etc, /lib and so on..` folders. 

Firstly type the following to modify `/.bashrc`. This file is executed at user log in.

```shell
$ nano ~/.bashrc
```

Secondly add at the bottom the command that sets your env variable. 

```shell
#kafka path  variable
export KAFKA_HOME=<path_to_confluent_folder>/confluent-7.2.2 # this is the name of my folder at the moment
```

Save, reboot (or run the same command in terminal) and you should be ready to call kafka scripts with the same command for every Pi. 



### 2.

Create a `/tmp` folder in a designated directory where to store kafka and zookeeper files. In my case is `$KAKFA_HOME`.

```shell
$ mkdir $KAFKA_HOME/tmp
```

Now create a folder inside `/tmp` to store ZooKeeper data. Name must match the `dataDir` variable we will set later on in ZooKeeper config file. 

```shell
$ mkdir $KAFKA_HOME/tmp/zookeeper
```





## Setting up ZooKeeper instances

Now we have everything ready, just need to configure ZooKeeper and (later) Brokers for each Pi. 



##### Firstly, in each Pi let's add a myid file in `$KAFKA_HOME/tmp/zookeeper`.

```shell
$ nano $KAFKA_HOME/tmp/zookeeper/myid
```

In the file write different ids.

```
1
```

For Pi1.

```
2
```

For Pi2.

```
3
```

For Pi3.

This `myid` allows the cluster to recognize each instance.



##### Secondly we configure the instances.

In `$KAFKA_HOME/etc/kafka` we got configuration files to pass to executables. 

Let's modify `$KAFKA_HOME/etc/kafka/zookeeper.properties` so that we can have multiple zookeeper instances running as a cluster.
Changing to proper IPs, add to the actual configuration the following lines:

```yaml
# It is used for heartbeats and timeouts. Rules Zookeeper time dependent operations. 
# Expressed in milliseconds.
tickTime=2000

# initLimit * tickTime is the time followers have to connect to the leader.
# syncLimit * tickTime is max time allowed for followers before becoming out of sync.
initLimit=5
syncLimit=2

# If autopurge.purgeInterval is positive integer it enables autopurge.
# Autopurge purges most recent snapshots and their transaction logs in dataDir
# except the last autopurge.snapRetainCount number of snapshots.
# autopurge.purgeInterval is expressed in hours of time between purges.
autopurge.purgeInterval=24
autopurge.snapRetainCount=3

# Migrate topic partition leadership before the broker is stopped.
controlled.shutdown.enable=true

# Here we specify the list of servers that host running ZooKeeper instances
# in the ZooKeeper cluster (or ensemble).
# The format is `server.<myid>=<hostname>:<leaderPort>:<electionPort>`
# the last two ports are needed for communication between the ensemble instances. 
server.1=192.168.1.122:2888:3888
server.2=192.168.1.123:2888:3888
server.3=192.168.1.124:2888:3888
```



Change the `dataDir` directory to the zookeper folder set up earlier.

```
dataDir=<full $KAFKA_HOME path>/tmp/zookeeper
```



The same changes can be applied to all Pis. 





## Setting up Kafka Brokers

Now it is time to configure the Broker instances.

Configuration is in `$KAFKA_HOME/etc/kafka/server.properties` and for **each Pi we have some differences**.

The content should be as follows. Comments and other are not mine. I can't find the source to give credit. 

 Remember to:

- put appropriate IPs and ports.
- change `broker.id` for each Pi.
- modify `logs.dir` with the full path to the `/tmp` folder chosen earlier.

```yaml
# Enable auto creation of topic on the server
# Default: true
auto.create.topics.enable=true

# The id of the broker. This must be set to a unique integer for each broker.
# If unset, a unique broker id will be generated.
# To avoid conflicts between zookeeper generated broker id's and user configured broker id's, generated broker ids start from reserved.broker.max.id + 1.
broker.id=1

# Specify the final compression type for a given topic.
# This configuration accepts the standard compression codecs ('gzip', 'snappy', 'lz4', 'zstd').
# It additionally accepts 'uncompressed' which is equivalent to no compression; and 'producer' which means retain the original compression codec set by the producer.
compression.type=producer

# Disable Confluent Support Metrics
# Default: true
confluent.support.metrics.enable=false

# Migrate topic partition leadership before the broker is stopped.
controlled.shutdown.enable=true

# Enables delete topic. Delete topic through the admin tool will have no effect if this config is turned off.
# Default: true
delete.topic.enable=true

# The amount of time the group coordinator will wait for more consumers to join a new group before performing the first rebalance.
# A longer delay means potentially fewer rebalances, but increases the time until processing begins.
# Default: 3000
group.initial.rebalance.delay.ms=3000

# Default replication factors for automatically created topics
# Default: 1
default.replication.factor=2

# The maximum allowed session timeout for registered consumers.
# Longer timeouts give consumers more time to process messages in between heartbeats at the cost of a longer time to detect failures.
# Default: 1800000
group.max.session.timeout.ms=900000

# The frequency in milliseconds that the log cleaner checks whether any log is eligible for deletion
# Default: 300000
log.retention.check.interval.ms=60000

# The number of hours to keep a log file before deleting it (in hours), tertiary to log.retention.ms property
# Default: 168
log.retention.hours=24

# The maximum size of a single log file
# Default: 1073741824
log.segment.bytes=67108864

# The amount of time to wait before deleting a file from the filesystem
# Default: 60000
log.segment.delete.delay.ms=1000

# The number of threads that the server uses for processing requests, which may include disk I/O
# Default: 8
num.io.threads=8

# The number of threads that the server uses for receiving requests from the network and sending responses to the network.
# Default: 3
num.network.threads=6

# The default number of log partitions per topic.
# Default: 1
num.partitions=1

# The number of threads per data directory to be used for log recovery at startup and flushing at shutdown
# Default: 1
num.recovery.threads.per.data.dir=4

# The replication factor for the offsets topic (set higher to ensure availability).
# Internal topic creation will fail until the cluster size meets this replication factor requirement.
# Default: 3
offsets.topic.replication.factor=3

# The SO_RCVBUF buffer of the socket server sockets. If the value is -1, the OS default will be used.
# OS default /proc/sys/net/core/rmem_default
# Default: 102400
socket.receive.buffer.bytes=102400

# The maximum number of bytes in a socket request
# Default: 104857600
socket.request.max.bytes=104857600

# The SO_SNDBUF buffer of the socket server sockets. If the value is -1, the OS default will be used.
# OS default /proc/sys/net/core/wmem_default
# Default: 102400
socket.send.buffer.bytes=102400

# Overridden min.insync.replicas config for the transaction topic.
# Default: 2
transaction.state.log.min.isr=2

# The replication factor for the transaction topic (set higher to ensure availability).
# Internal topic creation will fail until the cluster size meets this replication factor requirement.
# Default: 3
transaction.state.log.replication.factor=3

# Indicates whether to enable replicas not in the ISR set to be elected as leader as a last resort, even though doing so may result in data loss.
# Default: false
unclean.leader.election.enable=false

# The max time that the client waits to establish a connection to zookeeper. If not set, the value in zookeeper.session.timeout.ms is used.
# Default: null
zookeeper.connection.timeout.ms=6000

# The directories in which the log data is kept. If not set, the value in log.dir (Default: /tmp/kafka-logs) is used.
log.dirs=<path to tmp>/tmp/kafka-data

# Listener List - Comma-separated list of URIs we will listen on and the listener names.
# If the listener name is not a security protocol, listener.security.protocol.map must also be set.
# Specify hostname as 0.0.0.0 to bind to all interfaces. Leave hostname empty to bind to default interface.
# Examples of legal listener lists: PLAINTEXT://myhost:9092,SSL://:9091 CLIENT://0.0.0.0:9092,REPLICATION://localhost:9093
listeners=PLAINTEXT://0.0.0.0:9092

# Listeners to publish to ZooKeeper for clients to use, if different than the listeners config property.
# In IaaS environments, this may need to be different from the interface to which the broker binds.
# If this is not set, the value for listeners will be used.
# Unlike listeners it is not valid to advertise the 0.0.0.0 meta-address.
advertised.listeners=PLAINTEXT://192.168.1.122:9092

# Specifies the ZooKeeper connection string in the form hostname:port where host and port are the host and port of a ZooKeeper server.
# To allow connecting through other ZooKeeper nodes when that ZooKeeper machine is down you can also specify
# multiple hosts in the form hostname1:port1,hostname2:port2,hostname3:port3
zookeeper.connect=192.168.1.122:2181,192.168.1.123:2181,192.168.1.124:2181
```

 

### Extra

Make sure all the ports in the previous sections are allowed. If not just:

```shell
$ sudo ufw allow <port>
```





## Testing Time

Now, let's:

- Run all the instances;
- Create a topic;
- Start the producer console;
- Start the consumer console;
- Write messages in the producer console and watch them appear on the consumer side;
- Kill both services on one Pi, just Zookeeper or just a Broker and watch the cluster still functioning.

On each Pi:

- Zookeeper (to start all before brokers)

  ```shell
  $ $KAFKA_HOME/bin/zookeeper-server-start $KAFKA_HOME/etc/kafka/zookeeper.properties
  ```

- Brokers

  ```shell
  $ $KAFKA_HOME/bin/kafka-server-start $KAFKA_HOME/etc/kafka/server.properties
  ```

On one random Pi, create a topic:

```shell
$ $KAFKA_HOME/bin/kafka-topics --create --bootstrap-server 192.168.1.122:9092,192.168.1.123:9092,192.168.1.124:9092 --replication-factor 3 --partitions 1 --topic test
```

Open the Producer console:

```shell
$ $KAFKA_HOME/bin/kafka-console-producer --broker-list 192.168.1.122:9092,192.168.1.123:9092,192.168.1.124:9092 --topic test
```

Open the Consumer console:

```shell
$ $KAFKA_HOME/bin/kafka-console-consumer --bootstrap-server 1192.168.1.122:9092,192.168.1.123:9092,192.168.1.124:9092 --topic test
```



Enjoy! :D 