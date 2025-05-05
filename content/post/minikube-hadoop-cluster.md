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

# Reference

- [Accessing services in minikube via DNS](https://www.andreasgerstmayr.at/2022/11/23/accessing-services-in-minikube-via-dns.html)

