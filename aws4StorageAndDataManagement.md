# Introduction to S3 (**important for exam**)
* Definition: durable, high-scalable storage.
* Characteristics:
  * Simple WS to store and retrieve any amount of data from anywhere on the web. Not designed for database or OS.
  * Object based storage (<> block storage): mean objects/files are the unit of storage. Additional info are available: metadata and identifier...
  * Data is spread across multi devices and facilities(=building). Disk can crash and data will still available for end user.
    * Amazon S3 `PUT` and `PUT Object copy` operations synchronously store your data across multiple facilities before returning `SUCCESS`
  * Files: between 0 bytes to 5TB
  * Unlimited storage (nothing to manage to extend disk...)
  * Files are stored in buckets
  * S3 bucket/namespace names must be unique globally (even cross region, even between AWS account)
  * Different tiers/classes storage available depending of your requirements
  * Lifecycle management (see next chapters)
  * Versioning (if activated: call rollback files)
  * Encryption
  * Bucket policies, access control list => control access to your bucket/files
  * Bucket are created in a specified regions. **Objects are sync across all AZs in the region (extreme HA and durability).** Region should be chose carefully to be near from customers / EC2 instances.
* Data consistency model for S3:
  * Create (PUT) file in S3: after execution, the file can be directly read
  * Update (PUT) file or delete (DELETE) file in S3: after execution, the file is not synchronously updated/deleted. Can take a while.
* S3 is simple key-value storage:
  * Key: name of the object/file
  * Value: data of the object/file
  * Version ID: for versioning
  * Metadata: can be user-define
  * Bucket specific configuration (=sub-resources):
    * Bucket policies, access control list
    * Cross origin resources sharing (CORS) => access between buckets
    * Transfer acceleration => service to accelerate transfer speed when upload lot of files for long distance. Client uses nearest CloudFront to upload files and then CloudFront upload files in S3 in high speed with fast connection.
* Tiers/classes of storage (can be different by objects/files inside the same bucket):
  * **Standard**: Stored across devices/facilities. Up to 2 facilities can crash and data still available. Immediately available. Frequently accessed. Most expensive class.
    * 99.99% of availability for one year (! Amazon guaranteed only 99.9%).
    * 99.999999999% of durability.
  * **Infrenquently Accessed (S3-IA / Standard-IA)**: pay when retrieve. Rapid access to file when needed. Less expensive than standard and RRS and EBS volume.
    * 99.9% of availability for one year
    * 99.999999999% of durability.  
  * **One zone IA**: pay when retrieve. Data stored in a single AZ. Cost is 20% less than S3 - IA.
    * 99.5% of availability for one year
    * 99.999999999% of durability.
  * **Reduced Redundancy Storage - RRS**: can be used for thumbnails. Less expensive then standard.
    * 99.99% of availability for one year
    * 99.99% of durability
  * **Glacier**: very cheap and can be used for archival (_not to be used for backup !_). Take 3-5 hours to restore from Glacier. Pay when retrieve.
    * 99.99% of availability for one year (after your restore the objects)
    * 99.999999999% of durability.
  * **Intelligent tiering**
    * Designed for unknown or unpredictable access patterns
    * Composed of 2 tiers: frequent and infrequent access. Objects will be moved automatically on most suitable tiers. Objects not accessed during 30 days: move in infrequent tiers. If object accessed from infrequent: moved in frequent tier.
    * Allow to optimize the costs. No fees for access but monthly costs for automation (0.0025$ / 1000 objects).
    * 99.9% of availability for one year
    * 99.999999999% of durability.
* Charges:
  * Storage per GB
  * Number of request (GET, PUT, COPY...)
  * Storage management pricing: inventory, analytics, objects tags
  * Data management pricing: data transferred out of S3 (transfer IN is free)
  * Transfer acceleration
* Example s3 bucket link: https://s3-eu-west-1.amazonaws.com/acloudguru
* Useful link: https://aws.amazon.com/s3/faqs/

# S3 - lab
* Create S3 bucket:
  * Can enable/disable versioning.
  * Can enable/disable access log (you are charged for logs files storage only).
  * Can use tags to track project costs.
  * Can enable/disable API call tracing via CloudTrail (additional costs)
  * Can enable encryption (only at server-side)
  * Can enable CloudWatch for monitor requests (additional costs)
  * Tips: public access to S3 bucket is disable by default
* Files can be make public individually
* Files can be changed from tiers/classes storage individually

# S3 lifecycle policies
* Lifecylce management: define rules which can move(=transition action) an object from standard tiers to another tiers to reduce cost. Rule can also be a delete(=expire action) of objects. Lifecycle rules are based on object creation date. Examples:
  * Files older than 90 days: moved/transition in IA storage
    * _Note_: move to "RRS" is not possible via a lifecycle. Can be only chosen during object creation or can be edited manually by object.
  * Archive objects created one year ago
  * Auto delete files older than one year (e.g. access log files to S3 bucket).

# S3 Cross region replication
* Cross region replication is useful if all region fail (war...)
* Replication should be activated at bucket-level
* Cross region copy is automatic when activated and asynchronous. Copy can be limited to some tag or prefix (more or less folder name) value.
* Retain during region copy: storage class (can be changed if requested), object names, owners, permissions
* Not replicated:
  * Objects which exist before activation of replica
  * Encrypted objects using private provided customer key (SSE-C)
  * Encrypted objects using KMS managed keys (SSE-KMS) but can be explicitly requested
  * Lifecycle policy
  * Objects already replicated in source bucket: objects can be replicated only once.

# S3 versioning and MFA delete
* Versioning:
  * S3 versioning enables you to revert to older versions of S3 objects.
  * Multiple version of an objects are stored in the same bucket.
  * Versioning protect you against accidental/malicious delete (can un-delete).
  * With versioning: DELETE action doesn't delete the object version but mark it as deleted. Still possible to permanently delete object by providing version ID in DELETE request.
* If MFA (=multi-factor authentication) delete is enable, you get additional layer of protection on your version-controlled S3 bucket:
  * You need to MFA device code to permanently delete an object version
  * MFA device code is also needed to enable/disable versioning on a S3 bucket
  * _Note_: same MFA device can be use to login in AWS account.
  * Activate MFA delete: `aws s3api put-bucket-versioning -profile MasterUser -bucket MyVersionBucket -versioning-configuration MFADelete=Enabled,Status=Enabled -mfa 'arn:…. 012345'`

# S3 encryption
* Term: Cipher (=chiffrement)
* Encryption possibilities:
  * **Encryption during transit**: SSL/TLS (https): it is client side encryption
  * **Encrypt stored data (=at rest)**: it's a server side encryption. Three types:
    * **SSE-S3**: S3 managed key, each object is encrypted with his own unique data key. Data key is also encrypted by a master key. All is managed by Amazon.
      * _Note:_ cipher a key by a master key is a process called enveloping.
    * **SSE-KMS**: data key is managed by Amazon. Same as SSE-S3 but you manage the Custmer Master Key (CMK) through AWS Key Management Service (AWS KMS. Master key can be visible to some users or not depending of associate permissions. Moreover, audit is provided to know who access to the data.
    * **SSE-C**: server side encrypted based on key managed by customer
* When create a key (IAM menu): you can decide who (user or role) can administrate the key and also who can use the key.
* How to encrypt at upload:
  * Provide HTTP header parameter `x-amz-server-side-encryption` with value `AES256` (SSE-S3 - S3 managed key) or with value `aws:kms` (SSE-KMS - KMS managed key)
  * _Note_: you can enforce encryption during PUT request (also applied to AWS Console UI) by define a Bucket Policy which denied request without this parameter.

# S3 encryption - lab
* Create S3 bucket:
  * Can choose encryption for object stored on S3: AES-256 or AWS-KMS. If not chosen, you can enforce encryption with Bucket Policies.
* Create Bucket policy (under permission tab):
  * Use policy generator and created policy for all requests (principal: \*), effect: deny, actions: s3:PutObject, ARN: bucket ARN id, condition: string not equals `x-amz-server-side-encryption`/`aws:kms`

# EC2 volume types (EBS volume vs. instance store volume)
* Two volume types:
  * **EBS** (Elastic Block Storage) volume: persistence/permanently storage. Can be detach and re-attach to another instance. Can take backed up and fully encrypted.
  * **Instance store** volume (very uncommon): hardly linked to the instance (instance store persists during lifetime of its associate instance (even after reboot) but not in case of terminated instance.). Ideal for temporary storage because data is lost after termination or hardware failure.
* Volumes characteristics:
  * Root(=boot) volume can be either EBS or instance store volume
  * EBS root volume is deleted at termination expect if, during creation, you uncheck "Delete on termination" in console or set `deleteontermination` flag to false in command line. Cannot be unchecked for instance store root volume.
  * Root volume limitation:
    * Instance store root device volume is limited to 10Gb
    * EBS root device volume can be up to 1Tb-2Tb (depends of OS)
  * Additional volume (E:, F:, /dev/sdb...): not deleted at termination expect for instance store volume. Don't forget to delete them manually to avoid charges.
* Stop instance information depending of root volume type:
  * EBS backed instance can be stopped
  * Instance stored backed instance cannot be stopped. Only rebooted or terminated. Consequence: cannot change RAM, kernel...
* Don't rely on instance store (used for cache/buffer). Instead of use EBS volume or S3.
* When you create a new volume and attach it to an instance: the volume doesn't have file system and is not deleted at termination. Instructions to check and create file system:
  * 1) `sudo file -s /dev/xvdf` return 'data': mean no file system
  * 2) `sudo mkfs -t ext4 /dev/xvdf` create ext4 file system
  * 3) `sudo file -s /dev/xvdf` return '...ext4...'
  * 4) `mkdir mount_point`
  * 5) `sudo mount /dev/xvdf mount_point`


# EBS volumes - lab (**important for exam**)
* EC2 instance and associate EBS volumes are ALWAYS in the same AZ.
* Root volume (gp2) can be upgraded to another volume type (io1...) and/or size of disk can be updated with no down time on EC2. Attention: partition needs to be resized in OS to take into account new size.
* All others EBS volume types can also be modified with no down time.
* A root volume can be switched with another volume but instance must be stopped first. During volume attachment, you must specify device to `/dev/xvda` to inform that it is the boot volume.
  * _Note_: it is also possible to define a new "Auto-scaling Launch Configuration" with new desired characteristics of the EBS volume. Then, on  the auto scaling group (ASG), you can change the launch configuration and select the new one. Finally, you can terminate the instances linked to this ASG in order the ASG automatically restart new instances with the new volume characteristics.
* Snapshots information:
  * They are stored on S3 (no visible, 99.999999999% of durability).
  * Snapshots are incremental based on previous snapshot. First snapshot can take some times to be created.
  * Snapshot of encrypted volume are encrypted automatically. You cannot create encrypted snapshot from a non-encrypted volume.
  * Volume restored from encrypted snapshots are encrypted automatically.
  * From snapshot, you can
    * create a new volume with new characteristics of volume type/size, etc.
    * create an AMI and use it to create new EC2 instance. Tags, security groups, etc. are not copied in AMI: you should re-specify them during creation of EC2.
  * You can shared snapshot only if they are un-encrypted. Share can be done to others AWS account or made public.
  * No automatic backup/snapshot but can be done thanks to CloudWatch event, API, AWS CLI or Lambda (_more details in next chapter_).
    * _News_: can be automatically created with "AWS Data Lifecycle Manager".
  * Snapshot decrease performance.
* Move region/AZ details:
  * Move volume in another AZ of same region:
    * 1) Create a (volume) snapshot
    * 2) Create volume from this (volume) snapshot and choose the target AZ.
    * 3) You can attach this volume only to EC2 instances in this AZ.
  * Move volume in another region:
    * 1) Create snapshot from the volume.
    * 2) Copy this snapshot in another region to use it in the target region.
    * 3) Create a volume from a snapshot.
  * Move EC2 instance in another region:
    * 1a) Create image (AMI) from the EC2 instance.
      * _Info_: it is advised to reboot instance to avoid inconsistency during copy
    * 1b) Create image (AMI) from a (volume) snapshot (you must choose some machine information like architecture x86 and can choose to add additional volumes).
    * 2) Copy this image in another region to use it in the target region.
    * 3) Create an EC2 instance from the image.
  * _Note_: snapshots and images are usable in all AZ of the regions where they belong.

# EC2 automating backups
* `aws configure` //require role on EC2 instance which allow AWS CLI. Provide: region.
* `sudo yum install phyton-pip`
* `sudo pip install boto3` //Boto3: AWS SDK for python
* `vim backups.py` //use boto3 SDK to loop over volumes (`ec2.volume().all()`) and create snapshot (`ec2.create_snapshot(vol_id, vol_description)`)
* `chmod u+x backups.py`
* `./backups.py` //create snapshot for all volumes even if not attached to an instance
* `vim backups.py` //update script. Loop over running instances (`ec2.instances.filter(instance-state-name: running)`) and then over attached volumes (`ec2.volumes.filter(attachement.instance-id: instanceId)`)
* `./backups.py`
* `sudo pip install pytz`
* `vim backups.py` //update script. Loop over snapshot (`ec2.volumes[x].snapshots.all()`) and remove then when too old (`snapshot.delete()`)
* `./backups.py`

# Encryption and downtime
* For most AWS resources: encryption can only activate at creation. Examples:
  * EFS: to encrypt an existing EFS: create a new one encrypted and move data in it.
  * DynamoDb (NoSQL database)
  * RDS (Relational Database): to encrypt an existing RDS: create new one encrypted and migrate your data.
  * EBS volume: active encryption only possible at creation time.
    * Solution 1: can migrate data in new encrypted volume thanks to rsync or Robocopy. EBS handle encrypting <> un-encrypting for you.
    * Solution 2: create snapshot, then create volume from snapshot and choose to encrypt the volume. According to CloudGuru: it is possible to create encrypted snapshot from un-encrypted volume !
    * _Note_: during creation of an encrypted volume, you can specify your own key (created in IAM menu) or the default one (aws/ebs).
  * _Note for migration to encrypted resource:_ downtime is required to copy these data in a consistent way.
* S3 buckets or objects: can enable encryption at any time without disrupting your applications.

# KMS & CloudHSM
* Both allow you to generate, store and manage cryptography keys used to protect your data in AWS. Both offer high security level. Both services are now HA (high availability).
* KMS (not free but cheaper than CloudHSM)
  * Managed multi-tenant service (=one application installation used by several clients) on shared hardware
  * Suitable for applications where multi-tenancy for security is not an issue. Could be an issue for banking sector.
  * Usable during free-tier period for free
  * Encrypt data stored in AWS including: EBS volumes, S3, RDS, DynamoDB, etc.
  * Only symmetric encryption (not suitable for EC2 public/private keys)
  * Combination of AWS and you manage the keys
* CloudHSM (Hardware Security Module, more expensive):
  * Dedicated HSM instance
  * No free-tier period
  * HSM is under your exclusive control within your VPC
  * Compliant with FIPS 140-2 Level 3 / EAL-4 - includes tamper-evident (=inviolable) physical security.
  * Suitable for applications which have contractual or regulatory requirements for dedicated hardware to manage cryptography keys (pay card industry, banking...)
  * Symmetric or asymmetric encryption
  * Only you manage the keys
  * Use cases:
    * Database encryption
    * Digital Right Management (DRM): restrict access to digital content by country, etc.
    * Public Key Infrastructure (PKI)
    * Authentication or authorization
    * Document signing
    * Transaction processing
    * AWS data

# AMIs
* Provide all information to launch EC2 instance (kind of template):
  * Template for root volume (OS, applications...)
  * Launch permissions (which AWS account can use this AMI). Can be also public, private or shared to specific AWS accounts
  * Specify EBS colume to attach at launch time
* Can create custom AMI (see "EBS volumes - lab" for creation steps)
  * AMIs must be registered before it can be used to launch instances (only for AWS CLI/SDK)
  * AMIs are region-bound

# Sharing AMIs
* AMIs can be private (default), share with specific AWS account, public or even sell.
* Sharing account still has control and ownership of the AMI. It also still charged for storage (generally in S3) of the AMI.
* Copy an AMI:
  * AMI can be copied if user of AMI has given read permission on his AMI which is stored in S3 or in an EBS snapshot. When AMI is copied, you are charged for storage of the AMI on your account.
  * Additional charges also exist if you copied your own AMI image in another region.
  * You cannot directly copy an shared encrypted AMI. You need access to underlying snapshot of this AMI in order to copy it and re-encrypt it with your own key. You also need the original key used to encrypt this AMI. Finally, you can register this re-encrypted snapshot as an AMI.
  * You cannot directly copy an AMI having associate `billingProduct` code (examples: Windows, RedHat, AMIs buy on AWS marketplace). You must first launch EC2 from this shared AMI and then create AMI from this new instance. Monthly fee also exist on these AMI.

# Snowball & snowball edge
* Definition: physical device used for transporting many terabytes/petabytes of data into or out of AWS. Make large scale transfer fast, easy and secure (tamper-resistant enclosure, 256-bit encryption)
* How it works (in transfer)?
  * 1) Connect the dive on your network
  * 2) Copies your data into snowball (data is encrypted)
  * 3) Ship back the device to AWS. Snowball are region specific and cannot be transported from one region to another.
* When to use it?
  * When you have a lot of data to transfer (TB or PB or more than 1 week to upload data)
  * When your connection is limited in bandwidth or cost a lot
* What is "Snowball edge"?:
  * 100TB device with onboard computer power which can be clustered to act as a single storage and computer pool.
  * Designed for local processing / edge processing and data transfer
  * S3-compatible endpoint, supports NFS (Network File System), can execute lambda function during transfer.
  * S3 and lambda come pre-configured on device (they must be defined when you ordering a snowball edge)
* Difference: snowball is only for data transfer while snowball edge can also do some simple computing (=edge computing)
* _Migration note_: AWS Management Portal for vCenter Server: this service allows to migrate your VMWare machine into AWS EC2.

# Disaster recovery strategies
* Three types of DR with on-prem environment (can be also used in AWS environment):
  * **Backup and restore**: use AWS as backup for VMs, snapshot and other data. Could take time to restore but it the cheapest.
  * **Pilot light**: having a small environment running in AWS but configured as scalable. When failover (=basculement) is required from on-prem to AWS: the AWS environment is scale out. More expensive but faster (no instantaneous because of scaling). Requirement a full automation in deployment (even for DNS).
  * **Hot standby / multi-site**: resources are ready to be used at any moment. Less downtime but the most expensive solution. Difficult to maintain because full duplication between environment.
* DR between AWS environments:
  * **Duplicate environment in other region**: can use read-replica for RDS. In Route53, you can use "Failover routing policy". Attention points:
    * AMI and key pair are by regions specific and must be copied/imported.
    * Replica lag (bandwdith limitation, etc.)

# Storage gateway
* Definition: external (on-promise) software which connect to AWS cloud storage. Allow to on-promise applications to use AWS storage in transparency.
* How it works:
  * You must install "Storage Gateway Virtual Appliance" on your data center
  * You data center is seamlessly integrate with AWS storage (example: s3)
* Type of storage gateway:
  * **File gateway** through NFS4 (Network File System) or SMB (Server Message Block - Windows)
    * File are stored as object in S3 bucket
    * Access using NFS4/SMB mount point
    * Take benefit of S3: versioning, policies, lifecycle management, replication...
    * Low cost (no need to invest in SAN...)
  * **Volume gateway** (iSCSI - Network protocol for storage). Two types of volumes:
    * **Gateway-Stored volume**: store all data locally and only backup to AWS. Low latency for the applications but require own data infrastructure. Backup are created periodically and asynchronously to S3 in form of EBS incremental snapshots.
    * **Gateway-Cached volume**: S3 is primary storage and cache frequently accessed data in on-promise. Only required local storage capacity equals to frequent accessed data. Low latency for applications when they use frequently accessed data and low cost for local infrastructure.
  * **Tape gateway** (VTL - Storage system which emulate tape designed for backup)
    * Cost effective data archiving in cloud using Glacier.
    * No need to invest in your own tape backup infrastructure.
    * Data is stored on virtual tape on S3 which are then stored in Glacier according to your own lifecyle rules.
    * Can run as a VM on-prem or on an EC2 instance.

# Introducing Athena
* Athena is a interactive query (SQL) for S3 data.
* Serverless (don't need to provision any infrastructure). Pay per TB scanned during query execution.
* Use cases:
  * Can be used to query EBS logs, access logs, cloud trail logs, etc. stored in S3
  * Generate business report on data stored on S3
  * Analyse AWS cost and usage report
  * Run queries on click-stream data

# Athena lab
* Create cloud trail:
  * Provide name, region/global
  * Select or create new S3 bucket to store logs
* In Athena:
  * `CREATE DATABASE myathenadb`
  * Select created database
  * Create table with different columns of cloud trail logs: region, request ID, event type, user identity, API version and the data to use by specifying the S3 bucket.
  * `SELECT useridentity.arn, eventname, sourceipaddress, eventtime FROM cloudtrail_logs LIMIT 100;`
