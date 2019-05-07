# CloudWatch introduction
* Monitor AWS resources & applications. Can be also used on promise: install CloudWatch agent.
* **Metrics** feature:
  * EC2 default metrics (others metrics are custom metrics and require to install CloudWatch agent):
    * CPU utilization
    * Disk(Read/Write)Bytes: value different of zero only on stored-backed instances (see aws3DeploymentAndProvisioning)
    * Disk(Read/Write)Ops: value different of zero only on stored-backed instances (see aws3DeploymentAndProvisioning)
    * Network IN/OUT
    * Network packet IN/OUT
    * Status check
  * For burstable instances (T2, not T2 unlimited), they have 2 additional metrics: CPU credit usage and CPU credit balance.
  * Most of metrics are activated by default
  * Min granularity for custom metrics: 1 min
    * Tips: on EC2, the default monitoring is 5 min (=standard, free) and to have the min granularity of 1min (=detailed, not free), it must be turned on.
    * Tips: for metrics every 2/3/4 minutes: use detailed granularity
* **Logs** feature:
  * Logs are coming from EC2, On-promise servers, Route53 and CloudTrail API. The two first require to install an agent named "CloudWatch Logs agent".
  * You can retrieve log data for EC2/ELB instance after its termination
  * By default logs data are stored indefinitely. You can change retention in each log group.
  * Logs can be encrypted with KMS (see "aws4StorageAndDataManagement.md" for more details).
  * Scenario:
    * 1) Get logs from CloudTrail
    * 2) Filter logs (only logs from DescribeTargetHealth API call)
    * 3) Create graph from filtered logs
    * 4) Add custom alarm when have more then 15 call/logs to "DescribeTargetHealth" in period of 10 minutes
* **Alarm** feature:
  * Alarm states are: OK, ALARM (threshold reached), INSUFFICIENT (missing data)
  * Possibility to define a period of time before alarm is triggered
  * Alarm target:
    * Notification: SNS (SQS, Lambda, SMS...) or email list
    * Auto-scaling: trigger a scale up/down
    * EC2: stop, reboot, terminate, recover
* **Events** feature:
  * Similar to alarm but not based on threshold but on event.
  * Examples:
    * 1) Event source: trigger, rule: someone who share one EC2 snapshot, target: SNS notification.
    * 2) Event source: schedule, rule: every 1 day, target: take an EBS snapshot.
* SNS info: https://aws.amazon.com/sns/faqs/

# EC2 essential
* Instance type (e.g.: **t2**.micro)
  * Instance type can be specialized for general (t2, m4, m5), compute (c4, c5), memory (r4, r1, x1e), GPU computing (p2, p3, g3, f1) or storage (h1: 16TB, d2: 64TB, t3: high IOPS)
* Instance size (e.g.: t2.**micro**)
  * Define number of virtualized CPU, memory, network, clock speed.
  * When need "Jumbo frame" (network packet) > 1500 MTU/octets: use better instance size.

# Monitoring EC2
* Custom metrics (memory usage, disk usage):
  * Install CloudWatch agent (AWS agent zip + unzip + config.sh + start)
    * Once agent is installed/configured/started: the available metrics appears in AWS console and can be used.
  * Old alternative is to use AWS Perl script on start-up of EC2 instance
    * Use these scripts + cron to create custom memory metrics
* EC2 status:
  * **System status check**: check the AWS system. You don't have control over it but you can change the hardware by stop/start an "EBS-backed instance" or terminate a "Instance store-backed instance". See "aws3DeploymentAndProvisioning.md" for more details about backed instances.
  * **Instance status check**: check that your OS accept network traffic, check also application (no more details). Solution in case of problem: review configuration.
  * Alarm can be configured on statuses

# EBS essential
* Up to 16 TiB
* EBS belong to only one instance
* RAID is supported if OS support it (could improve performance with RAID 0 & RAID 10)
* EBS is in one AZ and instance must be in same AZ
* Snapshot are incremental and are saved in S3 (charge for volume total size).
  * Snapshots create an exact copy (encryption included)
  * If first snapshot is suppressed, it is still possible to restore second snapshot (AWS does magic with incremental in this case)
* Each Amazon EBS volume is automatically replicated within its AZ to protect you from component failure

# Monitoring EBS
* IOPS (Input/output operations per second):
  * SSD: good for frequent read/write of small I/O size. IOPS are measured with 256 KiB I/O size:
    * One 1024 KiB => 4 IOPS
    * Four 64 KiB sequential => 1 IOPS
    * Four 64 KiB random => 4 IOPS
  * HDD: good for large and sequantial operations (streaming...). IOPS are measured with 1024 KiB I/O size:
    * One 1024 KiB => 1 IOPS (SSD is 4 IOPS !)
    * Eight 128 KiB sequential => 1 IOPS
    * Eight 128 KiB random => 8 IOPS
* Volume type:
  * **General purpose SSD** (gp2): can be a boot volume.
    * Min 1 GiB, max 16 TiB
    * 3 IOPS/GiB with min of 100 IOPS and max of 16 000 IOPS
    * Between 1 GiB (100 IOPS) and 1000 GiB (3000 IOPS): you can temporary burst until 3000 IOPS with credits earn. Credits are earned automatically in time. You start with 5.4 million of credit (=3000 IOPS during 30 minute).
    * (Max throughput I/O: 160 MiB/s)
  * **Provisioned IOPS SSD** (io1): designed for big data database like MongoDB, Cassandra, Oracle to ensure high IO, can be a boot volume.
    * Min 1 GiB, max 16 TiB
    * To use when need > 16 000 IOPS and/or > 160 MiB/s
    * IOPS must be chosen by user between 100 and 64 000 IOPS
      * Tips: for more IOPS: use RAID 0 or 10
    * IOPS must be maximum 50 times higher than the volume size in GiB.
    * (Advice from AWS: if 4000 IOPS are required, the volume size should be less than 1000 GiB)
    * (Max throughput I/O: 500 MiB/s)
  * **Throughput Optimized HDD** (st1): ideal for frequently accessed and fast throughput at low price, cannot be a boot volume
    * Min 500 GiB, max 16 TiB
    * Throughput: maximum 500 MB/s. Bigger volumes offer better throughput (40 MB/s per TiB). Volume below 12.7 TiB can beneficiate of a temporary burst thanks to credits. Maximum burst depends of volume size and is limited to 500 MB/s.
  * **Cold HDD** (sc1): low price, large volume, ideal for infrequently access, cannot be a boot volume
    * Min 500 GiB, max 16 TiB
    * Throughput: maximum 250 MB/s in burst. Bigger volumes offer better throughput (12 MB/s per TiB). All volumes can beneficiate of a temporary burst thanks to credits. Maximum burst depends of volume size and is limited to 250 MB/s.
  * **Magnetic** (deprecated): ideal for small volume at low cost
    * Min 1 GiB, max 1 TiB
    * IOPS: not applicable
* Pre-warming EBS Volume
  * For max performance: read first all block volume to avoid pre-warming (A.K.A: initialization). Pre-warming is necessary after snapshot restore.
  * Utilities which can be used: `dd` and `fio`
  * Pre-warm commands:
      * 1) Restore volume from snapshot
      * 2) Connect to this instance via SSH
      * 3) Execute following command to see new disk: `lsblk` (example: /dev/xvdf)
      * 4) Execute following command to pre-warm new disk: `sudo dd if=/dev/xvdf of=/dev/null bs=1M`
      * 5) Alternative to 4: `sudo fio --filename=/dev/xvdf --rw=read --bs=128k --iodepth=32 --ioengine=libaio --direct=1 --name=volume-initialize`
* Metrics
  * Data available for free every 5 minutes (basic). Provisioned IOPS SSD (io1) is set to 1 minutes automatically (detailed).
  * List:
    * **BurstBalance**: credits reported in CloudWatch. Only for gp2, st1 and sc1. If value is 100%: that means that disk is not used a lot (you have the max of credits).
    * **Volume(Read/Write)Ops**: total number of I/O operations in period of time. To get IOPS: I/O operations in period / seconds in that period.
    * **Volume(Read/Write)Bytes**
    * **VolumeTotal(Read/Write)Time**: total time spent for read or write
    * **VolumeQueueLength**: number of read and write operations requests waiting to be completed in period of time
    * **VolumeIdleTime**: if volume is unused: take a snapshot and stop volume to reduce cost
    * **VolumeThroughputPercentage** / **VolumeConsumedReadWriteOps**: only for provisioned IOPS SSD. Allow to know if volume can be decreased or increased.
  * Volume statuses check (updated every 5 minutes):
    * OK
    * WARNING: performance degraded or severely degraded
    * IMPAIRED: stalled or not available. Solution: create new volume from a snapshot
    * INSUFFICIENT_DATA: data missing to provide a status
* Modifying EBS volumes: can change size, volume type, adjust IOPS performance (io1 only)

# Monitoring EFS
* EFS: Elastic File System. File system which can be used by several EC2 instances or on-premise instances.
* HA (span over AZ) and scalable
* Throughput for parallel workload. Use cases: big data, analytics, media processing
* OS Windows not supported
* Performance mode:
  * General purpose: for most of use cases
  * Max I/O: when used by a lot of instances (>100). Scale throughput and IOPS but with more latency.
* Throughput mode:
  * Bursting (recommended by default):
    * Minimum of 100 MiB/s for any file system size
    * More than 1 TiB => bursting 100 MiB/s per TiB of data stored
    * Credit: earn credit at 50 Mib/s par TiB of data stored
  * Provisioned (recommended when "BurstCreditBalance" metrics is near to zero):
    * Can provision the throughput independent of the amount of data stored.
* Metrics:
  * **PermittedThroughput** (in Mb/s): should be stable (e.g.: 105 Mb/s) except if we have too much EC2 instances on it.
  * **BurstCreditBalance**
* Use security group of EC2 for EFS
* KMS Encryption is possible

# Monitoring ELB
* Three types of ELB: Application Load Balancer (ALB), Network Load Balancer (NLB), Classic load balancer (deprecated). More details in "aws3DeploymentAndProvisioning".
* ELB have by default metrics for itself and also for back-end instances
* Monitoring type:
  * **CloudWatch metrics**: for performance (nb requests/s, etc.). Several metrics are created by default. Can create alarm when a metrics reach a limit / range
    * Errors metrics:
      * BackendConnectionErrors: number of unsuccessful connections to backend instances
      * HealthlyHostCount: number of healthy instances registered
      * UnHealthlyHostCount: number of unhealthy instances
      * HTTPCode_Backend_2XX,3XX,4XX,5XX (not for NLB): number of request returning 2XX,3XX,4XX,5XX
    * Performance metrics:
      * ActiveConnectionCount
      * Latency: number of seconds taken for instances to respond
      * RequestCount: number of request completed / connection made in the interval (1 or 5 min)
      * SurgeQueueLength (classic load balancer only): number of pending request. If queue size > 1024: request is refused.
      * SpilloverCount (classic load balancer only): number of request refused
  * **CloudTrail log**: for monitor API call (user created, S3 created, provision ELB/EC2, provision DynamoDB table). Log can be stored on S3.
  * **Access log** (optional, not for NLB): log request access with IP client, time, path, concerned resources, etc. Store file on S3. Access logs are not deleted when EC2 instances are deleted (= manually or by auto scaling)
  * **VPC flow logs** (NLB only): see "aws6Networking" for more details.
  * **Request tracing** (ALB only): add trace ID into the HTTP request header

# Monitoring RDS
* Different instance classes/size:
  * Standard (db.m1, db.m3, db.m4)
  * Memory optimized (db.m2)
  * Burstable performance (db.t2).
* Different storage types:
  * General purpose (SSD): 3 IOPS per GiB, burstable until 3000 IOPS
  * Provisioned IOPS SDD: 1000 to 40 000 IOPS.
  * Magnetic
* Monitoring type:
  * **CloudWatch metrics**:
    * SwapUsage: if increase: no enough RAM => change instance classes/size
    * ReadIOPS / WriteIOPS: to determine if storage type must be updated
    * ReadLatency / WriteLatency: higher latency, then more IOPS required
    * ReadThroughput / WriteThroughput: average bytes per second. If too high, then change instance classes/size.
  * **RDS event**:
    * A record of events: restore snapshot, security group update, parameter group update
  * **Enhanced monitoring**:
    * When create a database, you can choose enhanced monitoring to see how thread/processes use the CPU (real time metrics for OS of the DB instance).

# Monitoring Elasticache
* Two servers types: Memcached & Redis
* Node types:
  * Families (t2, m2, m4, r3, r4) impact the CPU
  * Sizes (large, medium...) impact memory and network
* Monitoring type in CloudWatch metrics:
  * CPU utilization:
    * Memcached is multi-thread and new node (aka read replicas) or better node families is required when >= 90% of CPU is used
    * Redis is not multi-thread. New/better node is required when >=90% of 1 core CPU is used. If CPU has more than 1 core, the percentage must be divided by number of core.
  * Swap Usage:
    * Typical swap size = size of RAM
    * Memcached should not exceed 50Mb of Swap usage (best is 0Mb). If exceed: increase parameter `memcached_connections_overhead` which increase memory for connections but reduce memory for items (=> more eviction). Alternative: use a better node size.
    * Redis has no swap usage metrics. Use `reserved-memory` metrics.
  * Evictions (occur when new item is added and there is no free space):
    * Memcached: no recommended setting. Solutions: scale up (increase memory = node size), scale out (add more nodes)
    * Redis: no recommended setting. Solution: scale out (add more nodes / read replicas)
  * CurrConnections:
    * Memcached/Redis: no recommended setting. Solutions: set alarm on number of connections. Cause: large traffic or app. which not close connection.

# CloudWatch custom dashboard
* Can create dashboard in CloudWatch. A dashboard is composed of widgets and a widget can be a graph/text of one or several metrics (metrics are superposed on one widget for comparison).
* A dashboard is visible in all regions but metrics selection is limited to current selected regions when you create a new widget.

# Create a billing alarm
* In billing preference, you can tick "Received billing alarm"
* The "Manage Billing Alerts" page allows to configure billing alerts when threshold on cost is reach (global for all regions). Email address must be specified and verified by AWS.

# AWS organizations
* Central manage policies across multiple accounts: create groups of accounts and apply policies (allow EC2 creation, ...) on groups. Account can be a new fresh account or it is possible to invite existing account (link sent by email). Groups are also named "Organizational units".
* Control access: can create Service Control Policies (SCPs) to allow/deny AWS services. SCPs override IAM.
  * *__Notes:__*
    * If a user or role has an IAM permission policy that grants access to an action that is also allowed by the applicable SCPs, the user or role can perform that action.
    * If a user or role has an IAM permission policy that grants access to an action that is either not allowed or explicitly denied by the applicable SCPs, the user or role can't perform that action.
* Automate AWS account creation via API. Account can be attached to a group and policies of this group will be automatically applied.
* Consolidate the billing: can create consolidated billing with unique payment method for all the accounts. Can also have aggregate benefits as volume discounts on EC2/S3 or reserved instances.

# Billing Management Console
* Definitions of payer account: account composed of "linked accounts" which consolidate billing for all these accounts.
* Cost and usage report:
  * Allow to produce reports which contain all resources, start and end date, usage value (to compare against others resources)
  * Reports can be produced daily or hourly
  * Reports are stocked on S3 (S3 policy must allow GetBucketAcl, GetBucketPolicy and PutObject)
* Budgets:
  * Billing prediction
  * Create a budget:
    * Types: cost budget (alert and view based on cost), usage budget (alert and view based on usage), reservation budget
    * Can select resources by tags, instance types...
    * Can create an alarm based on % of budgeted amount or based on an amount. Email or SNS can be used.
* CloudWatch
  * Possibility to create alarm based on billing metrics (must be activated first in billing preference).
  * Metrics are TotalEstimatedCharge or EstimatedCharge per services or by linked accounts.
  * Metrics currency can be updated in account settings
* Cost explorer:
  * Graphics tools for cost from 12 month in past to 12 months in future.
  * Can filter on instance type, linked account, service, region, AZ, tag, usage type (e.g. CloudWatch metrics, DNS queries, Data IN/OUT, EC2 transfer between region/AZ...).

# Tagging & resources groups
* Tags:
  * Key/value pair attached to AWS resources
  * Tags are metadata
  * Can be sometimes inherited (e.g. Elastic Beantalk can create other resources, auto-scaling, CloudFormation)
  * Tag can be added individually one by one on each resources. Tags can be also added via tag editor to all, for example, EC2 instances.
* Resources groups
  * Goal: regroup all AWS resources having one or more tags with specified name/value.
  * Possibility to execute common actions on the resources of a resources group (e.g.: create EC2 images instances)
  * Two types of resource groups:
    * Classic resource groups (deprecated ?): can be global for all regions or per region
    * AWS Systems Manager: per region only.

# EC2 Pricing / cost optimization
* **On demand**: pay fixed rate per hour/seconds
  * Low cost and flexibility without up-front payment
  * Application with unpredictable workload
  * For test application the first time
* **Reserved**: discount on hourly charge for an instance
  * Usage: app with stable state or predictable usage or app that require reserved capacity
  * User can make upfront payment (reduce cost as much as possible), partial upfront or not upfront
  * If you have reserved too many instances: can be sell them on marketplace. Advantage for buyer: term could be shorter.
  * If reserved instances arrive at term: you will be switched back to on demand model. Solution: plan new reservation to save money.
  * Type of reserved instances:
    * Standard Reserve Instances (RI's): discounts up to 75%. 1 or 3 years term. Flexibility to change availability zone, instance size, networking type.
    * Convertible RI's: discount up to 54%. 1 or 3 years term. More flexible as standard: possibility to change instance families, operating system...
    * Scheduled RI's: for predictable events in time windows
* **Spot**: Use unused computation resources from AWS Cloud. Possibility to define a bid/maximum/spot price. Prices are determined by demand and supply.
  * Spot instance can have interruption (instance termination) if EC2 needs the capacity back. New feature: some instances (M5, C5) can be only paused and therefore volume data are still there.
  * Use cases:
    * Applications having flexible start/end times.
    * Very low prices
    * Needs large amount of additional capacity
* **Dedicated hosts**: physical EC2 servers dedicated
  * Useful for regulatory / licensing which not allow multi-tenant virtualization. Can reduce costs for application licenses.
  * Can be purchased on demand or by reservation (discount up to 70%)
* Cost optimization tips:
  * Low utilization tips: can setup CloudWatch alarm if CPU is under 5% during 50 minutes
  * Check balance between HA and services unavailable.
  * Unused load balancer: remove them to avoid charges
  * EBS volume are charged even if not used and should be (snapshot and) removed. IOPS cost more than others EBS. Downsize volumes that aren't near to full capacity.
  * RDS are charged even if not used and should be (snapshot and) removed. Can use CloudWatch to check if "connection over time" is to 0.
  * EIP (Elastic IP = IP which can be attached to an EC2, etc.). These EIP cost money even when not used.
  * Use "Trusted advisor" service: provide advise to reduce cost, improve performances, security, fault tolerance...

# CloudTrail
* Logs API call (Console, CLI, SDK)
* Can provide information for many type of audit: governance, compliance, operational, risk
* Functionality named "Trail" allows to centralize logs in one S3 bucket (by region or for all regions). This logs can be analyzed by your preferred tool or by CloudWatch logs near in real-time and alert you in case of problem.

# AWS Config / AWS Config Lab
* Goal:
  * Continuous monitoring: provide inventory of AWS resources and their configuration history.
  * Continuous audit and compliance: check AWS resources are compliant with configured rules. Possibility to use SNS (Simple Notification Service) to have notification in case of changes.
  * Operational troubleshooting: allow to find cause of issues because of a change in configuration.
  * Change management: review resource dependencies prior to making changes
  * Resource tracking: what resources is used where
  * Security analysis: find role of an user when a problem occur
* UI configuration:
  * Dashboard: display all resources and graphs with resources compliant/not compliant with rules
  * Rules: configure rules which can be predefined rule or custom rule (via lambda). Rule can be triggered at configuration change or periodically. Rules can be applied on certain type of resources.
    * Example of predefined rules: check password of IAM user match the security requirement, check EBS volume are encrypted, check if S3 buckets have replication over regions, check AMI used are in predefined range, check instance type are in predefined range, EBS are attached to an instance, RDS as multi-AZ, MFA on root is activated, S3 versioning is enable...
  * Resources: display/search resources and allow to see configuration history (point in time).
  * Settings: define what you monitor. You can choose to stream the configuration change to have an alert at each changes. You must define the S3 bucket to use.
* How it works: when change is performed on resources, AWS Config receives a message and save it in S3 bucket. Moreover, SN (Simple Notification) can be triggered.
* AWS Config is per region
* AWS Config requires an IAM role with: read only permission on resources, write access to S3 logging bucket, publish access to SNS (Simple Notification Service). This role can be created automatically by the AWS console.
* AWS Config is attended to be managed by administrator. Other users can have read only access to AWS Config. Indeed, if user turn off AWS Config: it could be a security problem.
* Tips:
  * Use CloudTrail with AWS Config to provide deeper insight into resources.
  * Use CloudTrail to monitor access to AWS Config such as someone stopping the Config Recorder.
  * Useful link: https://aws.amazon.com/config/faq/

# Health dashboard
* Two types of dashboard:
  * **Service Health Dashboard**: give health of AWS services. It's global dashboard for everyone to see if AWS is running correctly. Possibility to go back in time to see health of services.
  * **Personal Health Dashboard**: notifications/logs from Amazon when issues occur in their services and which have an impact on your AWS account.

# Services type - quick overview:
  * **CloudWatch**: Metrics for performance + alarm + logs. Example: see CPU utilization for EC2 instance.
  * **CloudTrail**: Record events/API call. Example: allow to know who provision/delete the security group 3 weeks ago.
  * **Config**:
    * Provide configuration history of resources. Example: allow to see what was the security group rules 3 weeks ago.
    * Check resources compliance thanks to rules which define desired configuration settings
    * Resource tracking: what resources is used where
  * **Trusted advisor**: advisor for cost (low usage of EC2...), performance (provisioned IOPS not attached to EBS-optimized instance...), security, fault tolerant, service limits
  * **Inspector**: analyzing the behavior of you AWS resources and identifying potential security issues
  * **Systems manager**: regroup resources to execute common tasks on them
  * **Organization**: handle AWS accounts for an organization
