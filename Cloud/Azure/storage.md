## Azure Storage

### Azure storage services
The Azure Storage platform includes the following data services:

- Azure Blobs: A massively scalable object store for text and binary data. Also includes support for big data analytics through Data Lake Storage Gen2.
- Azure Files: Managed file shares for cloud or on-premises deployments.
- Azure Queues: A messaging store for reliable messaging between application components.
- Azure Disks: Block-level storage volumes for Azure VMs.
- Azure Tables: NoSQL table option for structured, non-relational data.


### Benefits of Azure Storage
Azure Storage services offer the following benefits for application developers and IT professionals:

- Durable and highly available. Redundancy ensures that your data is safe if transient hardware failures occur. You can also opt to replicate data across data centers or geographical regions for additional protection from local catastrophes or natural disasters. Data replicated in this way remains highly available if an unexpected outage occurs.
- Secure. All data written to an Azure storage account is encrypted by the service. Azure Storage provides you with fine-grained control over who has access to your data.
- Scalable. Azure Storage is designed to be massively scalable to meet the data storage and performance needs of today's applications.
- Managed. Azure handles hardware maintenance, updates, and critical issues for you.
- Accessible. Data in Azure Storage is accessible from anywhere in the world over HTTP or HTTPS. Microsoft provides client libraries for Azure Storage in a variety of languages, including .NET, Java, Node.js, Python, PHP, Ruby, Go, and others, as well as a mature REST API. Azure Storage supports scripting in Azure PowerShell or Azure CLI. And the Azure portal and Azure Storage Explorer offer easy visual solutions for working with your data.


### Storage tiers

Data stored in the cloud can grow at an exponential pace. To manage costs for your expanding storage needs, it's helpful to organize your data based on attributes like frequency of access and planned retention period. Data stored in the cloud can be handled differently based on how it's generated, processed, and accessed over its lifetime. Some data is actively accessed and modified throughout its lifetime. Some data is accessed frequently early in its lifetime, with access dropping drastically as the data ages. Some data remains idle in the cloud and is rarely, if ever, accessed after it's stored. To accommodate these different access needs, Azure provides several access tiers, which you can use to balance your storage costs with your access needs.

Azure Storage offers different access tiers storage. The available access tiers include:

- <b>Hot access tier</b>: Optimized for storing data that is accessed frequently (for example, images for your website).
- <b>Cool access tier</b>: Optimized for data that is infrequently accessed and stored for at least 30 days (for example, invoices for your customers).
- <b>Cold access tier</b>: Optimized for storing data that is infrequently accessed and stored for at least 90 days.
- <b>Archive access tier</b>: Appropriate for data that is rarely accessed and stored for at least 180 days, with flexible latency requirements (for example, long-term backups).