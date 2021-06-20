# Apache Kafka Cluster (3 Brokers) setup on Ubuntu

## Getting started

We will be following below steps to setup our Kafka Cluster on 3 Ubuntu machines (each with kafka and zookeeper). Note that, steps guided below needs to be replicated in other 2 machines with only `broker.id` and `advertised.listeners` changed across machines (example demonstrated)

1. Download the latest Kafka binaries
2. Install Java
3. Disable RAM swap
4. Create a directory with appropriate permissions for Kafka logs
5. Create a directory with appropriate permissions for Zookeeper snapshots
6. Specify an ID for the Zookeeper servers to reach quorum
7. Modify the Kafka configuration file to reflect the appropriate settings
8. Modify the Zookeeper configuration file to reflect the appropriate settings
9. Create Zookeeper as a service, so it can run without manual intervention
10. Create Kafka as a service, so it can run without manual intervention
11. Create a topic to test that the cluster is working properly


### Download the latest Kafka binaries
1. Use wget to download the tar file from the mirror:

```shell
wget http://mirror.cogentco.com/pub/apache/kafka/2.6.2/kafka_2.12-2.6.2.tgz
```
2. Extract the tar file and move it into the /opt directory:

```shell
tar -xvf kafka_2.12-2.6.2.tgz

sudo mv kafka_2.12-2.6.2 /opt/kafka
```
3. Extract the tar file and move it into the /opt directory:

```shell
cd /opt/kafka
ls
```

### Install Java and Disable RAM swap

1. Use the following command to install Java Developer Kit (As of now JDK 8 is used):

```shell
sudo apt install -y openjdk-8-jdk
```

2. Use the following command to verify that Java has been installed:

```shell
java -version
```

3. Use the following command to disable RAM swap:

```shell
swapoff -a
```

4. Use the following command to comment out swap in the `/etc/fstab` file:

```shell
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Create a New Directory for Kafka and Zookeeper

1. Use the following command to create a new directory for Kafka message logs:

```shell
sudo mkdir -p /data/kafka
```

2. Use the following command to create a snapshot directory for Zookeeper:

```shell
sudo mkdir -p /data/zookeeper
```
3. Change ownership of those directories to allow the user `ubuntu` control:

```shell
sudo chown -R ubuntu:ubuntu /data/kafka

# Give permission to Zookeeper snapshot directory
sudo chown -R ubuntu:ubuntu /data/zookeeper
```

### Specify an ID for Each Zookeeper Server

1. Use the following command to create a file in `/data/zookeeper` (on server #1) called `myid` with the contents "1" to specify Zookeeper server #1:

```shell
echo "1" > /data/zookeeper/myid
```

2. Use the following command to create a file in `/data/zookeeper` (on server #2) called `myid` with the contents "2" to specify Zookeeper server #2:

```shell
echo "2" > /data/zookeeper/myid
````

3. Use the following command to create a file in `/data/zookeeper` (on server #3) called `myid` with the contents "3" to specify Zookeeper server #3:

```shell
echo "3" > /data/zookeeper/myid
```

### Modify the Kafka and Zookeeper Configuration Files

1. Use the following command to remove the existing `server.properties` file (in the `config` directory) and create a new `server.properties` file:

```shell
rm config/server.properties

vi config/server.properties
```

2. Copy and paste the following into the contents of the `server.properties` file and change the `broker.id` and the `advertised.listeners`:

```shell
# change this for each broker
broker.id=[broker_number]   # example, broker.id=1 for server1, broker.id=2 for server2 and broker.id=3 for server 3 
# change this to the hostname of each broker
# example advertised.listeners=PLAINTEXT://kafka1:9092
advertised.listeners=PLAINTEXT://[hostname]:9092  # example, hostname -> kafka1 for server1, hostname -> kafka2 for server2 and hostname -> kafka3 for server 3 
# The ability to delete topics
delete.topic.enable=true
# Where logs are stored
log.dirs=/data/kafka
# default number of partitions
num.partitions=8
# default replica count based on the number of brokers
default.replication.factor=3
# to protect yourself against broker failure
min.insync.replicas=2
# logs will be deleted after how many hours
log.retention.hours=168
# size of the log files 
log.segment.bytes=1073741824
# check to see if any data needs to be deleted
log.retention.check.interval.ms=300000
# location of all zookeeper instances and kafka directory
zookeeper.connect=zookeeper1:2181,zookeeper2:2181,zookeeper3:2181/kafka
# timeout for connecting with zookeeper
zookeeper.connection.timeout.ms=6000
# automatically create topics
auto.create.topics.enable=true
````

3. Use the following command to remove the existing `zookeeper.properties` file (in the `config` directory) and create a new `zookeeper.properties` file:

```shell
rm config/zookeeper.properties

vi config/zookeeper.properties
````

4. Copy and paste the following into the contents of the `zookeeper.properties` file (don't change this file):

```shell
# the directory where the snapshot is stored.
dataDir=/data/zookeeper
# the port at which the clients will connect
clientPort=2181
# setting number of connections to unlimited
maxClientCnxns=0
# keeps a heartbeat of zookeeper in milliseconds
tickTime=2000
# time for initial synchronization
initLimit=10
# how many ticks can pass before timeout
syncLimit=5
# define servers ip and internal ports to zookeeper
server.1=zookeeper1:2888:3888
server.2=zookeeper2:2888:3888
server.3=zookeeper3:2888:3888
````

### Add custom hostnames to /etc/hosts file

1. Since we are creating 3 Ubuntu Servers with custom hostnames for kafka and zookeeper, we are required to add this ip to hostname mapping in `/etc/hosts` file

```shell
sudo vi /etc/hosts

<ip-of-server1>  kafka1
<ip-of-server1>  zookeeper1
<ip-of-server2>  kafka2
<ip-of-server2>  zookeeper2
<ip-of-server3>  kafka3
<ip-of-server3>  zookeeper3
```
### Create the Kafka and Zookeeper Service

1. Create the file `/etc/init.d/zookeeper` on each server and paste in the following contents:

```shell
sudo vi /etc/init.d/zookeeper
````

```shell
#!/bin/bash
#/etc/init.d/zookeeper
DAEMON_PATH=/opt/kafka/bin
DAEMON_NAME=zookeeper
# Check that networking is up.
#[ ${NETWORKING} = "no" ] && exit 0

PATH=$PATH:$DAEMON_PATH

case "$1" in
  start)
        # Start daemon.
        pid=`ps ax | grep -i 'org.apache.zookeeper' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
            echo "Zookeeper is already running";
        else
          echo "Starting $DAEMON_NAME";
          $DAEMON_PATH/zookeeper-server-start.sh -daemon /opt/kafka/config/zookeeper.properties
        fi
        ;;
  stop)
        echo "Shutting down $DAEMON_NAME";
        $DAEMON_PATH/zookeeper-server-stop.sh
        ;;
  restart)
        $0 stop
        sleep 2
        $0 start
        ;;
  status)
        pid=`ps ax | grep -i 'org.apache.zookeeper' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
          echo "Zookeeper is Running as PID: $pid"
        else
          echo "Zookeeper is not Running"
        fi
        ;;
  *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
esac

exit 0
````

2. Change the file to executable, change ownership, install, and start the service:

```shell
sudo chmod +x /etc/init.d/zookeeper
```
```shell
sudo chown root:root /etc/init.d/zookeeper
```
```shell
sudo update-rc.d zookeeper defaults
```
```shell
sudo service zookeeper start
```
```shell
sudo service zookeeper status
```

3. Create the file `/etc/init.d/kafka` on each server and paste in the following contents:

```shell
sudo vi /etc/init.d/kafka
````

```shell
#!/bin/bash
#/etc/init.d/kafka
DAEMON_PATH=/opt/kafka/bin
DAEMON_NAME=kafka
# Check that networking is up.
#[ ${NETWORKING} = "no" ] && exit 0

PATH=$PATH:$DAEMON_PATH

# See how we were called.
case "$1" in
  start)
        # Start daemon.
        pid=`ps ax | grep -i 'kafka.Kafka' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
            echo "Kafka is already running"
        else
          echo "Starting $DAEMON_NAME"
          $DAEMON_PATH/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
        fi
        ;;
  stop)
        echo "Shutting down $DAEMON_NAME"
        $DAEMON_PATH/kafka-server-stop.sh
        ;;
  restart)
        $0 stop
        sleep 2
        $0 start
        ;;
  status)
        pid=`ps ax | grep -i 'kafka.Kafka' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
          echo "Kafka is Running as PID: $pid"
        else
          echo "Kafka is not Running"
        fi
        ;;
  *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
esac

exit 0
````

4. Change the file to executable, change ownership, install, and start the service:

```shell
sudo chmod +x /etc/init.d/kafka
```
```shell
sudo chown root:root /etc/init.d/kafka
```
```shell
sudo update-rc.d kafka defaults
```
```shell
sudo service kafka start
```
```shell
sudo service kafka start
```

### Create a Topic

1. Use the following command to create a topic named `awesome-kafka`:

```shell
./bin/kafka-topics.sh --zookeeper zookeeper1:2181/kafka --create --topic awesome-kafka --replication-factor 1 --partitions 3
```

2. Use the following command to describe the topic`:

```shell
./bin/kafka-topics.sh --zookeeper zookeeper1:2181/kafka --topic awesome-kafka --describe
```:

```shell
./bin/kafka-topics.sh --zookeeper zookeeper1:2181/kafka --create --topic awesome-kafka --replication-factor 1 --partitions 3
```

