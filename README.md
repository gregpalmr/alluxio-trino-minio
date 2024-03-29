# alluxio-trino-minio

### Employ the Alluxio Transparent URI capability with Trino, Hive Metastore and Spark and with Minio as the S3 object store

---

## INTRO

This git repo provides a complete environment for demonstrating how to configure Alluxio's Transparent URI capability for use with Trino as a query engine and Hive as the metastore, as well as with Spark and SparkSQL.

This docker compose package was based on the docker compose setup described here:

     https://github.com/bitsondatadev/trino-getting-started/tree/main/hive/trino-minio

This modified compose package deploys Trino, Alluxio, Hive metastore, Spark and Minio. It configures a Trino Hive catalog to use the Alluxio Transparent URI feature. See: 

     https://docs.alluxio.io/ee/user/stable/en/operation/Transparent-Uri.html

## USAGE

### Step 1. Install Docker desktop 

Install Docker desktop on your laptop, including the docker-compose command.

     See: https://www.docker.com/products/docker-desktop/

### Step 2. Clone this repo

Use the git command to clone this repo (or download the zip file from the github.com site).

     git clone https://github.com/gregpalmr/alluxio-trino-minio

     cd alluxio-trino-minio

### Step 3. Specify your Alluxio Enterprise Edition license key

Get a base64 formatted version of your Alluxio Enterprise Edition license key by running the following command:

     export ALLUXIO_LICENSE_BASE64=$(cat ./my-alluxio-license-file.json | base64)

### Step 4. Launch the docker containers.

Launch the containers defined in the docker-compose.yml file using the command:

     docker-compose up -d

The command will create the network object and the docker volumes, then it will take some time to pull the various docker images. When it is complete, you see this output:

     $ docker-compose up -d
     Creating network "trino-minio_trino-network" with driver "bridge"
     Creating volume "trino-minio_minio-data" with local driver
     Creating volume "trino-minio_alluxio-data" with local driver
     Creating trino-coordinator    ... done
     Creating spark-master         ... done
     Creating minio             ... done
     Creating mariadb              ... done
     Creating spark-worker         ... done
     Creating minio-create-buckets ... done
     Creating alluxio-master       ... done
     Creating alluxio-mount-minio-bucket ... done
     Creating alluxio-worker1            ... done
     Creating hive-metastore             ... done

### Step 5. View the buckets in Minio

Use the Minio web console to view the pre-staged "hive" bucket in Minio. Follow these steps:

     - Point your web browser to http://localhost:9001

     - Log in using the user id "minio" and the password "minio123"

     - Click on the "Buckets" link in the upper left margin

     - Click on the buck link named "hive"

     - View the folder named "warehouse" in the hive bucket

### Step 6. View the mounted Minio bucket in Alluxio

Use the Alluxio web console to view the pre-staged mount of the Minio "hive" bucket in Alluxio. Follow these steps:

     - Point your web browser to http://localhost:19999

     - Click on the "MountTable" tab at the top of the page

     - View the "/hive" mount point for the "s3a://hive" Minio bucket

     - Now, click on the "Browse" tab at the top of the page

     - Click on the "hive" directory name

     - View the /hive/warehouse directory in the "hive" bucket

If you want to use the Alluxio command line to view the bucket mount, you can follow these steps:

     docker exec -it alluxio-master bash

     alluxio fs mount 
     s3a://hive/ on /hive (s3, capacity=-1B, used=-1B, not read-only, not shared, properties={alluxio.underfs.s3.inherit.acl=false, alluxio.underfs.s3.disable.dns.buckets=true, alluxio.underfs.s3.endpoint=http://minio:9000, s3a.accessKeyId=******, s3a.secretKey=******})
     /opt/alluxio/underFSStorage on / (local, capacity=195.80GB, used=87.07GB(44%), not read-only, not shared, properties={})

Also, you can view the Alluxio audit log to see when Trino queries access the Alluxio virtual file system. Use this command:

     tail -f /opt/alluxio/logs/master_audit.log

### Step 7. (Optional) Enable debug mode on the Alluxio master

Run the following command if you would like to enable debugging:

     alluxio logLevel --logName=alluxio --target master --level=DEBUG

Later, you can disable debugging mode with the command:

     alluxio logLevel --logName=alluxio --level=INFO

### Step 8. Explore the Trino configuration

Launch a bash session in the Trino coordinator container and explore the setup of the "minio" catalog and the integration with Alluxio. Use the following commands:
     
     docker exec -it trino-coordinator bash

First, see how the Alluxio client jar file was added to the Trino Hive plugin directory (and classpath). This is required for accessing Alluxio using the Transparent URI capability with s3a URIs (s3a://).

     ls -al /usr/lib/trino/plugin/hive/alluxio*client.jar
     -rw-r--r-- 1 trino trino 89061794 Nov 17 22:53 /usr/lib/trino/plugin/hive/alluxio-enterprise-2.9.0-1.0-client.jar

Display the Minio catalog's minio.properties file and notice that the Minio endpoint and credentials are NOT configured here. Instead, Trino will use the Alluxio Transparent URI capability when accessing s3a URIs. Also, see the "hive.config.resources" property pointing to the Alluxio cores-site.xml file:

     cat /etc/trino/catalog/minio/minio.properties

     connector.name=hive
     hive.s3-file-system-type=HADOOP_DEFAULT
     hive.metastore.uri=thrift://hive-metastore:9083
     hive.non-managed-table-writes-enabled=true
     hive.storage-format=ORC
     hive.allow-drop-table=true
     hive.config.resources=/etc/trino/core-site.xml

There is also a catalog for using the Trino Apache Iceberg plugin. Display the Iceberg catalog's iceberg.properties file and notice that connector.name is "iceberg", but it is still referencing the same Hive metastore on port 9083. When Trino queries use this catalog, they will be invoking the Trino Iceberg table format extensions.

     cat /etc/trino/catalog/minio/iceberg.properties

     # File: iceberg.properties
     connector.name=iceberg
     iceberg.catalog.type=hive_metastore
     iceberg.file-format=PARQUET
     hive.metastore.uri=thrift://hive-metastore:9083
     hive.s3-file-system-type=HADOOP_DEFAULT
     hive.config.resources=/etc/trino/core-site.xml

There is also a catalog for using the Trino Delta Lake plugin. Display the Delta Lake catalog's delta-lake.properties file and notice that connector.name is "delta_lake", but it is still referencing the same Hive metastore on port 9083. When Trino queries use this catalog, they will be invoking the Trino Delta Lake table format extensions.

     cat /etc/trino/catalog/minio/deltalake.properties

     # File: deltalake.properties
     connector.name=delta_lake
     delta.hive-catalog-name=deltalake
     delta.enable-non-concurrent-writes=true
     delta.vacuum.min-retention=1h
     delta.register-table-procedure.enabled=true
     delta.extended-statistics.collect-on-write=false
     hive.metastore.uri=thrift://hive-metastore:9083
     hive.s3-file-system-type=HADOOP_DEFAULT
     hive.config.resources=/etc/trino/core-site.xml

Next see how the Alluxio "shim" file system is configured to handle references to s3a URIs. See the properties named "fs.s3a.impl" and "fs.AbstractFileSystem.s3a.impl". Also, the "alluxio.master.hostname" property is defined to point to the Alluxio master node. If you are using Alluxio in high availability (HA) mode, with 3, 5 or 7 master nodes, then you would use the "alluxio.master.rpc.addresses" property instead.

Display the Alluxio core-site.xml file contents:

     cat /etc/trino/core-site.xml

     <?xml version="1.0"?>
     <configuration>

         <!-- Enable the Alluxio Transparent URI feature for s3 and s3a end-points -->
         <property>
             <name>fs.s3a.impl</name>
             <value>alluxio.hadoop.ShimFileSystem</value>
          </property>
          <property>
            <name>fs.AbstractFileSystem.s3a.impl</name>
            <value>alluxio.hadoop.AlluxioShimFileSystem</value>
          </property>

         <!-- Enable the Alluxio Transparent URI feature for HDFS end-points -->
          <property>
            <name>fs.hdfs.impl</name>
            <value>alluxio.hadoop.ShimFileSystem</value>
          </property>
          <property>
            <name>fs.AbstractFileSystem.hdfs.impl</name>
            <value>alluxio.hadoop.AlluxioShimFileSystem</value>
          </property>

          <!-- Don't apply Transparent URI for these files -->
          <property>
            <name>alluxio.user.shimfs.bypass.prefix.list</name>
            <value></value>
          </property>

          <!-- Don't auto mount Alluxio mounts -->
          <property>
            <name>alluxio.master.shimfs.auto.mount.enabled</name>
            <value>false</value>
          </property>

          <!-- Specify the Alluxio master node -->
          <property>
            <name>alluxio.master.hostname</name>
            <value>alluxio-master</value>
          </property>

          <!-- Tell Alluxio to CACHE data when it is read for the first time -->
          <property>
            <name>alluxio.user.file.readtype.default</name>
            <value>CACHE</value>
          </property>

          <!-- Enable client-side (northbound) impersonation in Alluxio -->
          <property>
            <name>alluxio.security.login.impersonation.username</name>
            <value>_HDFS_USER_</value>
          </property>

</configuration>

### Step 9. View the Trino Web console

Point your web browser to the Trino Web console to view query status and history

     http://localhost:8080

The user id is "trino" and there is no password.

### Step 10. Test Trino using the Alluxio Transparent URI feature

The Alluxio Transparent URI feature will redirect references to s3 and s3a URIs to the native Alluxio URI (alluxio://). Therefore Hive table definitions with the "external_location=s3a://<bucket_name>/<directory name>" will be redirected to Alluxio instead of to native Minio. All the Alluxio data orchestration and data caching capabilities will be employed.

Launch a bash session in the Trino coordinator container and run a CREATE TABLE command to create a table using the "minio" Trino cagtalog setup and the "s3a" URI. Then query the data. Use these commands:

     docker exec -it trino-coordinator bash

     trino --catalog minio --debug

     trino>

          USE default;

          CREATE TABLE default.customer_s3a
          WITH (
            format = 'ORC',
            external_location = 's3a://hive/warehouse/customer_s3a/'
          ) 
          AS SELECT * FROM tpch.tiny.customer;
        
          SELECT * FROM default.customer_s3a 
               WHERE acctbal > 3500.00 AND acctbal < 9000.00 
               ORDER BY acctbal LIMIT 25;
 
### Step 11. View the Alluxio cache storage usage

When Trino queries data using Alluxio's Transparent URI feature, it will cache data to Alluxio cache storage, when it is first read from the under file system (in this case, Minio).

To see if the customer_s3a data files are cached, go back to the bash session on the alluxio-master container and list the data files. You should see that some data is cached for the cusotmer_s3a data set by the fact that the 8th column shows a percentage being cached. If it shows 100%, then Alluxio cached all of the data in that file.

     docker exec -it alluxio-master bash

     alluxio fs ls -R /hive/warehouse/customer_s3a
     -rwx------  alluxio  alluxio  78509  PERSISTED 11-17-2022 22:29:47:610  100% /hive/warehouse/customer_s3a/20221117_222944_00003_76isd_03efa9e5-a56a-4d83-8f8a-5bbdbaf9bc1f

You can also view the Alluxio Web console you launched in Step 6 to see if any data is being cached by Alluxio, or run the following Alluxio CLI command:

     alluxio fsadmin report

### Step 12. Explore the Spark configuration

Launch a bash session in the Spark master container and explore how the Spark environment is integrated with the Alluxio Transparent URI capability. Use the following commands:

     docker exec -it spark-master bash

See that the Alluxio client jar file was added to the Spark "jars" directory (and classpath). This is required for accessing Alluxio using both the native protocol ("alluxio://") and the Transparent URI s3a protocol (s3a://).

     ls -al $SPARK_HOME/jars/*alluxio*
     -rw-r--r-- 1 1001 root 89061794 Nov 28 16:34 /opt/bitnami/spark/jars/alluxio-enterprise-2.9.0-1.0-client.jar

Next see how the Alluxio "shim" file system is configured to handle references to s3a URIs. See the properties named "fs.s3a.impl" and "fs.AbstractFileSystem.s3a.impl". Also, the "alluxio.master.hostname" property is defined to point to the Alluxio master node. If you are using Alluxio in high availability (HA) mode, with 3, 5 or 7 master nodes, then you would use the "alluxio.master.rpc.addresses" property instead.

     cat $SPARK_HOME/conf/core-site.xml

     <?xml version="1.0"?>
     <configuration>

         <!-- Enable the Alluxio Transparent URI feature for s3 and s3a end-points -->
         <property>
             <name>fs.s3a.impl</name>
             <value>alluxio.hadoop.ShimFileSystem</value>
          </property>
          <property>
            <name>fs.AbstractFileSystem.s3a.impl</name>
            <value>alluxio.hadoop.AlluxioShimFileSystem</value>
          </property>

         <!-- Enable the Alluxio Transparent URI feature for HDFS end-points -->
          <property>
            <name>fs.hdfs.impl</name>
            <value>alluxio.hadoop.ShimFileSystem</value>
          </property>
          <property>
            <name>fs.AbstractFileSystem.hdfs.impl</name>
            <value>alluxio.hadoop.AlluxioShimFileSystem</value>
          </property>

          <!-- Don't apply Transparent URI for these files -->
          <property>
            <name>alluxio.user.shimfs.bypass.prefix.list</name>
            <value></value>
          </property>

          <!-- Don't auto mount Alluxio mounts -->
          <property>
            <name>alluxio.master.shimfs.auto.mount.enabled</name>
            <value>false</value>
          </property>

          <!-- Specify the Alluxio master node -->
          <property>
            <name>alluxio.master.hostname</name>
            <value>alluxio-master</value>
          </property>

          <!-- Tell Alluxio to CACHE data when it is read for the first time -->
          <property>
            <name>alluxio.user.file.readtype.default</name>
            <value>CACHE</value>
	     </property>

          <!-- Enable client-side (northbound) impersonation in Alluxio -->
          <property>
            <name>alluxio.security.login.impersonation.username</name>
            <value>_HDFS_USER_</value>
          </property>

     </configuration>

Finally, see how Spark is configured to integrate with the Hive metastore:

     cat $SPARK_HOME/conf/spark-defaults.conf

     spark.hadoop.hive.metastore.uris=thrift://hive-metastore:9083

### Step 13. Test Spark using the Alluxio Transparent URI feature

The Alluxio Transparent URI feature will redirect references to s3 and s3a URIs to the native Alluxio URI (alluxio://). Therefore Hive table definitions with the "external_location=s3a://<bucket_name>/" will be redirected to Alluxio instead of native Minio. All the Alluxio data orchestration and data caching capabilities will be employed.

Launch a bash session in the Spark master container and run some Spark Scala commands to access the Hive table via Alluxio and also access the data file directly without using the Hive metastore.  Use these commands:

     docker exec -it spark-master bash

     spark-shell --master "spark://spark-master:7077" \
     --driver-java-options "-Dspark.hadoop.hive.metastore.uris=thrift://hive-metastore:9083"

     scala>

       // Read the Hive table using the hive metastore
       // Note: It will read via Alluxio's Transparent URI capability
	  
       import org.apache.spark.sql.hive.HiveContext

       val hiveContext = new org.apache.spark.sql.hive.HiveContext(sc)

       hiveContext.sql("USE default")

       hiveContext.sql("SHOW TABLES").show()

       hiveContext.sql("SELECT * FROM default.customer_s3a WHERE acctbal > 3500.00 AND acctbal < 9000.00 ORDER BY acctbal LIMIT 25").show()

       // Read the S3 bucket directly without using the Hive metastore
       // Note: It will read via Alluxio's Transparent URI capability

       val df=spark.read.orc("s3a://hive/warehouse/customer_s3a/").show(25)

       // Create a new table in the Hive warehouse
       val df = Seq((1, 2, 3),(2, 3, 4),(3, 4, 5)).toDF("a", "b", "c")

       df.write.saveAsTable("default.test_table")

       spark.catalog.listTables.show(false)

       spark.sql("SELECT * FROM default.test_table LIMIT 10").show() 

To view the data files created by the Spark df.write.saveAsTable() operation, go back to the Alluxio web console at:

     http://localhost:19999

And click on the "Browse" tab at the top, then click on the "hive" folder and then the "warehouse" folder. The "test_table" folder will show the parquet files created for the new table.

### Step 14. Destroy the containers

When finished, destroy the docker containers and clean up the docker volumes using these commands:

     docker-compose down

     docker volume prune

---

Please direct questions or comments to greg.palme@alluxio.com
