# Deploy an EC2 instance lab
* AMI: Amazon Machine Images. Some AMI are only OS but others propose pre-installed software.
* Configure instance options:
  * Can decide how many instance you want.
  * Can choose Spot instance:
    * Current prices are displayed by region and maximum price can be provided.
    * One shot instance or persistent request can be selected. One shot mean that instance will be available only one time. Persistent mean that instance will be available several times (could depend of from/to dates if defined)
    * Interruption behavior can be chose: stop, terminate or hibernate.
    * Launch group option: if defined, all instances requested must be launched in same time or not at all.
  * VPC: Virtual Private Cloud: kind of network for the AWS user. VPC can be chose/created during EC2 instance creation.
  * Placement group: allow to decide if all instances will be in same Availability Zone (AZ). Offer better network bandwidth (about 10Gb) between instances but if AZ is unvailable: all instances are inaccessible.
  * IAM role (Identify Access Management): role which allow EC2 to access to others resources (S3, database...)
  * Monitoring (CloudWatch) is free and activated by default (5 min). Detailed monitoring (1 min) can be activated with additional charges.
  * Possibility to choose dedicated hosts.
  * User data: field to define a shell script to run at start-up:
  ```
  #!/bin/bash
  yum update -u
  yum install httpd -y
  service httpd start
  chkconfig httpd on
  ```
  * Add storage option:
    * On "EBS-backed instances" (most common)
      * Root volume: hard link to the EC2 instance and generally destroy at termination
      * Elastic Block Store (EBS) volume: soft linked to instance. Can be detached and attached to another instance. Can be keep alive after instance is terminated.
    * On "Instance store-backed instances" (not very common)
      * Cannot be stopped/started, only terminated
      * Cannot add additional volume
      * The default volume is deleted at termination
  * Add tags option
    * Allow to define key/value pair (user define tag). Example: environment=dev.
  * Configure security group options:
    * Control in/out traffic on the EC2 instance
    * Default: ssh on port 22
  * Review and generate/use SSH key pair

# EC2 launch issues
* InstanceLimitExceeded error: you have reached max instance you can launch in a region. Default is 20 but can ask Amazon to increase this number.
  * Example: `aws ec2 run-instances --images-id ami-XYZ --count 5 --instance.type t2.micro`
* InsufficientInstanceCapacity error: Amazon doesn't have enough capacity/hardware to execute the request.
  * Solution: wait few minutes, request less instances, select different instance types, purchasing request instances, submit new request without specifying AZ.

# EBS volumes & IOPS
* IOPS: indicate the performance for SSD volumes
* General purpose (gp2) - SSD: boot volume (but not only)
  * min 100 IOPS, 3 IOPS/Gb, max of 16 000 IOPS
* Provisioned IOPS (io1) - SSD: for I/O intensive, NoSQL/relational database, latency sensitive workloads.  
  * 50 IOPS/Gb, max of 64 000 IOPS
* If you reach the max of IOPS of the volume for gp2: I/O requests will be queuing and application could become slow. Solutions:
  * Increase size of gp2. If you already have 5.2TB: you already have 16 000 IOPS and increase size will not help
  * If you need more than 16 000 IOPS: change storage to io1.

# What is a Bastion host
* Description:
  * Host located in your public subnet (network in availability zone with a gateway to internet)
  * Allows you to connect to your EC2 instances using SSH/RDP (Remote Desktop Protocol) only.
  * You can log-in to the Bastion host over internet from your local machine
  * You then use the Bastion host to initiate an SSH/RDP session over the private subnet to your EC2 instances in the private subnet.
* Allow you to safely administer EC2 instances without exposing them on internet.
* Does not enable outgoing requests (example: no internet access for EC2 instances)
* Best practice: lock down Bastion host as much as possible and limit access to user. Make sure bastion host is up to date.
* Can use EIP (Elastic IP) to have always same IP for Bastion host. Can help for whitelist this IP. When associate EIP to Bastion host: the current public IP is replaced by EIP.
* Can put Bastion host in ASG (auto-scaling group) with min/max of 1. If bastion is dead: ASG will create a new one.
* Connection to Bastion host then to instances in private subnet:
  * `ssh-agent bash`
  * `ssh-add key1.pem`: private key on local machine used by Bastion and private instances. Therefore, private key is not stored on Bastion host.
  * `ssh -A ec2-user@{IP_BASTION}`: option A allow to forward agent connection
  * `ssh ec2-user@{IP_PRIVATE_INSTANCE}`: work well
  * Now, if you terminate Bastion host which is inside an ASG: new Bastion host will be created. The EIP must be associate manually to this new Bastion host.
  * `ssh -A ec2-user@{IP_BASTION}`: will fail because of `~/.ssh/known_hosts` contains ID of old Bastion host. Solution: remove it.

# Elastic load balancers 101
* Elastic load balancer types:
  * Version 2 - Application load balancer (layer 7):
    * **Load balancer**: receive HTTP/HTTPS requests
    * **Listener**: read requests from client and use rules to redirect them to target group. Routing rules can be based on path (/dev, /prod) or on host specified in HTTP header to redirect to a different target group.
    * **Target group**: received requests. Health check are configured by target group. Can choose to routes to instances (including auto scaling group), to IP addresses (including on-premise) or to lambda function. Work also for micro-services (allows dynamic port mapping).
      * _Note_: to target an auto scaling group: the "target group" must be selected during creation/edition of the auto scaling group.
  * Version 2 - Network load balancer (layer 4): used for very high performance (million of connections) but less flexible.
    * **Load balancer**: receive client requests
    * **Listener**: read requests from client and use rules to redirect them to target group
    * **Target group**: use TCP protocol and port to route request to instances (including auto scaling group) or IP addresses (including on-premise). Health check are configured by target group and are simplified compare to health check for ALB.
      * _Note_: to target an auto scaling group: the "target group" must be selected during creation/edition of the auto scaling group.
    * More expensive that application load balancer.
  * Classic load balancer (_almost deprecated_)
    * Can operate at layer 7 or 4. At layer 7, it is limited to X-Forwarded header or sticky session. Not possible to load balance based on URL like on ALB.
    * Cross-zone load balancer (activated by default on ALB/NLB): if "cross-zone load balancer" is enable: allow to evenly distribute traffic to all registered instances. Recommended to keep roughly the same number of request in each zone/AZ.
* Pre-warming the load balancer (for application/classic load balancer only)
  * To avoid overload of ELB (Black Friday), you can contact AWS to pre-warm your ELB (only when you expect sudden traffics increase). AWS needs to know following infos:
    * Start/end dates
    * Request rate per second
    * Total size of a typical request
* IP addresses:
  * Application load balancer scale automatically to adapt to your workload (through auto scaling group). Side effect: IP can change during auto-scale.
  * Network load balancer create a NLB node in each associate subnet of the balancer. A static IP address (or Elastic IP address) is attributed to these nodes. Firewall rules are kept simple and client can use the static IP.
    * _Note_: depending of how target are define, the source adresses conservation is different:
      * Target is "Instance ID": source addresses of the clients are preserved
      * Target is "IP address": source addresses of the clients are the private IP addresses of the NLB node.
* Both balancers types (ALB & NLB) can be chosen to get benefits from both (static IP & intelligent routing). Place an ALB behind a NLB.

# ELB error messages
* Application/classic load balancers errors:
  * HTTP 4xx: client side errors
    * HTTP 400: malformed request (example: header is malformed)
    * HTTP 401: unauthorized - user access denied
    * HTTP 403: forbidden - block by the WAF (Web App. Firewall)
    * HTTP 460: client closed connection before load balancer answer (client timeout too short ?)
    * HTTP 463: HTTP header X-Forwarded-For with >30 IP addresses
  * HTTP 5xx: server side errors (application, load balancer, db, ... problems)
    * HTTP 500: internal server error
    * HTTP 502: bad gateway. Example: application server close the connection OR response is malformed
    * HTTP 503: service unavailable. No registered target on load balancer (example: REST WS application). Or returned if it does not have enough resources to handle large spikes in traffic
    * HTTP 504: gateway timeout: Application not responding. Requests sent to the backend instances are timing out.
    * HTTP 561: unauthorized. Applicable when the load balancer use an identity provider

# Deploying an application load balancer lab
* Create an application load balancer:
  * Must specify at least two availability zones for current region.
  * Tips: select security group of your EC2 instance when create the load balancer to have access to the instance.
  * You must specify the "target group" for the load balancer. You decide if you will provide a list of instance IDs or a list of IP addresses. You also decide the protocol to the instance (example: HTTP / 80)
  * Then, you select the target (instance)
  * A path for health check on instances must be specified (example: /index.html)
* Once ALB is created: a DNS is available to access to your instance though the load balancer
* Metrics are displayed in CloudWatch and are available every 5 minutes.

# Lambda
* Lambda is serverless and auto scalable
* You pay for only for compute time you consume (by ms)
* By default, it is: fault-tolerant, scalable, elastic and cost efficient. Know the differences:
  * Fault-tolerant: remain in operation even if some of the components used to build the system fail (Hystrix). Very costly due to hardware replication.
  * HA: provide high availability on a system. Can have some down time.
  * Scalable: can add more stronger hardware (scale up) or more hardware (scale out)
  * Elastic: scale according to current load using 'on demand' resources
* Languages supported: Java, Node.JS, Go, C#, Python
* Lambda can be outside a VPC. For example: when it is a lambda accessing to S3.
* Lambda has metrics for CloudWatch (number of invocation, execution duration...).
* Lambda can have triggers (new object in S3, specific path called though ELB...) or can be called by an application. Lambda has role with policy which define access allowed to lambda (access to S3 objects, access to CloudWatch logs to write logs...).

# ECS: Elastic Container Service
* Goal: container management service that supports Docker. Manage container on EC2 cluster.
* Existing container registry: ECR (AWS EC2 Container Registry), 3rd party registry (e.g. Docker Hub), Self hosted registry.
* ECS has a task definition. It is a JSON file (or UI) which contains the blueprint (=plan) of your application. JSON file contains the Docker image, the registry of the image, ports to open for instances and data volumes.
* Definitions:
  * **Service**: define the number of desired task definition to instantiate and keep running in your cluster
  * **Cluster**: define the number of EC2 instances and instance type/size to use.
  * **Fargate**: new service which manage the cluster (=instances) for you.

# Lightsail and batch
* Lightsail:
  * A VPC (Virtual Private Server) service. Allow to create a server with pre-configured (WordPress, Node.js, LAMP, GitLab).
  * Single instance
  * You can connect to the server/instance via SSH
  * The instance is not a EC2 instances but something else. Lightsail instance doesn't live in a VPC/subnet: it is independent.
* Batch:
  * You submit your jobs (shell script, container...) to AWS. Jobs are first stored to a AWS Batch queue. Then, the service launch appropriate size of computing for the jobs.
  * Job definitions allow you to specifies how the job run.
  * This is a managed service (configure and manage of environment are automatic).

# RDS deployment and provisioning
* RDS security group:
  * Rules which define which EC2 instances can access to database.
  * Source can be IP range or a security group (=all EC2 using this security group can access to the database)
* When create a database, a subnet group must be specified.
* RDS subnet group:
  * Defines where DB instances are placed in term of AZ
  * Should contain at least two subnets in different AZ
  * Upon failover, the standby is promoted and another standby is created in a different AZ
* _Note_: subnet group can be also used by ElastiCache

# EFS deployment and provisioning
* Create EFS lab:
  * Create EFS over two chosen AZ: 'a' and 'b'
  * Create EC2 in AZ with subnet 'a' with security group allow SSH and HTTP/80
    * `ssh -i "key1.pem" ec2-user@ec2-35-172-109-87.compute-1.amazonaws.com`
    * `sudo yum install httpd -y`
    * `sudo service httpd start`
    * `sudo mount -t nfs4 fs3db57a75.efs.us-east-1.amazonaws.com:/ /var/www/html`
    * `df -T` # allow to see EFS is mount in /var/www/html/
  * Create EC2 in AZ with subnet 'b' with security group allow SSH and HTTP/80
    * `ssh -i "key1.pem" ec2-user@ec2-35-172-109-228.compute-1.amazonaws.com`
    * `sudo yum install httpd -y`
    * `sudo service httpd start`
    * `sudo mount -t nfs4 fs3db57a75.efs.us-east-1.amazonaws.com:/ /var/www/html`
    * `cd /var/www/html`
    * `sudo touch index.html`
    * `sudo vim index.html` # Hello world
  * Go to http://35.172.109.87/ # display 'Hello world'

# DynamoDB deployment and provisioning (**not very important for exam**)
* Key-value store (no schema). Similar to MongoDB.
* **Fully managed service: provisioning, auto-scaling, fault-tolerant, high available. Not attached to a VPC, it's a public database available using endpoint.**
* Synchronize all data between AZ of same region
* You can specify the desired throughput
* Design to easily move data to hadoop cluster in Elastic Map Reduce
* Encryption of data of DynamoDB
* Popular use cases:
  * IoT. Store: meta data
  * Gaming. Store: session information, leaderboard
  * Mobile. Store: user profile, personalization
* Create DynamoDB:
  * Specify a table name (name for DB)
  * Specify a primary key (e.g: name). Primary key is part of the partition/hash key. This partition key is used to spread data over hosts for scalability and availability.

# AWS system manager
* Definition: System manager (SSM) give visibility and control over your AWS infrastructure. Can organize your environment/inventory to regroup resources. Can also manage on-promises system (resources not in AWS).
* Integrated with CloudWatch to view dashboard, operational data, detect problems
* A "run command" feature is available to automate process through resources
  * Pre-defined commands for EC2 instances:
    * Stop, restart, terminate, re-size
    * Attach / dettach volume
    * Create snapshot, backup DynamoDB DB tables
    * Apply patches and updates
    * Execute Ansible/shell script
* AWS Console demo:
  * Step 1: create role which can be used by EC2 with policy "AmazonEC2RoleforSSM"
  * Step 2: create EC2 instances with this role and some tags
  * Step 3: in System Manager: select instances by tags/names/etc. System managers provide lot of information on selected resources:
    * AWS Config information
    * CloudTrail information
    * Trusted advisor
    * CloudWatch dashboards
    * Inventory: top 5 OS version, top 5 services, top 5 roles, all instances list...
  * Step 4: in System Manager: automation allow to execute pre-defined commands (=also named document name) or custom ones.
    * Possibility to define SNS notifications based on command result.
    * Possibility to write command result in S3 bucket.
    * Possibility to schedule command (cron)
    * State manager allows to have instances which remain in a consistent state by applying a pre-defined configuration (network settings, configuration settings...). Useful when someone apply manually a configuration on one of EC2 instance => state manager will override it every x hours.
  * Step 5: in System Manager: shared resources
    * Can see all instances status, OS, etc.
    * Activation: allow to manage on-promise system. SSM agent must be installed on your machine to communicate with AWS.
    * Parameter store: allow to store password for database, connection string for database etc. These parameters are encrypted.
