# Spark dynamic allocation on Standalone cluster in Docker

Apache Spark is an open-source, distributed processing system used for big data workloads.

In this demo, a Spark container uses a Spark Standalone cluster as a resource management and job scheduling technology to perform distributed data processing.

This Docker image contains Spark binaries prebuilt and uploaded in Docker Hub.

## Start Swarm cluster

1. start swarm mode in node1
```shell
$ docker swarm init --advertise-addr <IP node1>
$ docker swarm join-token worker  # issue a token to add a node as worker to swarm
```

2. add 3 more workers in swarm cluster (node2, node3, node4)
```shell
$ docker swarm join --token <token> <IP nodeN>:2377
```

3. label each node to anchor each container in swarm cluster
```shell
docker node update --label-add hostlabel=hdpmst node1
docker node update --label-add hostlabel=hdp1 node2
docker node update --label-add hostlabel=hdp2 node3
docker node update --label-add hostlabel=hdp3 node4
```

4. create an external "overlay" network in swarm to link the 2 stacks (hdp and spk)
```shell
docker network create --driver overlay mynet
```

5. start the Spark cluster and Hadoop standalone
```shell
$ docker stack deploy -c docker-compose.yml spk
$ docker stack ps spk
nx0huvb6ent1   spk_hdpmst.1    mkenjis/ubhdp_img:latest          node1     Running         Running 36 minutes ago             
7sgbf3tgwcug   spk_spk1.1      mkenjis/ubspkcluster_img:latest   node2     Running         Running 36 minutes ago             
jgr9as5irt6r   spk_spk2.1      mkenjis/ubspkcluster_img:latest   node3     Running         Running 36 minutes ago             
rdgxc68jrdub   spk_spk3.1      mkenjis/ubspkcluster_img:latest   node4     Running         Running 36 minutes ago             
ux3l1ywtvjf7   spk_spk_mst.1   mkenjis/ubspkcluster_img:latest   node1     Running         Running 36 minutes ago
```

## Set up Spark External Shuffle Service

1. access spark master node
```shell
$ docker container ls   # run it in each node and check which <container ID> is running the Spark client constainer
CONTAINER ID   IMAGE                                 COMMAND                  CREATED         STATUS         PORTS                                          NAMES
ca25353fc988   mkenjis/ubspkcluster_img:latest   "/usr/bin/supervisord"   42 minutes ago   Up 42 minutes   4040/tcp, 7077/tcp, 8080-8082/tcp, 10000/tcp   spk_spk_mst.1.ux3l1ywtvjf7n54kwixdzf2dw
9c59a0b6e029   mkenjis/ubhdp_img:latest          "/usr/bin/supervisord"   42 minutes ago   Up 42 minutes   9000/tcp                                       spk_hdpmst.1.nx0huvb6ent1on9ziprti9ldh

$ docker container exec -it <spk_mst ID> bash
```

2. stop worker nodes on standalone cluster and edit spark-defaults.conf
```shell
$ stop-slaves.sh
$ cd $SPARK_HOME
$ vi spark-defaults.conf

spark.shuffle.service.enabled true
spark.dynamicAllocation.enabled true
spark.dynamicAllocation.initialExecutors 1
spark.dynamicAllocation.minExecutors 1
spark.dynamicAllocation.maxExecutors 20
spark.dynamicAllocation.schedulerBacklogTimeout 1m
spark.dynamicAllocation.executorIdleTimeout 2m
```

3. copy spark-defaults.conf to all spark slaves
```shell
$ scp spark-defaults.conf root@spk1:/usr/local/spark-2.3.2-bin-hadoop2.7/conf
spark-defaults.conf                                                               100%  285   216.7KB/s   00:00    
$ scp spark-defaults.conf root@spk2:/usr/local/spark-2.3.2-bin-hadoop2.7/conf
spark-defaults.conf                                                               100%  285   239.7KB/s   00:00    
$ scp spark-defaults.conf root@spk3:/usr/local/spark-2.3.2-bin-hadoop2.7/conf
spark-defaults.conf                                                               100%  285    59.8KB/s   00:00
```

4. start external shuffle service in each spark slave

In spk1, spk2 and spk3, run :
```shell
$ start-shuffle-service.sh
```

In spkmst, run :
```shell
$ start-slaves.sh
```

5. start spark-shell
```shell
$ spark-shell --master spark://<spk_mst>:7077
2023-09-06 14:46:07 WARN  NativeCodeLoader:62 - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
2023-09-06 14:46:18 WARN  Utils:66 - spark.executor.instances less than spark.dynamicAllocation.minExecutors is invalid, ignoring its setting, please update your configs.
Spark context Web UI available at http://ca25353fc988:4040
Spark context available as 'sc' (master = spark://ca25353fc988:7077, app id = app-20230906144617-0000).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.2
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```



