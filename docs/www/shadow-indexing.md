---
title: Data Management
order: 1
---

Data Management
Header in the menu bar - contains Shadow Indexing and Data Archiving (https://vectorized.io/docs/data-archiving) pages. 


Shadow Indexing
Shadow indexing is a multi-tiered remote storage solution that provides the ability to archive log segments to a cloud object store in real time as the topic is being produced. You can recover a topic that no longer exists in the cluster, and replay and read log data as a stream directly from cloud storage even if it doesn’t exist in the cluster. Shadow indexing provides a disaster recovery plan that takes advantage of infinitely scalable storage systems, is easy to configure, and works in real time.

Shadow indexing uses the following topic configuration flags: 

redpanda.remote.write - Uploads data from Redpanda to cloud storage. 
redpanda.remote.read - Fetches data from cloud storage to Redpanda. This allows clients to fetch the data from Redpanda.   
redpanda.remote.recovery - Recovers or reproduces a topic from cloud storage. Use this flag during topic creation. 

To enable shadow indexing, you can set the redpanda.remote.write and redpanda.remote.read flags on a new topic or an existing topic. Use the following command to create a new topic with shadow indexing enabled: 

rpk topic create <topic-name> -c redpanda.remote.read=true -c redpanda.remote.write=true

And use this command to enable shadow indexing on an existing topic: 

rpk topic alter-config <topic-name> --set redpanda.remote.read=true --set redpanda.remote.write=true
Configuring Shadow Indexing
Shadow indexing is supported for Amazon S3 and Google Cloud Storage. Before you enable shadow indexing for a topic, you must configure cloud storage in the redpanda.yaml file. 

The sections below give detailed information on all of the shadow indexing configuration options, but the parameters listed here are the minimum required parameters to use shadow indexing: 

For Amazon S3, you must configure these parameters: 

cloud_storage_access_key: ***
cloud_storage_api_endpoint: storage.googleapis.com
cloud_storage_bucket: pandabucket
cloud_storage_enabled: true
cloud_storage_region: eu-north-1
cloud_storage_secret_key: ***

For Google Cloud Storage, you must configure these parameters: 

cloud_storage_enabled: true
cloud_storage_access_key: ***
cloud_storage_secret_key: ***
cloud_storage_region: eu-north-1
cloud_storage_bucket: pandabucket

Remote write
Remote write is the process that constantly uploads log segments to cloud storage. The process is created for each partition and runs on the leader node of the partition. It only uploads the segments that contain only batches with offsets that are smaller than the last stable offset. This is the largest offset that the client can read. 

To enable shadow indexing, use remote write in conjunction with remote read. If you only enable remote write on a topic, you will have a simple backup that you will still be able to run recovery on. 

To create a topic with remote write enabled, use this command: 

rpk topic create <topic-name> -c -c redpanda.remote.write=true

And to enable remote write on an existing topic, use this command: 

rpk topic alter-config <topic-name> --set redpanda.remote.write=true
Idle timeout
You can configure Redpanda to start a remote write periodically. This is useful if the ingestion rate is low and the segments are kept open for long periods of time. You specify a number of seconds for the timeout, and if that time has passed since the previous write and the partition has new data, Redpanda will start a new write. This limits data loss in the event of catastrophic failure and adds a guarantee that you will only lose the specified number of seconds of data. 

Use the cloud_storage_segment_max_upload_interval_sec parameter in the redpanda.yaml file to set the number of seconds for idle timeout. By default, the parameter is empty. 
Reconciliation 
The reconciliation loop runs constantly to check which partition replicas are leaders on the node. If a new leader is elected on a node, the remote write process starts for that partition on the node. If an existing leader loses leadership, the reconciliation loop stops the upload process for the node.  This means that the redpanda node will start archiving new partition or recognize any change in leadership only when the reconciliation loop will kick in.

Reconciliation is configured with the following parameters in the redpanda.yaml file: 

cloud_storage_reconciliation_interval_ms - Sets the interval, in milliseconds, that is used to reconcile partitions that need to be uploaded. Default is 10000ms.
cloud_storage_max_connections - The maximum number of connections to cloud storage on a node per CPU. Default is 20. 
Upload controller
Remote write uses a proportional derivative (PD) controller to minimize the backlog size for the write. The backlog consists of the data that has not been uploaded to cloud storage, but must be uploaded eventually. 

The upload controller prevents Redpanda from running out of disk space. If remote.write is set to true, Redpanda cannot evict log segments that have not been uploaded to cloud storage. If the remote write process cannot keep up with the amount of data that needs to be uploaded to cloud storage, the upload controller increases priority for the upload process. The upload controller measures the size of the upload periodically and tunes the priority of the remote write process. 

The upload controller is configured with the following parameters in the redpanda.yaml file. Under normal circumstances, you will not need to change these values from their defaults: 

cloud_storage_upload_ctrl_update_interval_ms - The recompute interval for the upload controller. Default is 60000ms. 
cloud_storage_upload_ctrl_p_coeff - The proportional coefficient for the upload controller. Default is -2. 
cloud_storage_upload_ctrl_d_coeff - The derivative coefficient for the upload controller. Default is 0. 
cloud_storage_upload_ctrl_min_shares - The minimum number of I/O and CPU shares that the remote write process can use. Default is 100. 
cloud_storage_upload_ctrl_max_shares - The maximum number of I/O and CPU shares that the remote write process can use. Default is 1000. 
Remote read
Remote read fetches data from cloud storage using the Kafka API. Use remote read in conjunction with remote write to enable shadow indexing. If you use remote read without remote write, there will be nothing for Redpanda to read. 

Normally, when data is evicted locally, it is no longer available. If the consumer starts consuming the partition from the beginning, the first offset will be the smallest offset available locally. However, if shadow indexing is enabled with the redpanda.remote.read and redpanda.remote.write flags, the data is always uploaded to remote storage before it is deleted. This guarantees that the data is always available either locally or remotely. 

When data is available remotely and shadow indexing is enabled, the client can start consuming data from offset 0 even if the data is no longer stored locally. 

To create a topic with remote read enabled, use this command: 

rpk topic create <topic-name> -c -c redpanda.remote.read=true

And to enable remote read on an existing topic, use this command: 

rpk topic alter-config <topic-name> --set redpanda.remote.read=true
Caching 
When the Kafka client fetches an offset range that isn’t available locally in the Redpanda data directory, Redpanda downloads remote segments from cloud storage. These downloaded segments are stored in the cloud storage cache directory. The cache directory is created in the Redpanda data directory by default, but it can be placed anywhere in the system. For example, you might want to put the cache directory to a dedicated drive with cheaper storage. 

If you don’t specify a cache location in the redpanda.yaml file, the cache directory will be created here: 

<redpanda-data-directory>/cloud_storage_cache. 

Use the cloud_storage_cache_directory parameter in the redpanda.yaml file to specify a different location for the cache directory. You must specify the full path. 

Redpanda checks the cache periodically, and if the size of the stored data is larger than the configured limit, the eviction process starts. The eviction process removes segments that haven’t been accessed recently, until the size of the cache drops 20%. Use the following parameters in the redpanda.yaml file to set the maximum cache size and cache check interval: 

cloud_storage_cache_size - Maximum size of the disk cache that is used by shadow indexing. Default is 20GiB.
cloud_storage_cache_check_interval - The time, in milliseconds, between cache checks. The size of the cache can grow quickly, so it’s important to have a small interval between checks, but if the checks are too frequent, they will consume a lot of resources. Default is 30000ms. 
Remote Recovery 
When you create a topic, you can use remote recovery to download the topic data from cloud storage. Only the data that matches the retention policy of the topic will be downloaded. The data that is not downloaded from cloud storage will still be accessible through remote read. 

You can use remote recovery to restore a topic that was deleted from a cluster, or you can replicate a topic in another cluster. 

Use the following command to create a new topic using remote recovery: 

rpk topic create <topic-name> -c redpanda.recovery=true

You can also create a new topic using remote recovery and enable shadow indexing on the new topic by adding the redpanda.remote.write and redpanda.remote.read flags: 

rpk topic create <topic-name> -c redpanda.recovery=true -c redpanda.remote.write=true -c redpanda.remote.read=true
Retries and backoff 
If the cloud provider replies with an error message that the server is busy, Redpanda will retry the request. Redpanda always uses exponential backoff with cloud connections. 

Redpanda retries the request if it receives any the following errors: 

Connection refused
Connection reset by peer
Connection timed out
Slow down REST API error

For other errors, Redpanda will not retry the request. For example, Redpanda will not retry the request after a NoSuchKey error. 

In the redpanda.yaml file, you can configure the cloud_storage_initial_backup_ms parameter to set the time, in milliseconds, that is used as an initial backoff interval in the exponential backoff algorithm that is used to handle an error. The default is 100ms. 
HTTP and TLS implementation 
Shadow indexing creates a connection pool for each CPU. It also uses persistent HTTP connections with a configurable maximum idle time. Custom cloud storage clients are used to send and receive data. 

For normal usage, you will not need to configure the HTTP parameters and the Redpanda defaults will be sufficient and the certificates used to connect to the cloud storage client are available through private key infrastructure. Redpanda detects the location of the certificates on any GNU Linux distribution automatically. 

Redpanda uses the following parameters in the redpanda.yaml file to configure HTTP and TLS: 

cloud_storage_segment_upload_timeout_ms - Timeout for segment upload. Redpanda retries the upload after the timeout. Default is 30000ms. 
cloud_storage_manifest_upload_timeout_ms - Timeout for manifest upload. Redpanda retries the upload after the timeout. Default is 10000ms. 
cloud_storage_max_connection_idle_time_ms - The maximum idle time for persistent HTTP connections. Differs depending on the cloud provider. AWS uses a 5s interval. Google Cloud Storage uses a much larger interval. Default is 5000ms, which will be sufficient for most providers. 
cloud_storage_segment_max_upload_interval_sec - Sets the number of seconds for idle timeout. By default, the parameter is empty. 
Configuration Parameters
The list below contains all the available configuration parameters for shadow indexing in the redpanda.yaml file. 

cloud_storage_enabled - Global flag that enables shadow indexing. Default is false. 
cloud_storage_access_key - Cloud storage access key. Required. 
cloud_storage_secret_key - Cloud storage secret key. Required. 
cloud_storage_region - Cloud storage region. Required. 
cloud_storage_bucket - Cloud storage bucket name. Required. 
cloud_storage_api_endpoint - API endpoint. This can be left blank for AWS, where it’s generated automatically using the region and bucket. For Google Cloud Service, use storage.googleapis.com. 
cloud_storage_reconciliation_interval_ms - Sets the interval, in milliseconds, that is used to reconcile partitions that need to be uploaded. Default is 10000ms.
cloud_storage_max_connections - The maximum number of connections to cloud storage on a node per CPU. Default is 20. 
cloud_storage_disable_tls - Disables TLS encryption. You can set this to true if TLS termination is done by the proxy. Default is false. 
cloud_storage_api_endpoint_port - Overrides the default port for the TLS connection. Default is 443. 
cloud_storage_trust_file - The public certificate that is used to validate the TLS connection to cloud storage. If this is empty, Redpanda will use PKI. Default is auto. 
cloud_storage_initial_backoff_ms - The time, in milliseconds, that is used as an initial backoff interval in the exponential backoff algorithm that is used to handle an error. The default is 100ms. 
cloud_storage_segment_upload_timeout_ms - Timeout for segment upload. Redpanda retries the upload after the timeout. Default is 30000ms. 
cloud_storage_manifest_upload_timeout_ms - Timeout for manifest upload. Redpanda retries the upload after the timeout. Default is 10000ms. 
cloud_storage_max_connection_idle_time_ms - The maximum idle time for persistent HTTP connections. Differs depending on the cloud provider. AWS uses a 5s interval. Google Cloud Storage uses a much larger interval. Default is 5000ms, which will be sufficient for most providers. 
cloud_storage_segment_max_upload_interval_sec - Sets the number of seconds for idle timeout. By default, the parameter is empty. 
cloud_storage_upload_ctrl_update_interval_ms - The recompute interval for the upload controller. Default is 60000ms.
cloud_storage_upload_ctrl_p_coeff - The proportional coefficient for the upload controller. Default is -2. 
cloud_storage_upload_ctrl_d_coeff - The derivative coefficient for the upload controller. Default is 0. 
cloud_storage_upload_ctrl_min_shares - The minimum number of I/O and CPU shares that the remote write process can use. Default is 100. 
cloud_storage_upload_ctrl_max_shares - The maximum number of I/O and CPU shares that the remote write process can use. Default is 1000. 
cloud_storage_cache_size - Maximum size of the disk cache that is used by shadow indexing. Default is 20GiB.
cloud_storage_cache_check_interval - The time, in milliseconds, between cache checks. The size of the cache can grow quickly, so it’s important to have a small interval between checks, but if the checks are too frequent, they will consume a lot of resources. Default is 30000ms. 
cloud_storage_cache_directory - The directory for the shadow indexing cache. You must specify the full path. Default is <redpanda-data-directory>/cloud_storage_cache. 



