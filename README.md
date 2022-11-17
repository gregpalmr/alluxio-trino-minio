# alluxio-trino-minio

### Run Alluxio with Trino and Minio as the S3 object store

---

This docker compose package was based on the docker compose setup described here:

     https://github.com/bitsondatadev/trino-getting-started/tree/main/hive/trino-minio

This modified compose package deploys Trino, Alluxio, Hive metastore and Minio. It configures a Trino Hive catalog to use the Alluxio Transparent URI feature. See: 

     https://docs.alluxio.io/ee/user/stable/en/operation/Transparent-Uri.html

## USAGE:

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
     Creating mariadb           ... done
     Creating trino-coordinator ... done
     Creating alluxio-master    ... done
     Creating minio             ... done
     Creating hive-metastore    ... done
     Creating alluxio-worker1   ... done

### Step 5. Create a bucket in Minio

The minio container does not include the minio client command (mc) so you will have to use the web console to create a bucker. Follow these steps:

     - Point your web browser to http://localhost:9001

     - Log in using the user id "minio" and the password "minio123"

     - Click on the "Buckets" link in the upper left margin

     - Click on the "Create Bucket +" button on the right

     - Enter a bucket name of "bucket1" and then click on the "Create Bucket" button

### Step 6. Mount the Minio bucket in Alluxio

Open a bash session in the alluxio-master container and mount the minio bucket using the following commands:

     docker exec -it alluxio-master bash

     alluxio fs mount \
       --option alluxio.underfs.s3.endpoint=http://minio:9000 \
       --option alluxio.underfs.s3.disable.dns.buckets=true \
       --option alluxio.underfs.s3.inherit.acl=false \
       --option s3a.accessKeyId=minio \
       --option s3a.secretKey=minio123 \
       /bucket1 s3://bucket1/

     alluxio fs chmod 777 /

     alluxio fs chmod -R 777 /bucket1

If you attempt to list the files in the bucket you will get a 404 error since it is empty.

### Step 7. (Optional) Enable debug mode on the Alluxio master

Run the following command if you would like to enable debugging:

     alluxio logLevel --logName=alluxio --target master --level=DEBUG

Later, you can disable debugging mode with the command:

     alluxio logLevel --logName=alluxio --level=INFO

### Step 8. Explore the Trino configuration

Launch a bash session in the Trino coordinator container and explore the setup of the "minio" catalog and the integration with Alluxio. Use the following commands:
     
     docker exec -it trino-coordinator bash

Change to the "minio" catalog configuration directory:

     cd /etc/trino/catalog/minio

Display the minio.properties file and see the "hive.config.resources" property pointing to the Alluxio cores-site.xml file:

     cat minio.properties

     connector.name=hive-hadoop2
     hive.metastore.uri=thrift://hive-metastore:9083
     hive.s3.path-style-access=true
     hive.s3.endpoint=http://minio:9000
     hive.s3.aws-access-key=minio
     hive.s3.aws-secret-key=minio123
     hive.non-managed-table-writes-enabled=true
     hive.s3select-pushdown.enabled=true
     hive.storage-format=ORC
     hive.allow-drop-table=true
     hive.config.resources=/etc/trino/alluxio-core-site.xml

Display the Alluxio core-site.xml file contents:

     cat /etc/trino/alluxio-core-site.xml

     <?xml version="1.0"?>
     <configuration>

         <!-- Enable the Alluxio Transparent URI feature for s3 and s3a end-points -->
         <property>
             <name>fs.s3.impl</name>
             <value>alluxio.hadoop.ShimFileSystem</value>
          </property>
          <property>
            <name>fs.AbstractFileSystem.s3.impl</name>
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

          <!-- Tell Alluxio to use a block size of 16MB -->
          <property>
            <name>alluxio.user.block.size.bytes.default</name>
            <value>16MB</value>
          </property>

     </configuration>

Confirm that the Alluxio client jar file is in the Trino hive plugin directory:

     ls -al /usr/lib/trino/plugin/hive/alluxio*client.jar

### Step 9. View the Alluxio Web console

Point your web browser to the Alluxio Web console to see if any files are being cached.

     (http://localhost:19999)

Note that there is zero storage space being used to cache data. Later, when you run a Trino query, some data will be cached.

### Step 10. Test Trino using the Alluxio Transparent URI feature

The Alluxio Transparent URI feature will redirect references to the s3 and s3a URIs to the native Alluxio URI (alluxio://). Therefore Hive table definitions with the "external_location=s3a://<bucket_name>/<directory name>" will be redirected to Alluxio instead of native Minio. All the Alluxio data orchestration and data caching capabilities will be employed.

Launch a bash session in the Trino coordinator container and run a CREATE TABLE command to create a table using the "minio" Trino cagtalog setup and the "s3a" URI. Then query the data. Use these commands:

     docker exec -it trino-coordinator bash

     trino --catalog minio --debug

     USE default;

     CREATE TABLE default.customer_s3a
     WITH (
       format = 'ORC',
       external_location = 's3a://bucket1/customer_s3a/'
     ) 
     AS SELECT * FROM tpch.tiny.customer;
   
     SELECT * FROM default.customer_s3a LIMIT 500;
 
Step 10. View the Alluxio cache storage usage

When Trino queries data using Alluxio's Transparent URI feature, it will cache data to Alluxio cache storage, when it is first read from the under file system (in this case, Minio).

To see if the customer_s3a data files are cached, go back to the bash session on the alluxio-master container and list the data files. You should see that some data is cached for the cusotmer_s3a data set by the fact that the 8th column shows a percentage being cached. If it shows 0%, then Alluxio did not cache any of the data in that file.

     docker exec -it alluxio-master bash

     alluxio fs ls -R /bucket1/customer_s3a
     -rwx------  alluxio  alluxio  78509  PERSISTED 11-17-2022 22:29:47:610  0% /bucket1/customer_s3a/20221117_222944_00003_76isd_03efa9e5-a56a-4d83-8f8a-5bbdbaf9bc1f

You can also view the Alluxio Web console you launched in Step 9 to see if any data is being cached by Alluxio, or run the following Alluxio CLI command:

     alluxio fsadmin report

### Step 11. Destroy the containers

When finished, destroy the docker containers and clean up the docker volumes using these commands:

     docker-compose down

     docker volume prune


