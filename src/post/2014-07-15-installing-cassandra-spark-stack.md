---
id: "2014-07-15-installing-cassandra-spark-stack"
title: "Installing the Cassandra / Spark OSS Stack"
abstract: "An overview of the steps required to install the Apache Cassandra + Spark stack on Linux."
tags: ["cassandra", "spark"]
pubdate: 2014-07-15T23:00:00Z
---

## Init

As mentioned in my [portacluster system imaging](/post/2014-07-14-portacluster-system-imaging.html) post,
I am performing this install on 1 admin node (node0) and 6 worker nodes (node\[1-6]) running 64-bit Arch Linux.
Most of what I describe in this post should work on other Linux variants with minor adjustments.

## Overview

When assembling an analytics stack, there are usually myriad choices to make. For this build, I decided to
build the smallest stack possible that lets me run Spark queries on Cassandra data. As configured it is
not highly available since the Spark master is standalone. (note: Datastax Enterprise Spark's master has
HA based on Cassandra). It's a decent tradeoff for portacluster, since
I can run the master on the admin node which doesn't get rebooted/reimaged constantly. I'm also going to
skip HDFS or some kind of HDFS replacement for now. Options I plan to look at later are GlusterFS's HDFS
adapter and [Pithos](http://pithos.io) as an S3 adapter. In the end, the stack is simply Cassandra and
Spark with the [spark-cassandra-connector](https://github.com/datastax/spark-cassandra-connector).

## Responsible Configuration

For this post I've used my [perl-ssh-tools](https://github.com/tobert/perl-ssh-tools) suite. The intent
is to show what needs to be done and one way to do it. For production deployments, I recommend using
your favorite configuration management tool.

perl-ssh-tools uses a configuration similar to dsh, which uses simple files with one
host per line. I use two lists below. Most commands run on the fleet of workers. Because
cl-run.pl provides more than ssh commands, it's also used to run commands on node0 using
its --incl flag e.g. `cl-run.pl --list all --incl node0`.

```
cat .dsh/machines.workers
node1
node2
node3
node4
node5
node6
```

`machines.all` is the same with node0 added.

## Install Cassandra

My first pass at this section involved setting up a package repo, but since I don't have time to package
Spark properly right now, I'm going to use the tarball distros of Cassandra and Spark to keep it simple.
[joschi](https://github.com/joschi) maintains a package on the [AUR](https://aur.archlinux.org/packages/cassandra/)
but I have chosen not to use it for this install.
I'm also using the Arch packages of OpenJDK, which isn't supported by Datastax, but works fine for hacking.
The JDK is pre-installed on my Arch image, it's as simple as `sudo pacman -S extra/jdk7-openjdk`.

First, I downloaded the Cassandra tarball from [apache.org](http://cassandra.apache.org/download/) to
node0 in /srv/public/tgz. Then on the worker nodes, it gets downloaded and expanded in /opt.

```
pkg="apache-cassandra-2.0.9-bin.tar.gz"
sudo curl -o /srv/public/tgz/$pkg \
  http://mirrors.gigenet.com/apache/cassandra/2.0.9/apache-cassandra-2.0.9-bin.tar.gz

cl-run.pl --list workers -c "curl http://node0/tgz/$pkg |sudo tar -C /opt -xzf -"
cl-run.pl --list workers -c "sudo ln -s /opt/apache-cassandra-2.0.9 /opt/cassandra"
```

To make it easier to do upgrades without regenerating the configuration, I
relocate the conf dir to /etc/cassandra to match what packages do. This assumes there
is no existing /etc/cassandra.

```
cl-run.pl --list workers -c "sudo mv /opt/cassandra/conf /etc/cassandra"
cl-run.pl --list workers -c "sudo ln -s /etc/cassandra /opt/cassandra/conf"
```

I will start Cassandra with a systemd unit, so I push that out as well. This unit
file runs Cassandra out of the tarball as the cassandra user with the stdout/stderr going
to the systemd journal (view with `journalctl -f`). I also included some
ulimit settings and bump the OOM score downwards to make it less likely that the kernel
will kill Cassandra when out of memory. Since we're going to be running two large JVM apps
on each worker node, this unit also enables cgroups so Cassandra can be given priority
over Spark. Finally, since the target machines have 16GB of RAM, the heap needs to be
set to 8GB (cassandra-env.sh calculates 3995M which is way too low).

```
cat > cassandra.service <<EOF
[Unit]
Description=Cassandra Tarball
After=network.target

[Service]
User=cassandra
Group=cassandra
RuntimeDirectory=cassandra
PIDFile=/run/cassandra/cassandra.pid
ExecStart=/opt/cassandra/bin/cassandra -f -p /run/cassandra/cassandra.pid
StandardOutput=journal
StandardError=journal
OOMScoreAdjust=-500
LimitNOFILE=infinity
LimitMEMLOCK=infinity
LimitNPROC=infinity
LimitAS=infinity
Environment=MAX_HEAP_SIZE=8G HEAP_NEWSIZE=1G CASSANDRA_HEAPDUMP_DIR=/srv/cassandra/log
CPUAccounting=true
CPUShares=1000

[Install]
WantedBy=multi-user.target
EOF

cl-sendfile.pl --list workers -x -l cassandra.service -r /etc/systemd/system/multi-user.target.wants/cassandra.service
cl-run.pl --list workers -c "sudo systemctl daemon-reload"
```

Since all Cassandra data is being redirected to /srv/cassandra and it's going to run as the
cassandra user, those need to be created.

```
cat > cassandra-user.sh <<EOF
mkdir -p /srv/cassandra/{log,data,commitlogs,saved_caches}
(grep -q '^cassandra:' /etc/group)  || groupadd -g 1234 cassandra
(grep -q '^cassandra:' /etc/passwd) || useradd -u 1234 -c "Apache Cassandra" -g cassandra -s /bin/bash -d /srv/cassandra cassandra
chown -R cassandra:cassandra /srv/cassandra
EOF

cl-run.pl --list workers -x -s cassandra-user.sh
```

## Configure Cassandra

Before starting Cassandra I want to make a few changes to the standard configurations. I'm not a big
fan of LSB so I redirect all of the /var files to /srv/cassandra so they're all in one place. There's
only one SSD in the target systems so the commit log goes on the same drive.

I configured portacluster nodes to have a bridge in front of the default interface, making br0 the default interface.

```
cat cassandra-config.sh
ip=$(ip addr show br0 |perl -ne 'if ($_ =~ /inet (\d+\.\d+\.\d+\.\d+)/) { print $1 }')

perl -i.bak -pe "
  s/^(cluster_name:).*/\$1 'Portable Cluster'/;
  s/^(listen|rpc)_address:.*/\${1}_address: $ip/;
  s|/var/lib|/srv|;
  s/(\s+-\s+seeds:).*/\$1 '192.168.10.11,192.168.10.12,192.168.10.13,192.168.10.14,192.168.10.15,192.168.10.16'/
" /opt/cassandra/conf/cassandra.yaml
# EOF

cl-run.pl --list workers -x -s cassandra-config.sh
```

The default log4-server.propterties has log4j printing to stdout. This is not desirable in a background
service configuration, so I remove it. The logs are also now written to /srv/cassandra/log.

```
cat > log4j-server.properties <<EOF
log4j.rootLogger=INFO,R
log4j.appender.R=org.apache.log4j.RollingFileAppender
log4j.appender.R.maxFileSize=20MB
log4j.appender.R.maxBackupIndex=20
log4j.appender.R.layout=org.apache.log4j.PatternLayout
log4j.appender.R.layout.ConversionPattern=%5p [%t] %d{ISO8601} %F (line %L) %m%n
log4j.appender.R.File=/srv/cassandra/log/system.log
log4j.logger.org.apache.thrift.server.TNonblockingServer=ERROR
EOF

cl-sendfile.pl --list workers -x -l log4j-server.properties -r /opt/cassandra/conf/log4j-server.properties
```

And with that, Cassandra is ready to start.

```
cl-run.pl --list workers -c "sudo systemctl start cassandra.service"
ssh node3 tail -f /srv/cassandra/log/system.log
```

## Installing Spark

The process for Spark is quite similar, except that unlike Cassandra, it has a master.

Since I'm not using any Hadoop components, any of the builds should be fine so I used the
hadoop2 build.

```
pkg="spark-1.0.1-bin-hadoop2.tgz"
sudo curl -o /srv/public/tgz/$pkg http://d3kbcqa49mib13.cloudfront.net/spark-1.0.1-bin-hadoop2.tgz

cl-run.pl --list all -c "curl http://node0/tgz/$pkg |sudo tar -C /opt -xzf -"
cl-run.pl --list all -c "sudo ln -s /opt/spark-1.0.1-bin-hadoop2 /opt/spark"
cl-run.pl --list all -c "sudo mv /opt/spark/conf /etc/spark"
cl-run.pl --list all -c "sudo ln -s /etc/spark /opt/spark/conf"
```

Create /srv/spark and the spark user.

```
cat > spark-user.sh <<EOF
mkdir -p /srv/spark/{logs,work,tmp,pids}
(grep -q '^spark:' /etc/group)  || groupadd -g 4321 spark
(grep -q '^spark:' /etc/passwd) || useradd -u 4321 -c "Apache Spark" -g spark -s /bin/bash -d /srv/spark spark
chown -R spark:spark /srv/spark
# make spark tmp world writable and sticky
chmod 4755 /srv/spark/tmp
EOF

cl-run.pl --list all -x -s spark-user.sh
```

## Configuring Spark

Many of Spark's settings are controlled by environment variables. Since I want all volatile data
in /srv, many of these need to be changed. Spark will pick up spark-env.sh automatically.

The Intel NUC systems I'm running this stack on have 4 cores and 16G of RAM, so I'll give
Spark 2 cores and 4G of memory for now.

One line worth calling out is the `SPARK_WORKER_PORT=9000`. It can be any port. If you don't set
it, every time a work is restarted the master will have a stale entry for a while. It's not
a big deal but I like it better this way.

```
cat > spark-env.sh <<EOF
export SPARK_WORKER_CORES="2"
export SPARK_WORKER_MEMORY="4g"
export SPARK_DRIVER_MEMORY="2g"
export SPARK_REPL_MEM="4g"
export SPARK_WORKER_PORT=9000
export SPARK_CONF_DIR="/etc/spark"
export SPARK_TMP_DIR="/srv/spark/tmp"
export SPARK_PID_DIR="/srv/spark/pids"
export SPARK_LOG_DIR="/srv/spark/logs"
export SPARK_WORKER_DIR="/srv/spark/work"
export SPARK_LOCAL_DIRS="/srv/spark/tmp"
export SPARK_COMMON_OPTS="$SPARK_COMMON_OPTS -Dspark.kryoserializer.buffer.mb=32 "
LOG4J="-Dlog4j.configuration=file://$SPARK_CONF_DIR/log4j.properties"
export SPARK_MASTER_OPTS=" $LOG4J -Dspark.log.file=/srv/spark/logs/master.log "
export SPARK_WORKER_OPTS=" $LOG4J -Dspark.log.file=/srv/spark/logs/worker.log "
export SPARK_EXECUTOR_OPTS=" $LOG4J -Djava.io.tmpdir=/srv/spark/tmp/executor "
export SPARK_REPL_OPTS=" -Djava.io.tmpdir=/srv/spark/tmp/repl/\$USER "
export SPARK_APP_OPTS=" -Djava.io.tmpdir=/srv/spark/tmp/app/\$USER "
export PYSPARK_PYTHON="/bin/python2"
EOF
```

spark-submit and other tools may use spark-defaults.conf to find the master and other configuration items.

```
cat > spark-defaults.conf <<EOF
spark.master            spark://node0.pc.datastax.com:7077
spark.executor.memory   512m
spark.eventLog.enabled  true
spark.serializer        org.apache.spark.serializer.KryoSerializer
EOF
```

The systemd units are a little less complex than Cassandra's. The spark-master.service unit
should only exist on node0, while every other node runs spark-worker. Spark workers are given
a weight of 100 compared to Cassandra's weight of 1000 so that Cassandra is given priority over
Spark without starving it entirely.

```
cat > spark-worker.service <<EOF
[Unit]
Description=Spark Worker
After=network.target

[Service]
Type=forking
User=spark
Group=spark
ExecStart=/opt/spark/sbin/start-slave.sh 1 spark://node0.pc.datastax.com:7077
StandardOutput=journal
StandardError=journal
LimitNOFILE=infinity
LimitMEMLOCK=infinity
LimitNPROC=infinity
LimitAS=infinity
CPUAccounting=true
CPUShares=100

[Install]
WantedBy=multi-user.target
EOF
```

The master unit is similar and only gets installed on node0. Since it is not competing
for resources, there's no need to turn on cgroups for now.

```
cat > spark-master.service <<EOF
[Unit]
Description=Spark Master
After=network.target

[Service]
Type=forking
User=spark
Group=spark
ExecStart=/opt/spark/sbin/start-master.sh 1
StandardOutput=journal
StandardError=journal
LimitNOFILE=infinity
LimitMEMLOCK=infinity
LimitNPROC=infinity
LimitAS=infinity

[Install]
WantedBy=multi-user.target
EOF
```

Now deploy all of these configs. Relocate the spark config into /etc/spark and copy
a couple templates, then write all the files there. spark-env.sh goes on all nodes.
The unit files are described above. Finally,
a command is run to instruct systemd to read the new unit files.

```
cl-run.pl --list all -c "sudo cp /opt/spark/conf/log4j.properties.template /opt/spark/conf/log4j.properties"
cl-run.pl --list all -c "sudo cp /opt/spark/conf/fairscheduler.xml.template /opt/spark/conf/fairscheduler.xml"
cl-sendfile.pl --list all -x -l spark-env.sh -r /etc/spark/spark-env.sh
cl-sendfile.pl --list all -x -l spark-defaults.conf -r /etc/spark/spark-defaults.conf
cl-sendfile.pl --list workers -x -l spark-worker.service -r /etc/systemd/system/multi-user.target.wants/spark-worker.service
cl-sendfile.pl --list all --incl node0 -x -l spark-master.service -r /etc/systemd/system/multi-user.target.wants/spark-master.service
cl-run.pl --list all -c "sudo systemctl daemon-reload"
```

With all of that done, it's time to turn on Spark to see if it works.

```
cl-run.pl --list all --incl node0 -c "sudo systemctl start spark-master.service"
cl-run.pl --list workers -c "sudo systemctl start spark-worker.service"
```

Now I can browse to the Spark master webui.

![screenshot](/images/spark-master-screenshot-2014-07-15.jpg)

## Installing spark-cassandra-connector

The connector is now published in Maven and can be installed easiest using ivy on the
command line. Ivy can pull all dependencies as well as the connector jar, saving a lot of
fiddling around. In addition, while ivy can download the connector directly, it will
end up pulling down all of Cassandra and Spark. The script fragment below pulls down only what
is necessary to run the connector against a pre-built Spark.

This is only really needed for the spark-shell so it can access Cassandra. Most projects
should include the necessary jars in a fat jar rather than pushing these packages
to every node.

I run these commands on node0 since that's where I usually work with spark-shell. To run it on
another machine, Spark will have to be present and match the version of the cluster, then this
same process will get everything needed to use the connector.

```
cat > download-connector.sh <<EOF
mkdir /opt/connector
cd /opt/connector

rm *.jar

curl -o ivy-2.3.0.jar \
  'http://search.maven.org/remotecontent?filepath=org/apache/ivy/ivy/2.3.0/ivy-2.3.0.jar'
curl -o spark-cassandra-connector_2.10-1.0.0-beta1.jar \
  'http://search.maven.org/remotecontent?filepath=com/datastax/spark/spark-cassandra-connector_2.10/1.0.0-beta1/spark-cassandra-connector_2.10-1.0.0-beta1.jar'

ivy () { java -jar ivy-2.3.0.jar -dependency \$* -retrieve "[artifact]-[revision](-[classifier]).[ext]"; }

ivy org.apache.cassandra cassandra-thrift 2.0.9
ivy com.datastax.cassandra cassandra-driver-core 2.0.3
ivy joda-time joda-time 2.3
ivy org.joda joda-convert 1.6

rm -f *-{sources,javadoc}.jar

EOF

sudo bash download-connector.sh
```

## Using spark-cassandra-connector With spark-shell

All that's left to get started with the connector now is to get spark-shell to pick it up. The easiest
way I've found is to set the classpath with --driver-class-path then restart the context in the REPL
with the necessary classes imported to make sc.cassandraTable() visible.

The newly loaded methods will not show up in tab completion. I don't know why.

```
/opt/spark/bin/spark-shell --driver-class-path $(echo /opt/connector/*.jar |sed 's/ /:/g')
```

It will print a bunch of log information then present scala&gt; prompt.

```
scala> sc.stop
```

Now that the context is stopped, it's time to import the connector.

```
scala> import com.datastax.spark.connector._
scala> val conf = new SparkConf()
scala> conf.set("cassandra.connection.host", "node1.pc.datastax.com")
scala> val sc = new SparkContext("local[2]", "Cassandra Connector Test", conf)
scala> val table = sc.cassandraTable("keyspace", "table")
scala> table.count
```

To make sure everything is working, I ran some code I'm working on for my 2048 game analytics
project. Each context gets an application webui that displays job status.

![screenshot](/images/spark-stages-screenshot-2014-07-15.jpg)

## Conclusion

It was a lot of work getting here, but what we have at the end is a Spark shell that can
access tables in Cassandra as RDDs with types pre-mapped and ready to go.

There are some things that can be improved upon. I will likely package all of this into
a Docker image at some point. For now, I need it up and running for some demos that will
be running on portacluster at [OSCON 2014](http://www.oscon.com/oscon2014).
