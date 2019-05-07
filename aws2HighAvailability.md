# Elasticity & scalability
* **Scalability**: built your infrastructure to meet your long term demand (months, years). Scale up: stronger hardware, Scale out: additional nodes.
  * EC2: increase instance size as required using reversed instances
  * RDS (Relational Database Service): increase instance size (example: from medium to large). Can be changed in few minutes with down time (preferable to plan during maintenance window). Other solution is to launch more read replica.
  * DynamoDB (key-value & document DB): unlimited amount of storage
* **Elasticity** (mainly application to Cloud): stretch or retract your infrastructure on demand during a short period of time (hours, days). Allow to reduce cost when load is low.
  * EC2: increase the number of EC2 instances based on auto-scaling
    * Auto-scaling: scale can be defined based on metrics (CPU utilization, network, disk I/O, SQS queue length...) or based on a schedule. Time for instances to start must be also specified (e.g.: 300 seconds).
  * RDS: not really scalable except for Aurora database thanks to Aurora Serverless
  * DynamoDB: you can define auto-scaling for read and write. Increase additional IOPS for additional spike in traffic and decrease after spike.
* Scale up or scale out ?
  * Prefer few large instances than a lot of small instances (less costly)
  * Prefer specialized instances (e.g.: compute optimized)
  * Prefer auto-scaling when load is variable. Otherwise, you will pay for a lot of instances during low period.
* Resizing instances:
  * EBS-backed instances (see aws3DeploymentAndProvisioning) need to be stopped before change instance type. Stored-backed instance required migration.
    * Downtime exist: be careful when you plan it.
* Tips: on auto-scaling configuration, you can suspend launch of instances, termination of instances, health check, etc. This can help for maintenance.

# Capacity reservation
* EC2 Reserved instances can help to increase capacity at best cost.
* Possibility to reserve RDS instances to reduce cost:
  * Term: 1 or 3 years
  * Payment/offering type: all upfront/partial upfront/no upfront.
* Possibility to reserve ElastiCache nodes to reduce cost:
  * Term: 1 or 3 years
  * Payment/offering type: all upfront/partial upfront/no upfront (new instances types) or heavy utilization (old instance types).
* To avoid "InsufficientInstanceCapacity" error (AWS doesn't have enough instances available on a particular region) when auto-scaling add more instances during peak hours: use reserved instances mechanism.

# ELB - high availability
* Two kind of elastic load balancer:
  * **External (=public-facing) load balancer**: distribute load between Web servers. Provide a public DNS hostname (e.g.: load-balancer-1456498413.us-east-1.elb.amazon.com).
  * **Internal load balancer**: distribute load between private backend servers. Provide a private DNS hostname.
    * _Note_: caller to this load balancer don't need to know the IP address of underlying backend servers.
* High availability:
  * When you create a load balancer: you must specify subnets from at least two AZ to increase the availability of your load balancer. Only instances in these subnets could be used as target for the load balancer.
  * Load balancer across regions in not possible (use Route 53).
  * Load balancer target:
    * Targets for ALB/NLB are called "target group". Target group can be created on the fly during creation of load balancers. Target group can be also created before and then it can be attached to the load balancer during his creation. Target group can define EC2 instances list, IP lists or lambda.
    * Targets for classical load balancer are EC2 instances list and can be defined at his creation.
* Sticky session:
  * Goal: allow to maintain user session to always route on same target. It requires cookies enable at client side.
  * A number of seconds/days must be specified for the session time.
  * On ALB/NLB: stickiness session can be enable on "Target groups". For CLB, the stickiness can be directly defined on load balancer it-self.
* Performance:
  * _Context_: with HTTPS/SSL which is managed by EC2 instances: the performance could be bad due to slow encryption/decryption process.
  * _Solution_: set the SSL termination to the load balancers. This way, the EC2 instances don't need anymore to manage SSL encryption/decryption.
    * 1) Create a certificate in "Certificate manager" service
    * 2) Validate the certificate for the domain (add CNAME DNS record on your domain)
    * 3) Add listener on ELB for HTTPS/443 and select the certificate

# SQS and SNS: scalability
* SQS: simple queue service
  * Use **pooling** model/mechanism: applications must pool the queue.
  * Two queue types:
    * Standard: unlimited throughput. Message not necessarily delivered in order (=best effort ordering) and can be delivered more than once !
    * FIFO: high throughput and message delivered in FIFO (order is preserved).
  * Creation important options
    * Retention message if not read/deleted from queue: from 1 minutes to 4 days
    * Default visibility timeout: from 0 second to 12 hours (default: 30 seconds). When a reader reads a message: the message is hidden for this period of time. If reader doesn't delete this message after that period of time: message reappears in queue.
    * KMS encryption is available
* SNS: simple notification service
  * Components:
    * Publisher: the client user or service which send notifications
    * Topic: the channel for routing notifications
    * Subscription: client user or service which receive the notifications (HTTP/S, SMS, Email (require email confirmation), Lambda, SQS)
  * Use **push** model/mechanism
  * Application sent the notification once and hundred of receiver can have the notification. Sender is independent of receivers.

# RDS and multi-AZ failover:
* Multi AZ keep an exact copy of your database (called secondary/standby) in a separate AZ **synchronously** and handle failure automatically for you. Multi AZ allows fault tolerance (=tolérance de panne) via failover (=basculement)
* How it works:
  * MySQL, PostgreSQL and Oracle: utilize synchronous physical replication
  * SQL Server: synchronous logical replication (use SQL Server-native mirroring technology)
* Multi AZ can be enable/disable manually. Can put down the database about 5 seconds when active the multi AZ.
* Multi AZ should be enable in production: avoid performance problems during backup/restore and read replicas creation.
* Multi AZ / replication can increase write operations time. Provisioned IOPS are recommended.
* When AWS perform a database maintenance: Amazon performs action on standby and then switch primary<>standby to perform also maintenance on the new standby (=old primary).
* Failover information:
  * Resources connect to a database with connection string. If the database in AZ is not available (cause: storm, hardware failure, maintenance, DB server type is changed): the private DNS is automatically updated to another AZ.
  * **Multi AZ is for disaster recovery / failover (=basculement) only**. Note: this is not to improve performance and not a scaling solution. For performance/scaling: use Read Replicas.
  * You can force a failover from one AZ by rebooting an instance (use: AWS Management Console or RebootDbInstance API call)
    * During failover test: the application using database can be down during about 30 seconds event when multi-AZ is on.
    * The new primary is the old standby.
  * How be aware of a failure?
    * Check RDS events in console
    * Subscribe to RDS events. Possible to define: Emails, SMS or ARN (=Amazon Resources Name ?)
* Backup information:
  * When you perform a database restore or backup: Amazon performs action on secondary/standby to avoid I/O on primary.
  * Backup/snapshot are performed by AWS (except if you disable them) during the specified backup windows and are saved according to the backup retention period (max 35 days) that you specify.
    * _Note_: if you delete the database, the automatic (not the manual) backup are also deleted.
    * _Note_: if you create manual snapshots: the limit of 35 days is not applicable.
  * Backup are stored in S3 (99.999999999% of durability).
  * Snapshots are incremental (but first one can be removed safely)
  * Transactional engine (default engine of Mysql is tx: InnoDB) which offer commit, rollback, etc. is recommended for durability. Indded, it is better that a memory engine where data can be lost in case of crash.
  * You can restore a DB instance to a specific point in time (inside the retention period) by creating a new DB instance. RDS uploads transaction logs for DB instances to Amazon S3 every 5 minutes. Due to these transaction logs: it is possible to restore DB when you want even if there is not snapshot at this exact time.
    * During restore: you can activate multi-AZ, change instance type, change DB engine (if closely related to previous one), etc.
    * During point in time restore: you should rename original database into "whatEverYouWant" and the restored DB with the original name. Therefore, the endpoint name (DNS) will be the same as the original one and applications will be able to use the restored DB without any modification.

# RDS & Using read replicas
* Read replicas: read only copies of your database.
* Read replicas creation: in AWS Management Console or through CreateDBInstanceReadReplica API. Creation doesn't cause down time.
* Read replicas are copied from source DB by using engines native **asynchronous** replication to update the read replicas.
  * Exception for Aurora: use virtualized SSD designed for database load. Replicas nodes use same storage (=cluster volume) and create 6 copies in parallel (see next chapter).
* Read replicas only for:
  * MySQL (up to 5 replicas)
  * MariaDB (up to 5 replicas)
  * PostgreSQL (up to 5 replicas)
  * Aurora (up to 15 replicas)
* When use read replicas:
  * When read heavy database workloads
  * When DB instance is unavailable (example: I/O suspension for backups or scheduled maintenance), you can redirect read-only traffic to read replicas.
  * For reporting / data warehousing (example: Amazon Redshift) scenarios
* Creating read replicas:
  * When creating read replicas: Amazon takes a snapshot of the database
    * If Multi-AZ is not enable: snapshot is taken from primary database and cause I/O suspension for around 1 minute
    * If Multi-AZ is enable: snapshot is taken from secondary/standby database and there is no performance decrease on primary database
  * Each read replicas has their own DNS address
  * Read replicas can be in different region
* It is possible to promote read replicas to it own database. Read replicas link will be removed in this case. Useful when region going down (multi-AZ is not enough if the region going down).
* Read replicas can be itself Multi-AZ.
* You can have a read replicas of a read replicas (beware of latency)
* Snapshots and automated backups cannot be taken from read replicas.
* Metrics to look is "REPLICA LAG" (time for read replicas to be up to date)

# RDS & Using Read Replicas - Lab
* Create RDS database:
  * **Attention**: replica can stand for Multi-AZ which is different from Read Replicas
  * Can choose between: DB instance without public IP and only available on VPC or with public IP
  * Must choose a VPC security group or create a new one
  * Can manage user credentials though IAM users and roles or not
  * Can enhance the monitoring (see different process or thread using CPU)
  * Select period for maintenance (upgrade database version, activate Multi-AZ...)
* Provide CloudWatch metrics: CPU utilization, read/write IOPS, DB connections...
* **Backup must be turn on to be able to create read replica**
* Create read replicas:
  * Can choose region and AZ (or no preference for AZ)
  * Security group should be checked because it could be wrong after read replica creation
  * _Note:_: main database stay available during read replica creation

# What version of RDS am I running - Lab
* Use AWS Console to see engine version
* Use command line (require IAM setup): `aws rds describe-db-instances --region eu-west-1`

# Network performance analysis
* Network performance depends of instance type. You can use `iperf3` to measure the performance:
  * On server: `sudo iperf3 -s`
  * On client: ` sudo iperf3 -c 10.992.232`
* Network performance depends of physical distance (AZ, Region, Continent)
* See "Direct connect gateway" in "aws6Networking" for more information.

# ElastiCache
* Definition: web service with in-memory cache. Mainly used to cache most frequent database queries (avoid I/O on disk) or intense computation. Allow significant performance improvement in latency and throughput.
* Redis supports multi-AZ and not Memcached. Redis also support Master/Slave replication.
* ElastiCache is a good choice if your database is particularly read-heavy and not prone to frequent changing
* Backup information:
  * Only Redis has backup.
  * Backup are created during a backup window specified and retain for retention specified.
  * Snapshot can degrade performance if there is no replicas.
  * Backup are stored in S3 (99.999999999% of durability).

# Redshift
* Definition: data warehouse easily scalable.
* Redshift is a good answer if the reason your database is feeling stress is because management keep running OLAP (Online Analytical Processing) transaction on it.
* Can use it to query large amount of data
* Backup information:
  * Can be manual or automatic
  * Restoring a snapshot create a new cluster and import the data
  * Snapshots are incremental (but first one can be removed safely)

# Aurora 101
* Aurora is MySQL/PostgreSQL compatible. Relational database engine which combines speed and availability (high availability - HA).
* Characteristics:
  * Up to 5/3 times better performance than MySql/PostgreSQL
  * 1/10 of cost price for similar perf / availability
* Scaling:
  * Start with 10 GB, scale in 10Gb increments to 64 TB (storage autoscaling)
  * Scale up to 64vCPU / 488 GiB of memory
  * Aurora have 2 copies of your data is each availability zone (don't explained why), with minimum of 3 AZ. Total: 6 copies of your data.
  * Aurora still works in write access after maximum 2 copies lost and in read access after maximum 3 copies lost.
  * Aurora storage is also self-healing (data blocks are scanned to detect errors and are automatically fixed)
* Aurora cluster architecture:
  * One primary Aurora for read/write in one AZ
  * Read replicas up to 15. Can have several read replicas in a AZ.
  * Primary Aurora write in the 6 copies in parallel
  * 6 copies are contained in a "cluster volume" across AZ
* Two type of Aurora read replicas:
  * Aurora Replicas
  * MySQL Read Replicas
* 100% of CPU used by Aurora?
  * If caused by write: scale up (increase instance size (example: from medium to large))
  * If caused by read: scale out (add read replicas)
* Aurora capacity type:
  * Provisioned: you provision and manage the instances sizes
  * Serverless: on-demand, auto-scaling configuration for MySQL Aurora which will automatically start, shutdown and scale up/down capacity based on application needs. You pay per second and usage of standard or serverless can be activated in AWS Console.

# Aurora Lab
* Aurora creation:
  * Can chose multi-AZ or not
  * Backtrack option: can rollback database (max 72 hours) without using backup. It adds additional cost.
  * Encryption is enable by default. Once encryption is turned on: all replicas will be encrypted too.
* Read replicas seems to have a latency (around 25 ms)
* Create replica for failover (not read replicas but Multi-AZ):
  * Failover are defined by Tiers.
  * Can choose a failover priority to replace the primary one in case of failure. Priority "tier-0" is the greatest priority compare to "tier-1"..."tier-15".
* Create cross region Aurora read replica:
  * Must choose a region (not AZ)
  * Creating a new cross region replica will also create a new Aurora cluster in the target region. If the replication is disrupted, you will have to set up again. It is recommended that you select "Multi AZ deployment" to ensure HA for the target cluster.

# Auto scaling groups - lab
* Step 1: create launch configuration
  * Choose AMI + instance type
  * Create/use security group (SSH...)
  * Create/use key pair
* Step 2: Create auto scaling group based on launch configuration
  * Choose subnets (=AZ) where to start the instances
  * Choose number of instances to start
  * Choose health check grace period (time for instance to be ready). Default: 300 seconds
  * Configure scale policy
    * None (manual scale)
    * Scale based on min and max value of instances. Instances created based on CPU utilization, network IN or OUT or load balancer request
  * Configure notifications when instances launched, terminated, fail to launch or fail to terminate.

# EC2 troubleshooting auto-scaling
* Errors related to auto-scaling for EC2:
  * Attempting to use the wrong subnet
  * AZ is not longer available or supported by AWS (very weird)
  * When create a "Auto scaling launch configuration":
    * You must specify rules for security group. Example: rule which accept SSH/22 on instances inside the auto-scaling group. If rules are wrongly defined: it could be a cause of a problem.
    * You must specify key pair for instances. If associate key pair does not exist, it could be a problem.
    * Instance type configured is not supported in the AZ where instance must be launched
  * CloudWatch alarm which trigger the auto-scaling are not configured as expected
  * Auto-scaling service is not enabled on your account (caused by AWS Organization or AMI)
  * Invalid EBS device mapping: caused by custom script which attach volume at instance start-up
  * Attempting to attach an EBS block device to an instance-store AMI
  * AMI issue (e.g. AMI specified via CloudFormation or DevOps tools is not valid in the specified region)
  * Placement group attempt to use m1.large (wrong instance type). Placement group: determines how instances are placed on underlying hardware in an AZ.
  * AWS doesn't have sufficient instance capacity in the AZ that you request.
  * In properties of auto scaling group, it is possible to suspended some actions like: launch and termination. If activated: it could be the cause of a problem.
