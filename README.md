# dw-stack

Various Data Warehouse Logical Architectures using open source projects like Spark, Delta Lake, Superset, MinIO, Iceberg and more...

## Delta Lake / Apache Spark / Trino / Minio / Superset

![minio-spark-delta-superset](https://user-images.githubusercontent.com/53151312/230783811-46a66ebb-bebd-4d9d-9140-62f0f955ce98.jpg)

### Setup

Delta Lake uses parquet [https://parquet.apache.org/] file format under the hoods and uses Apache Spark's unified engine (batchs & structured streams) for big data analytics. I visualize Apache spark playing a big role in data ingestion and streaming back out to downstreams from delta-lake.

In this particular setup, I started with Apache Spark as that's what you need to get started reading and writing delta lake tables.

#### Apache Spark - Version: 3.3.2 (spark-3.3.2-bin-hadoop3)

- Download the jar from maven repo and extract it to a location, let's say spark install directory.
- Spark can be setup on K8s, Docker or stand-alone mode. I chose stand-alone mode to know a bit more about the setup, as if you use a docker image/container it really extracts few of the nitty-gritties which one should if you are playing with the product first time around.
-   Spark can be run as an interactive shell or you could run a cluster. For any cluster based platform there is a control plane and nodes. Similarly in spark you have a master and a worker. So from <SPARK_INSTALL_DIR>/sbin - start a master using `start-master.sh` script.
- This will start a master on your local box and will provide a url (on console logs) which needs to be supplied to the worker. It would be in following format `spark://<host>:7077`
- Start a worker by providing url and invoking the `start-worker.sh` script under <SPARK_INSTALL_DIR>/sbin. The worker will join the cluster using the master url. The command to start the worker would look something like this:

```
<SPARK_INSTALL_DIR>/sbin/start-worker.sh --master spark://<host>:7077 
```

- Now that your spark cluster is up, you can submit your workloads like jobs or streams using Spark API in python/java/scala. I have used python to keep things simple.
- This is where it get's bit tricky, as the cluster is agnostic and doesn't know what spark jobs are going to do, so when you submit your workload via a python application, you need provide all the metadata and runtime dependencies too with the application submitted to spark cluster for execution.
- The sample application I ran was from the delta-lake's 'geting started' documentation and was submitted to the spark cluster using the below script. This is where you start seeing the dependencies of delta-lake.
```
#!/bin/bash
export PYSPARK_DRIVER_PYTHON=python # Do not set in cluster modes.
export PYSPARK_PYTHON=./environment/bin/python
export PYSPARK_MASTER=spark://localhost:7077
/data/delta_workspace/spark/bin/spark-submit \
--conf "spark.driver.extraJavaOptions=-Dlog4j.configuration=log4j-spark.properties" \
--conf "spark.executor.extraJavaOptions=-Dlog4j.configuration=log4j-spark.properties" \
--conf "spark.hadoop.fs.s3a.access.key=<access_key>" \
--conf "spark.hadoop.fs.s3a.secret.key=<secret_key>" \
--conf "spark.hadoop.fs.s3a.endpoint=http://<host-name>:9000" \
--conf "spark.databricks.delta.retentionDurationCheck.enabled=false" \
--packages io.delta:delta-core_2.12:2.2.0,org.apache.hadoop:hadoop-aws:3.3.2,com.amazonaws:aws-java-sdk-bundle:1.11.1026 \
--master ${PYSPARK_MASTER} \
--archives pyspark_venv.tar.gz#environment main.py
```
- You will notice the s3a conf properties in the above settings, ignore them for now. If you run the above script without the `*.s3a.*` properties; the delta lake's tables will be created on local file system under spark's default data dir.
- the `main.py` is the python script which houses the **ETL** code per say and API calls to delta lake layer to store the data.

- Notice the packages mentioned for delta, aws-s3 that's how you provide the runtime dependencies of delta lake and s3 compliant storage implementation.

#### MinIO

- Now it's time to decide whether you want the delta lake tables on 'hdfs' or 'object storage' like s3. I choose to use the s3 compliant fs for storage,as setting up hadoop and it's cluster take's quite a time. I did setup, but still eventually decided to use s3 compliant fs, as it was more straightforward and contemporary. 
- For s3 compliant fs - I choose to use minio implementation, whose installation is actually very straight forward. Download the latest minio binary and create a start up script as shown below and that's it! you have an s3 compliant object storage up and running - ready to be used.

```
#!/bin/bash

export MINIO_ROOT_USER=<access_key>
export MINIO_ROOT_PASSWORD=<secret_key>
export MINIO_DATA_PATH=/data/minio
./minio server $MINIO_DATA_PATH --console-address ":9001"

```

- So at this stage; you have a program which can create tables and run SQL/DML on delta lake tables which are using an s3 compliant object storage (minio) for data storage.

