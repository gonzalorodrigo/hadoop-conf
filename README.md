# hadoop-conf
## Introduction
This repo contains the configuration files for Yarn (hadoop 2) to test dynamic allocation of resources in Yarn+Spark:
- Jobs are given resources.
- Jobs resources are stolen.

Versions used:
- Hadoop: 2.7.1 
- Spark: 1.5.2 binary with Yarn support.
- Java: 1.7.0_45
 
Yarn is configueed
- Using the SPARK external suffler.
- Using Fair scheduler.
- Two queues, 50% of the share each.
- Preemption of containers enabled.

## Experiment description
Results can be observed in 2jobs_comments. The experiment was the following:
- Yarn cluster with FAIR scheduler. 1 management node, 4 compute nodes, each one with 10GB RAM, 8 vCores available for work.
- Scheduling policies:
    - two queues (qf1, qf2), both configured with a fair-share of 0.5.
    - If any of the queues has much less than its fair share for more than 20s -> Steal containers from jobs in other queues.
    - A queue cannot use more than 35GB of RAM and 25vcores.
- Jobs: two jobs, one runs in qf1, qf2. They are spark jobs running map/reduce. They are Identical: one reduce stage for 1M tasks. Each tasks consists in the calculation of a few PI decimals, i.e., embarrassingly parallel. 
- Timeline:
    - 15:03 Job2 starts (right): It deploys and takes all the resources of the system (35 GB RAM)
    - 15:08 Job1 starts (left), it can only create one executor with the remaining resources.
    - 15:07, 15:08, 15:09: Yarn takes a few containers from Job2.
    - 15:09: Job1 receives containers and creates three executors on then.

Jobs are called in the context of Spark
```
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-cluster --driver-memory 2g  \
   --executor-memory 1g --executor-cores 1  --conf spark.shuffle.service.enabled=true \ 
   --conf spark.dynamicAllocation.enabled=true --conf spark.dynamicAllocation.minExecutors=1 \ 
   --conf spark.dynamicAllocation.maxExecutors=40 --queue qf1 lib/spark-examples*.jar \
   1000000

./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-cluster --driver-memory 2g  \
   --executor-memory 1g     --executor-cores 1  --conf spark.shuffle.service.enabled=true \
   --conf spark.dynamicAllocation.enabled=true --conf spark.dynamicAllocation.minExecutors=1 \
   --conf spark.dynamicAllocation.maxExecutors=40 --queue qf2   lib/spark-examples*.jar \
   1000000
```
