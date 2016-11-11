# SparklingWater-azure-template

The goal of this repo is to provide an easy (click-and-go) way to deploy H2O Sparkling Water clusters on Microsoft Azure.

There are two kind of templates offered on this repo.

1. Simple:
	- HDInsight 3.5
	- Spark 1.6.2
	- Latest version of H2O Sparkling water 
	- VNet with NSG (Network Security Group)
2. Advanced:
	- HDInsight 3.5
	- Spark 1.6.2
	- Latest version of H2O Sparkling water 
	- VNet with NSG (Network Security Group)
	- Additional data source (Linked Storage Account)
	- External Hive/Oozie Metastore (SQL Database)
	

## Click the following buttons to deploy the ARM templates in the Azure Portal.         

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fpablomarin%2FSparklingWater-azure-template%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>
  

It takes about around 20 minutes to create the cluster and SQL database.


## Azure HDInsight Architecture

![HDI arch](https://acom.azurecomcdn.net/80C57D/cdn/mediahandler/docarticles/dpsmedia-prod/azure.microsoft.com/en-us/documentation/articles/hdinsight-hadoop-use-blob-storage/20160913101040/hdi.wasb.arch.png)

Hadoop supports a notion of the default file system. The default file system implies a default scheme and authority. It can also be used to resolve relative paths. During the HDInsight creation process, an Azure Storage account and a specific Azure Blob storage container from that account is designated as the default file system.

For the files on the default file system, you can use a relative path or an absolute path. For example, the *hadoop-mapreduce-examples.jar* file that comes with HDInsight clusters can be referred to by using one of the following:

	wasbs://mycontainer@myaccount.blob.core.windows.net/example/jars/hadoop-mapreduce-examples.jar
	wasbs:///example/jars/hadoop-mapreduce-examples.jar
	/example/jars/hadoop-mapreduce-examples.jar

In addition to this storage account, you can add additional storage accounts from the same Azure subscription or different Azure subscriptions during the creation process or after a cluster has been created. For instructions about adding additional storage accounts. Normally this is where your big data resides. The syntax is:

	wasb[s]://<containername>@<accountname>.blob.core.windows.net/<path>
	
HDInsight provides  also access to the distributed file system that is locally attached to the compute nodes (disks on the cluster nodes). You can use this as a local cache. Remember that this file system is gone once you delete the cluster. This file system can be accessed by using the fully qualified URI, for example:

	hdfs://<namenodehost>/<path>
	
Most HDFS commands (for example, <b>ls</b>, <b>copyFromLocal</b> and <b>mkdir</b>) still work as expected. Only the commands that are specific to the native HDFS implementation (which is referred to as DFS), such as <b>fschk</b> and <b>dfsadmin</b>, will show different behavior in Azure Blob storage.

Only the data on the linked storage account and the external hive meta-store will persist after the cluster is deleted. <b>MAKE SURE YOU DO NOT STORE YOUR IMPORTANT DATA ON THE DEFAULT STORAGE ACCOUNT CREATED BY THE CLUSTER</b>.


## H2O Sparkling Water Architecture

![Sparkling architecture](http://www.ibmbigdatahub.com/sites/default/files/quality_of_life_fig_1.jpg)



## Create Jupyter notebook with PySpark kernel 

In this section, you use Jupyter notebook to run Spark SQL queries against the Spark cluster. HDInsight Spark clusters provide two kernels that you can use with the Jupyter notebook. These are:

* **PySpark** (for applications written in Python)
* **Spark** (for applications written in Scala)

In this article, you will use the PySpark kernel. Couple of key benefits of using the PySpark kernel are:

* You do not need to set the contexts for Spark and Hive. These are automatically set for you.
* You can use cell magics, such as `%%sql`, to directly run your SQL or Hive queries, without any preceding code snippets.
* The output for the SQL or Hive queries is automatically visualized.


1. From the [Azure Portal](https://portal.azure.com/), from the startboard, click the tile for your Spark cluster (if you pinned it to the startboard). You can also navigate to your cluster under **Browse All** > **HDInsight Clusters**.   

2. From the Spark cluster blade, click **Quick Links**, and then from the **Cluster Dashboard** blade, click **Jupyter Notebook**. If prompted, enter the admin credentials for the cluster.

	> you may also reach the Jupyter Notebook for your cluster by opening the following URL in your browser. Replace __CLUSTERNAME__ with the name of your cluster:
	>
	> `https://CLUSTERNAME.azurehdinsight.net/jupyter`


## Where are the notebooks stored?

Jupyter notebooks are saved to the storage account associated with the cluster under the **/HdiNotebooks** folder.  Notebooks, text files, and folders that you create from within Jupyter will be accessible from WASB.  For example, if you use Jupyter to create a folder **myfolder** and a notebook **myfolder/mynotebook.ipynb**, you can access that notebook at `wasbs:///HdiNotebooks/myfolder/mynotebook.ipynb`.  The reverse is also true, that is, if you upload a notebook directly to your storage account at `/HdiNotebooks/mynotebook1.ipynb`, the notebook will be visible from Jupyter as well.  Notebooks will remain in the storage account even after the cluster is deleted.

The way notebooks are saved to the storage account is compatible with HDFS. So, if you SSH into the cluster you can use file management commands like the following:

	hdfs dfs -ls /HdiNotebooks             				  # List everything at the root directory – everything in this directory is visible to Jupyter from the home page
	hdfs dfs –copyToLocal /HdiNotebooks    				# Download the contents of the HdiNotebooks folder
	hdfs dfs –copyFromLocal example.ipynb /HdiNotebooks   # Upload a notebook example.ipynb to the root folder so it’s visible from Jupyter


In case there are issues accessing the storage account for the cluster, the notebooks are also saved on the headnode `/var/lib/jupyter`.



## Delete the cluster

To delete the Sparkling Water Cluster, go to the portal and delete the Resource Group.

## Tips for Executor memory allocation 

Assume there are 6 nodes available on a cluster with 25 core nodes and 125 GB memory per node. It is natural to try to utilize those resources as much as possible for your Sparkling Water application, before considering requesting more nodes (which might result in longer wait times in the queue and overall longer times to get the result). 

With YARN, a possible approach would be to use --num-executors 6 --executor-cores 24 --executor-memory 124G. Here, we subtracted 1 core and some memory per node to allow for operating system and/or cluster specific daemons to run. However, this approach would be not be optimal, because large number of cores per executor leads to HDFS I/O throughput and thus significantly slow down the application. Allocating a similar number of cores would be possible by increasing the number of executors and decreasing the number of executor-cores and memory.

A recommended approach when using YARN would be to use --num-executors 30 --executor-cores 4 --executor-memory 24G. Which would result in YARN allocating 30 containers with executors, 5 containers per node using up 4 executor cores each. The RAM per container on a node 124/5= 24GB (roughly).

This is the formula:<br>
usable_mem = mem_per_node - 1G<br>
usable_cores = cores_per_node - 1<br>
n = 3 or 5 (make it so the number of cores per executor should be maximum 5)<br>
<br>
num-executors = no_nodes x n - 1<br>
executor-cores = usable_cores / n<br>
executor-memory = usable_mem * 0.97 / n <br>





### Data source ###

The original Hadoop distributed file system (HDFS) uses many local disks on the cluster. HDInsight uses Azure Blob storage for data storage. Azure Blob storage is a robust, general-purpose storage solution that integrates seamlessly with HDInsight. Through an HDFS interface, the full set of components in HDInsight can operate directly on structured or unstructured data in Blob storage. Storing data in Blob storage helps you safely delete the HDInsight clusters that are used for computation without losing user data.

During configuration, you must specify an Azure storage account and an Azure Blob storage container on the Azure storage account. Some creation processes require the Azure storage account and the Blob storage container to be created beforehand. The Blob storage container is used as the default storage location by the cluster. Optionally, you can specify additional Azure Storage accounts (linked storage) that will be accessible by the cluster. The cluster can also access any Blob storage containers that are configured with full public read access or public read access for blobs only.  For more information, see [Manage Access to Azure Storage Resources](../storage/storage-manage-access-to-resources.md).

![HDInsight storage](./media/hdinsight-provision-clusters/HDInsight.storage.png)

> A Blob storage container provides a grouping of a set of blobs as shown in the following image.

![Azure blob storage](./media/hdinsight-provision-clusters/Azure.blob.storage.jpg)

We do not recommended using the default Blob storage container for storing business data. Deleting the default Blob storage container after each use to reduce storage cost is a good practice. Note that the default container contains application and system logs. Make sure to retrieve the logs before deleting the container.

>Sharing one Blob storage container for multiple clusters is not supported.

For more information on using secondary Blob storage, see [Using Azure Blob Storage with HDInsight](hdinsight-hadoop-use-blob-storage.md).

In addition to Azure Blob storage, you can also use [Azure Data Lake Store](../data-lake-store/data-lake-store-overview.md) as a default storage account for HBase cluster in HDInsight and as linked storage for all four HDInsight cluster types. For more information, see [Create an HDInsight cluster with Data Lake Store using Azure Portal](../data-lake-store/data-lake-store-hdinsight-hadoop-use-portal.md).

### Location (Region) ###

The HDInsight cluster and its default storage account must be located at the same Azure location.

![Azure regions](./media/hdinsight-provision-clusters/Azure.regions.png)

For a list of supported regions, click the **Region** drop-down list on [HDInsight pricing](https://go.microsoft.com/fwLink/?LinkID=282635&clcid=0x409).

## Use additional storage

In some cases, you may wish to add additional storage to the cluster. For example, you might have multiple Azure storage accounts for different geographical regions or different services, but you want to analyze them all with HDInsight.

You can add storage accounts when you create an HDInsight cluster or after a cluster has been created.  See [Customize Linux-based HDInsight clusters using Script Action](hdinsight-hadoop-customize-cluster-linux.md).

For more information about secondary Blob storage, see [Using Azure Blob storage with HDInsight](hdinsight-hadoop-use-blob-storage.md). For more information about secondary Data Lake Storage, see [Create HDInsight clusters with Data Lake Store using Azure Portal](../data-lake-store/data-lake-store-hdinsight-hadoop-use-portal.md).

##<a name="next-steps"></a>What components are included as part of a Spark cluster?

Spark in HDInsight includes the following components that are available on the clusters by default.

- [Spark Core](https://spark.apache.org/docs/1.5.1/). Includes Spark Core, Spark SQL, Spark streaming APIs, GraphX, and MLlib.
- [Anaconda](http://docs.continuum.io/anaconda/)
- [Livy](https://github.com/cloudera/hue/tree/master/apps/spark/java#welcome-to-livy-the-rest-spark-server)
- [Jupyter Notebook](https://jupyter.org)

Spark in HDInsight also provides an [ODBC driver](http://go.microsoft.com/fwlink/?LinkId=616229) for connectivity to Spark clusters in HDInsight from BI tools such as Microsoft Power BI and Tableau.

## Where do I start?

Start with creating a Spark cluster on HDInsight Linux. See [QuickStart: create a Spark cluster on HDInsight Linux and run sample applications using Jupyter](hdinsight-apache-spark-jupyter-spark-sql.md). 


## Use Hive/Oozie metastore

We strongly recommend that you use a custom metastore if you want to retain your Hive tables after you delete your HDInsight cluster. You will be able to attach that metastore to another HDInsight cluster.

> HDInsight metastore is not backward compatible. For example, you cannot use a metastore of an HDInsight 3.4 cluster to create an HDInsight 3.3 cluster.

The metastore contains Hive and Oozie metadata, such as Hive tables, partitions, schemas, and columns. The metastore helps you to retain your Hive and Oozie metadata, so you don't need to re-create Hive tables or Oozie jobs when you create a new cluster. By default, Hive uses an embedded Azure SQL database to store this information. The embedded database can't preserve the metadata when the cluster is deleted. When you create Hive table in an HDInsight cluster with an Hive metastore configured, those tables will be retained when you recreate the cluster using the same Hive metastore.

Metastore configuration is not available for HBase cluster types.

> When creating a custom metastore, do not use a database name that contains dashes or hyphens. This can cause the cluster creation process to fail.

### Manage resources

* [Manage resources for the Apache Spark cluster in Azure HDInsight](hdinsight-apache-spark-resource-manager.md)

* [Track and debug jobs running on an Apache Spark cluster in HDInsight](hdinsight-apache-spark-job-debugging.md)


## Use external packages with Jupyter notebooks 

1. From the [Azure Portal](https://portal.azure.com/), from the startboard, click the tile for your Spark cluster (if you pinned it to the startboard). You can also navigate to your cluster under **Browse All** > **HDInsight Clusters**.   

2. From the Spark cluster blade, click **Quick Links**, and then from the **Cluster Dashboard** blade, click **Jupyter Notebook**. If prompted, enter the admin credentials for the cluster.

	> You may also reach the Jupyter Notebook for your cluster by opening the following URL in your browser. Replace __CLUSTERNAME__ with the name of your cluster:
	>
	> `https://CLUSTERNAME.azurehdinsight.net/jupyter`

2. Create a new notebook. Click **New**, and then click **Spark**.

	![Create a new Jupyter notebook](https://raw.githubusercontent.com/Azure/azure-content/master/articles/hdinsight/media/hdinsight-apache-spark-jupyter-notebook-use-external-packages/hdispark.note.jupyter.createnotebook.png "Create a new Jupyter notebook")

3. A new notebook is created and opened with the name Untitled.pynb. Click the notebook name at the top, and enter a friendly name.

	![Provide a name for the notebook](https://raw.githubusercontent.com/Azure/azure-content/master/articles/hdinsight/media/hdinsight-apache-spark-jupyter-notebook-use-external-packages/hdispark.note.jupyter.notebook.name.png "Provide a name for the notebook")

4. You will use the `%%configure` magic to configure the notebook to use an external package. In notebooks that use external packages, make sure you call the `%%configure` magic in the first code cell. This ensures that the kernel is configured to use the package before the session starts.

		%%configure
		{ "packages":["com.databricks:spark-csv_2.10:1.4.0"] }


	>if you forget to configure the kernel in the first cell, you can use the `%%configure` with the `-f` parameter, but that will restart the session and all progress will be lost.

5. In the snippet above, `packages` expects a list of maven coordinates in Maven Central Repository. In this snippet, `com.databricks:spark-csv_2.10:1.4.0` is the maven coordinate for **spark-csv** package. Here's how you construct the coordinates for a package.

	a. Locate the package in the Maven Repository. For this tutorial, we use [spark-csv](http://search.maven.org/#artifactdetails%7Ccom.databricks%7Cspark-csv_2.10%7C1.4.0%7Cjar).
	
	b. From the repository, gather the values for **GroupId**, **ArtifactId**, and **Version**.

	![Use external packages with Jupyter notebook](https://raw.githubusercontent.com/Azure/azure-content/master/articles/hdinsight/media/hdinsight-apache-spark-jupyter-notebook-use-external-packages/use-external-packages-with-jupyter.png "Use external packages with Jupyter notebook")

	c. Concatenate the three values, separated by a colon (**:**).

		com.databricks:spark-csv_2.10:1.4.0

6. Run the code cell with the `%%configure` magic. This will configure the underlying Livy session to use the package you provided. In the subsequent cells in the notebook, you can now use the package, as shown below.

		val df = sqlContext.read.format("com.databricks.spark.csv").
        option("header", "true").
        option("inferSchema", "true").
        load("wasbs:///HdiSamples/HdiSamples/SensorSampleData/hvac/HVAC.csv")

7. You can then run the snippets, like shown below, to view the data from the dataframe you created in the previous step.

		df.show()

		df.select("Time").count()


## <a name="seealso"></a>See also


* [Overview: Apache Spark on Azure HDInsight](hdinsight-apache-spark-overview.md)
