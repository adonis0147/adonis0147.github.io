---
title: "Deploy a Hadoop Cluster by Minikube"
date: 2025-05-05T17:48:50+08:00
draft: false
summary: A guide to deploy a Hadoop cluster by minikube.
tags: ["Hadoop", "Kubernetes", "Minikube", "Helm"]
categories: ["English"]
---

This post is a guide to deploy a Hadoop cluster by minikube.


# macOS

## Prerequisite

- [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/)

```shell
$ brew install qemu socket_vmnet helm kubectl
```

## Start Minikube

```shell
$ minikube start --driver qemu --network socket_vmnet \
    --cpus "$(($(nproc) / 2))" --memory "$(nproc)g"
```

# Ubuntu 24.04

## Prerequisite

- [Helm](https://github.com/helm/helm/releases/latest)
- [kubectl](https://kubernetes.io/releases/download/)
- [Docker](https://docs.docker.com/engine/install/ubuntu/)

```shell
# https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user
$ sudo groupadd docker

$ sudo usermod -aG docker $USER

$ newgrp docker
```

## Start Minikube

```shell
$ minikube start --driver docker \
    --cpus "$(($(nproc) / 2))" --memory "$(nproc)g"
```

# Install Hadoop Cluster by Helm

```shell
$ git clone https://github.com/adonis0147/helm-hadoop

$ cd helm-hadoop

$ bash docker/build_image.sh

$ helm install --name-template hadoop .
```

# Check the Status of Hadoop Cluster

```shell
$ kubectl get pods

NAME   READY   STATUS    RESTARTS        AGE     IP            NODE       NOMINATED NODE   READINESS GATES
dn-0   1/1     Running   0               5m14s   10.244.0.21   minikube   <none>           <none>
dn-1   1/1     Running   0               5m9s    10.244.0.27   minikube   <none>           <none>
dn-2   1/1     Running   0               5m5s    10.244.0.30   minikube   <none>           <none>
hs-0   1/1     Running   0               5m13s   10.244.0.22   minikube   <none>           <none>
jn-0   1/1     Running   0               5m14s   10.244.0.24   minikube   <none>           <none>
jn-1   1/1     Running   0               5m7s    10.244.0.29   minikube   <none>           <none>
jn-2   1/1     Running   0               5m1s    10.244.0.31   minikube   <none>           <none>
nm-0   1/1     Running   0               5m14s   10.244.0.20   minikube   <none>           <none>
nm-1   1/1     Running   0               5m9s    10.244.0.25   minikube   <none>           <none>
nm-2   1/1     Running   0               5m6s    10.244.0.28   minikube   <none>           <none>
nn-0   1/1     Running   0               5m14s   10.244.0.19   minikube   <none>           <none>
nn-1   1/1     Running   1 (4m38s ago)   5m9s    10.244.0.26   minikube   <none>           <none>
rm-0   1/1     Running   0               5m14s   10.244.0.23   minikube   <none>           <none>
```

**Hadoop Cluster**

- `Namenode`: 2
- `Journalnode`: 3
- `Datanode`: 3
- `Resourcemanager`: 1
- `Nodemanager`: 3
- `Historyserver`: 1

# Access the services

## macOS

```shell
# Set route up
$ sudo route -n delete 10.244.0.0/16
$ sudo route -n add 10.244.0.0/16 "$(minikube ip)"

# Don't kill this process
$ minikube tunnel
```

## Ubuntu 24.04

```shell
# Set DNS up
$ interface="$(netstat -nr | grep "$(minikube ip | sed -n 's/\(.*\)\..*/\1.0/p')" |
    awk '{print $NF}')"
$ sudo resolvectl dns "${interface}" \
    "$(kubectl get -n kube-system service --no-headers | awk '{print $3}')"
$ sudo resolvectl domain "${interface}" cluster.local

# Set route up
$ sudo route del -net 10.244.0.0 netmask 255.255.0.0
$ sudo route add -net 10.244.0.0 netmask 255.255.0.0 gw "$(minikube ip)"

# Don't kill this process
$ minikube tunnel
```

# Test

## Ping

```shell
$ ping nn-0.namenode.default.svc.cluster.local

PING nn-0.namenode.default.svc.cluster.local (10.244.0.19) 56(84) bytes of data.
64 bytes from nn-0.namenode.default.svc.cluster.local (10.244.0.19): icmp_seq=1 ttl=63 time=0.069 ms
64 bytes from nn-0.namenode.default.svc.cluster.local (10.244.0.19): icmp_seq=2 ttl=63 time=0.079 ms
```

## Access HDFS

```shell
$ kubectl exec -it nn-0 -- hadoop fs -ls /

Found 1 items
drwxrwx---   - root supergroup          0 2025-05-05 11:10 /tmp
```

## MapReduce Wordcount

```shell
$ kubectl exec -it rm-0 -- bash -c 'for i in {0..999}; do echo ${i} >>numbers; done'

$ kubectl exec -it rm-0 -- hadoop fs -put numbers /numbers

$ kubectl exec -it rm-0 -- hadoop jar \
    hadoop-current/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.0.jar wordcount \
    /numbers /output

2025-05-05 11:29:30,858 INFO client.DefaultNoHARMFailoverProxyProvider: Connecting to ResourceManager at rm-0.resourcemanager.default.svc.cluster.local/10.244.0.23:8032
2025-05-05 11:29:31,682 INFO mapreduce.JobResourceUploader: Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/root/.staging/job_1746443418771_0001
2025-05-05 11:29:32,242 INFO input.FileInputFormat: Total input files to process : 1
2025-05-05 11:29:32,492 INFO mapreduce.JobSubmitter: number of splits:1
2025-05-05 11:29:32,711 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1746443418771_0001
2025-05-05 11:29:32,712 INFO mapreduce.JobSubmitter: Executing with tokens: []
2025-05-05 11:29:33,054 INFO conf.Configuration: resource-types.xml not found
2025-05-05 11:29:33,055 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
2025-05-05 11:29:33,723 INFO impl.YarnClientImpl: Submitted application application_1746443418771_0001
2025-05-05 11:29:33,854 INFO mapreduce.Job: The url to track the job: http://rm-0.resourcemanager.default.svc.cluster.local:8088/proxy/application_1746443418771_0001/
2025-05-05 11:29:33,856 INFO mapreduce.Job: Running job: job_1746443418771_0001
2025-05-05 11:29:45,261 INFO mapreduce.Job: Job job_1746443418771_0001 running in uber mode : false
2025-05-05 11:29:45,263 INFO mapreduce.Job:  map 0% reduce 0%
2025-05-05 11:29:51,384 INFO mapreduce.Job:  map 100% reduce 0%
2025-05-05 11:30:00,470 INFO mapreduce.Job:  map 100% reduce 100%
2025-05-05 11:30:00,494 INFO mapreduce.Job: Job job_1746443418771_0001 completed successfully
2025-05-05 11:30:00,638 INFO mapreduce.Job: Counters: 54
...

$ kubectl exec -it rm-0 -- hadoop fs -ls /output

Found 2 items
-rw-r--r--   3 root supergroup          0 2025-05-05 11:29 /output/_SUCCESS
-rw-r--r--   3 root supergroup       5890 2025-05-05 11:29 /output/part-r-00000
```

## Spark SparkPi

```shell
# Download Apache Spark
$ curl -LO 'https://archive.apache.org/dist/spark/spark-3.5.5/spark-3.5.5-bin-hadoop3.tgz'
$ tar -zxvf spark-3.5.5-bin-hadoop3.tgz

# Set Hadoop client up
$ cd spark-3.5.5-bin-hadoop3
$ SPARK_HOME="$(pwd)"

$ mkdir -p conf/hadoop
$ kubectl get configmaps hdfs-conf -o jsonpath="{.data.core-site\.xml}" \
    >conf/hadoop/core-site.xml
$ kubectl get configmaps hdfs-conf -o jsonpath="{.data.hdfs-site\.xml}" \
    >conf/hadoop/hdfs-site.xml
$ kubectl get configmaps yarn-conf -o jsonpath="{.data.yarn-site\.xml}" \
    >conf/hadoop/yarn-site.xml

# Set Spark up
$ cat >conf/spark-defaults.conf <<EOF
spark.master                  yarn
spark.submit.deployMode       cluster

spark.eventLog.enabled        true
spark.eventLog.dir            hdfs:///tmp/spark-events
spark.history.fs.logDirectory hdfs:///tmp/spark-events
EOF

$ cat >conf/spark-env.sh <<EOF
HADOOP_CONF_DIR="${SPARK_HOME}/conf/hadoop"
YARN_CONF_DIR="${SPARK_HOME}/conf/hadoop"
HADOOP_USER_NAME='root'
EOF

# Run SparkPi
$ kubectl exec -it nn-0 -- hadoop fs -mkdir -p /tmp/spark-events

$ ./bin/run-example org.apache.spark.examples.SparkPi 10000

25/05/16 16:34:09 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
25/05/16 16:34:09 INFO DefaultNoHARMFailoverProxyProvider: Connecting to ResourceManager at rm-0.resourcemanager.default.svc.cluster.local/10.244.0.4:8032
25/05/16 16:34:10 INFO Configuration: resource-types.xml not found
25/05/16 16:34:10 INFO ResourceUtils: Unable to find 'resource-types.xml'.
25/05/16 16:34:10 INFO Client: Verifying our application has not requested more than the maximum memory capability of the cluster (8192 MB per container)
25/05/16 16:34:10 INFO Client: Will allocate AM container, with 1408 MB memory including 384 MB overhead
25/05/16 16:34:10 INFO Client: Setting up container launch context for our AM
25/05/16 16:34:10 INFO Client: Setting up the launch environment for our AM container
25/05/16 16:34:10 INFO Client: Preparing resources for our AM container
25/05/16 16:34:10 WARN Client: Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME.
25/05/16 16:34:13 INFO Client: Uploading resource file:/private/var/folders/pq/ffp2x74d7gs2_y_z5v7q4dn00000gp/T/spark-5e05a1af-feba-4e50-a0ec-e05c0be847a7/__spark_libs__18048069634091976738.zip -> hdfs://hadoop-cluster/user/root/.sparkStaging/application_1747383152432_0004/__spark_libs__18048069634091976738.zip
25/05/16 16:34:14 INFO Client: Uploading resource file:/spark-3.5.5-bin-hadoop3/examples/jars/spark-examples_2.12-3.5.5.jar -> hdfs://hadoop-cluster/user/root/.sparkStaging/application_1747383152432_0004/spark-examples_2.12-3.5.5.jar
25/05/16 16:34:14 INFO Client: Uploading resource file:/spark-3.5.5-bin-hadoop3/examples/jars/scopt_2.12-3.7.1.jar -> hdfs://hadoop-cluster/user/root/.sparkStaging/application_1747383152432_0004/scopt_2.12-3.7.1.jar
25/05/16 16:34:14 WARN Client: Same name resource file:///spark-3.5.5-bin-hadoop3/examples/jars/spark-examples_2.12-3.5.5.jar added multiple times to distributed cache
25/05/16 16:34:14 INFO Client: Uploading resource file:/private/var/folders/pq/ffp2x74d7gs2_y_z5v7q4dn00000gp/T/spark-5e05a1af-feba-4e50-a0ec-e05c0be847a7/__spark_conf__8161596058161130912.zip -> hdfs://hadoop-cluster/user/root/.sparkStaging/application_1747383152432_0004/__spark_conf__.zip
25/05/16 16:34:14 INFO SecurityManager: Changing view acls to: adonis,root
25/05/16 16:34:14 INFO SecurityManager: Changing modify acls to: adonis,root
25/05/16 16:34:14 INFO SecurityManager: Changing view acls groups to:
25/05/16 16:34:14 INFO SecurityManager: Changing modify acls groups to:
25/05/16 16:34:14 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: adonis, root; groups with view permissions: EMPTY; users with modify permissions: adonis, root; groups with modify permissions: EMPTY
25/05/16 16:34:14 INFO Client: Submitting application application_1747383152432_0004 to ResourceManager
25/05/16 16:34:14 INFO YarnClientImpl: Submitted application application_1747383152432_0004
25/05/16 16:34:15 INFO Client: Application report for application_1747383152432_0004 (state: ACCEPTED)
...

# Check logs on historyserver
Log Type: stdout
Log Upload Time: Fri May 16 08:34:39 +0000 2025
Log Length: 33
Pi is roughly 3.1416564911416565
```

# Reference

- [Accessing services in minikube via DNS](https://www.andreasgerstmayr.at/2022/11/23/accessing-services-in-minikube-via-dns.html)

