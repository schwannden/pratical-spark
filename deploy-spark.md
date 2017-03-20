# Deploy Spark

Spark cluster can be managed in 3 ways:

1. [Mesos](http://spark.apache.org/docs/latest/running-on-mesos.html)
2. [Yarn](http://spark.apache.org/docs/latest/running-on-yarn.html)
3. [Spark Standalone](http://spark.apache.org/docs/latest/spark-standalone.html)

And since Spark Standalone comes with Spark directly, I will use this as an example to avoid further installation details.

In my deployment I use a cluster of two computers:

1. Machine A - intel-i7 - 32G RAM - IP 10.0.0.2 - \(master node\) \(slave node\)
2. Machine B - intel-i7 - 32G RAM - IP 10.0.0.1 - \(slave node\)

I will use machine A as master. The master is responsible for taking your submitted jobs and ask the slaves to run it. So as you can imagine, one of the machine in the cluster can be both master and slave because we should be able to also run computation on the computer we submitted jobs. But a cluster should have only one of its node as master.

### Configuring Master \(Machine A, IP 10.0.0.2\)

Copy `/usr/lib/spark/conf/spark-env.sh.template` to ` /usr/lib/spark/conf/spark-env.sh ` , then find these lines and uncomment.

```bash
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
SPARK_EXECUTOR_MEMORY=16g
SPARK_DRIVER_MEMORY=4g
SPARK_MASTER_HOST=10.0.0.2
SPARK_MASTER_PORT=7077
SPARK_WORKER_CORES=8
SPARK_WORKER_MEMORY=32g
```

To understand each parameters, not this is how a spark job is started

1. job is submitted, and spark driver declares the transformations and actions on RDDs of data and submits such requests to the master.
2. master partitions the job based on slaves connected to it, and submit jobs to slave
3. Each slave will have worker processes to receive command from master
4. Each workers spins of executors for the job received from master

For simplicity, lets assume each slave has only one worker, so the worker uses all the cores available on the machine and as much memory possible.

* SPARK\_EXECUTOR\_MEMORY: use half the amount of SPARK\_WORKER\_MEMORY, so that a worker can comfortably run two executors.
* SPARK\_DRIVER\_MEMORY: The memory limit for spark driver, 4~8g suffice.
* SPARK\_MASTER\_HOST: IP of the master machine.
* SPARK\_MASTER\_PORT: port of the master machine, slaves will connect to this port.
* SPARK\_WORKER\_CORES: The cores each worker is allowed to use, use all the cores for now.
* SPARK\_WORKER\_MEMORY: The memory limit for spark worker, use all the memory for now.

### Start Master \(Machine A, IP 10.0.0.2\)

```bash
start-master.sh
```

Yup, that's it. Default web UI for master will start at port 8080, so go to 10.0.0.2:8080 in your browser, you will see current status. Now there are no workers, no applications running.

![](/assets/1.png)

### Start Slave \(Machine A, IP 10.0.0.2, Machine B, IP 10.0.0.1\)

On each of your machine, start slave as follows

```bash
start-slave.sh spark://10.0.0.2:7077
```

Yup, that's it! How simple!

Check the web UI again, you should see two workers alive

![](/assets/2.png)

Congratulations, now you have a spark cluster!



