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

- So at this stage; you have a (spark based) program which can create tables and run SQL/DML on delta lake tables which are using an s3 compliant object storage (minio) for data storage. Now it's time to look at what tools can be used to query out the data and connect it with Business Intelligence tools like superset / Tableau / PowerBI etc.

#### Trino

##### Why Trino?

- Honestly I jumped to setup Apache superset, thinking Apache SQL would integrate fine with superset and wouldn't need anyother query enginge layer to keep things simpler, however I found myself out of support as Superset requires metadata about the underlying dataset in terms of catalog / schemas etc, which was not available in a simple way using spark as the STS or the spark's thriftserver can query the delta-lake tables, but couldn't provide the metadata and it also required me copy the delta-lake and aws-s3 jars into spark's jars dir which I didn't like much..

- With that backgaround, I started searching for a query engine which can sit on top of delta lake tables which are stored on minio's s3 fs. 
- I soon found that facebook/meta's Presto was a first class citizen as it had declared integration with Delta Lake using one of Delta's native connectors. 
- With Presto I was able to query the tables using the s3 paths as table names, which seemed a bit too odd for me to use. Soon I found out that Presto too requires metadata in terms of Hive Metastore - to know about the underlying schema. 
- With that I went on to setup a hive standalone metastore service, which will store the metatdata about the delta-lake tables. How it will store that was not known yet. 
- Once the HMS was up, and I was able to query delta-lake tables but still using path construct as table names, instead of simple table names. Also Presto's latest release 0.280 was spewing out of lot of errors on the console, but was still able to query the delta-tables. Due to the exceptions it encountered a simple query with three rows was also taking lot of time to complete.
```
select * from delta."$path$."s3a://delta-lake/demo_table";
```
- So with above experience, I went back to drawing board thinking Presto is not the right choice and then I looked at Trino. Trino is fork of Presto but with much better documentation on setup and configuration. It was in Trino's documentation where it was clearly mentioned that for using Trino's query engine a HMS is a must.
- Trino also provides a way to register the metadata into hive using an out of box stored procedure/ function as shown below. This made things crystal clear and everything else worked out of box.
```
CALL example.system.register_table(schema_name => 'testdb', table_name => 'customer_orders', table_location => 's3a://my-bucket/a/path')
```

##### Trino Setup

- For Trino, I choose to use the Docker Image as I had already got a flavour of manual installation because of an attempt to use Presto.
- To start Trino - below docker is the command to be used, however before you jump and fire up the container - a couple of configuration files are to be created.
```
#!/bin/bash
docker rm trino
docker run --name trino -d -p 8084:8080 -v $PWD/trino/etc:/etc/trino -v /data/trino:/var/trino trinodb/trino
```
- Create below configuration files under directory 'etc', such that it can mounted as shown above. (These details are avialable in Trino's documentation as well, however few settings you have to figure out as you go along - so specifying them here)
- `node.properties`
```
node.environment=docker
node.id=ffffffff-ffff-ffff-ffff-ffffffffffff
node.data-dir=/var/trino/data
```
- `jvm.config`
```
-server
-Xmx5G
-XX:InitialRAMPercentage=80
-XX:MaxRAMPercentage=80
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-XX:ReservedCodeCacheSize=512M
-XX:PerMethodRecompilationCutoff=10000
-XX:PerBytecodeRecompilationCutoff=10000
-Djdk.attach.allowAttachSelf=true
-Djdk.nio.maxCachedBufferSize=2000000
-XX:+UnlockDiagnosticVMOptions
-XX:+UseAESCTRIntrinsics
# Disable Preventive GC for performance reasons (JDK-8293861)
-XX:-G1UsePreventiveGC
```
- `config.properties`
```
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=8080
discovery.uri=http://localhost:8080
```
- 'catalog/delta.properties` - notice the path it's 'etc/catalog/delta.properites'. This is the properties file, which is used by Trino to interact with Delta Tables. The properties provides the HMS uri, the s3 end-point and creds.
```
connector.name=delta_lake
hive.metastore.uri=thrift://<host>:9083
delta.hive-catalog-name=delta
delta.register-table-procedure.enabled=true
hive.s3.aws-access-key=<access_key>
hive.s3.aws-secret-key=<secret_key>
hive.s3.endpoint=http://<host>:9000
hive.s3.ssl.enabled=false
```

