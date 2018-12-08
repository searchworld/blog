# Hadoop Pseudo-Distributed Operation
## Setup passphraseless ssh
1. System Preference -> Sharing -> Remote Login
2. keygen
    ```
    $ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
    $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    $ chmod 0600 ~/.ssh/authorized_keys
    $ ssh localhost
    ```

## Download hadoop
```
/etc/profile:
export HADOOP_HOME=/Users/ben/Downloads/hadoop/hadoop-2.8.5
export HADOOP_CONF_DIR=/Users/ben/Downloads/hadoop/hadoop-2.8.5/etc/hadoop
```

## Configuration HDFS
in `HADOOP_HOME`
```
etc/hadoop/hadoop-env.sh(this is important, otherwise fink run will fail because yarn default java to /bin/java):
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home


etc/hadoop/core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>

etc/hadoop/hdfs-site.xml:
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

### Execution
```
$ bin/hdfs namenode -format
$ sbin/start-dfs.sh
$ bin/hdfs dfs -mkdir /user
$ bin/hdfs dfs -mkdir /user/ben
```
find HDFS info in http://localhost:50070/

## YARN on a Single Node
in `HADOOP_HOME`
```
etc/hadoop/mapred-site.xml:
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>

etc/hadoop/yarn-site.xml:
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>

$ sbin/start-yarn.sh
```
find yarn info in http://localhost:8088/

## Download flink with hadoop dist

## Run a Flink job on YARN
in the fink dir:
```
./bin/flink run -m yarn-cluster -yn 2 -yjm 1024m -ytm 1024m ./examples/batch/WordCount.jar
```