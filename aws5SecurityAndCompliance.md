# Compliance frameworks on AWS
* **ISO** 27001: certification which can be get by an organization. Organization must define a documents describing security and apply it. AWS has this certification.
  * Details: specifies the requirements for establishing, implementing, operating, monitoring, reviewing, maintaining, and improving a documented Information Security Management System within the context of the organization's overall business risks.
* **FedRAMP** (Federal Risk and Authorization Management Program - USA): government program to define standard security for assessment, authorization, monitoring in the cloud.
* **HIPPA** (Health Insurance Portability and Accountability Act of 1996 - US): role is to keep confidentiality for health insurance, security on health information and control on health cost. If you want to store health insurance data on AWS, the service must be compliant HIPPA.
  * Details: check AWS website to see if a service is HIPPA compliant
* **NIST** (National Institute of Standard and Techno - US): provide set of industry standard and best practice to help organization manage cybersecurity risks.
* **PCI DSS** (Payment Card Industry Data Security Standard): set of policies for security of debit, credit and cash cards.
  * AWS is PCI compliant but that not mean that applications you develop in cloud are also PCI compliant.
* **Others** frameworks:
  * SAS70: Statement on Auditing Standard
  * SOC1: Service Organization Control: for accounting standard
  * FISMA: Federal Information Security Modernization Act
  * FIPS-140-2: security standard for cryptography. From level 1 to 4. CloudHSM is level 3.
* _Notes_:
  * in AWS console: the "Artifact" menu allow to see all documents related to compliance.
  * If your application data are not compliant with some AWS certifications: it isn't the responsibility of AWS.

# DDoS
* Recommended link (mainly for security certification): https://d1.awsstatic.com/whitepapers/Security/DDoS_White_Paper.pdf
* DDoS can be achieved by multiple mechanisms: location/pay of botnet, packet flood, reflection, amplification...
  * **Reflection**: Attacker sent requests to servers with a spoofed IP and servers reply to the spoofed/victim IP.
  * **Amplification**: Same as reflection but response is larger than the original request (generally: 28 to 54 times larger for an NTP (Network Time Protocol) server).
  * **Application attacks** (layer 7):
    * Sent a lot of GET request to server to attack.
    * Slowloris: sent partial request / partial header requests in order to keep a maximum of connection open.
* How to mitigate the attack ?
  * Minimize the attack surface: firewall, WAF (kind of firewall for WebApp - layer 7)...
  * Be ready to scale to absorb the attack (auto-scaling group...)
  * Protect/safeguard exposed resources
  * Learn normal behavior and create plan against attack (can use CloudWatch)
* AWS shield:
  * **Free**: protect all AWS customers on ELB, CloudFront, and Route 53. Turned on by default.
    * Protect against SYN/UDP floods, reflections attacks and others layers 3/4 attacks.
  * **Advanced**: provide more advanced protection against larger and most sophisticate attack on ELB, CloudFront and Route 53
      * $3000/month
      * Always on, network and application monitoring to have near real-time notification of DDoS
      * DDoS Reponse team 24x7 (help you to manage & mitigate DDoS attack)
      * Protect your bill on ELB, CloudFront and Route 53 in case of DDoS
* Technologies you can use to mitigate DDoS: CloudFront (shield protection), Route53 (shield protection), ELB  (shield protection), WAFs, Autoscaling, CloudWatch.

# AWS Marketplace - security products
* Provide a bunch of products (OS, software, CloudGuru...): mainly virtual machines already setup with particular software. Marketplace is region specifics.
* Can be free, hourly/monthly/yearly prices, license.
* For security, you can purchase security products from a third party vendors: penetration testing (Kali Linux), monitoring, firewall, WAF, anti-virus, hardened OS, CIS hardened OS (OS certified by CIS recommendations)
  * Attention: to execute penetration testing (also called port scanning), a request must be completed and sent to AWS event if products come from AWS Marketplace.

# IAM policies
* IAM: Identity and Access Management
* Json document which contain permissions
* An explicit "deny" always overrides an explicit "allow".
* Example of Json:
  ```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": "*"
        }
    ]
  }
  ```

* Two types of policies:
  * **Pre-built policies**: template policies defined by AWS (prefixed by box icon in AWS console). Examples: administrator access (full access to all AWS resources), read only access (e.g. S3 read only: "s3:Get*" & "s3:List*" actions), power user access (admin access but not possible to update users/groups)
  * **Custom policies**: _see next chapter_

# IAM custom policies - lab
* Step 1: create a custom policies via visual editor (or Json)
  * Select service: S3
  * Select access level: list (ListBucket...), read (GetObjectVersionAcl...), write
    * _Note_: ListBucket, GetObjectVersionAcl are the API method name.
  * Select concerned resources: all resources/S3, a bucket or an object
* Step 2: create role
  * Select "EC2 - allows EC2 instances to call AWS services on your behalf"
  * Add permissions (=policies created in step 1)
* Step 3: create buckets
  * Create two buckets in two different regions

# IAM users and groups
* User:
  * Root account:
    * Root user has FULL administrative right
    * Don't use root user for daily work and administration
    * Use MFA for root user
    * Root user shouldn't have access key. If have: delete them.
  * User account:
    * New user has implicit deny for all AWS services
    * Can apply policies directly to user or to an user through a group.
    * MFA can and should be activated
  * Never store/pass your access credential (access key ID & secret access key) in an EC2 => use SSH forwarding.
  * When you create an user, you can choose to have (both are possible):
    * Access key ID & secret access key for AWS CLI/SDK/API
    * Password for AWS Console
* Group:
  * Definition: a group of users where policies can be attached.
  * Best practices: assign policies to group instead of user. Define group by function (developers, architects, DB admins)
  * Group can have managed policies (AWS policies or custom) or inline policies (policy usable only for the group)
  * User can belong to several groups

# IAM role
* Role is a temporary security credentials (use AWS managed - Secure Token Service - STS) for an entity. Entity can be a resource (EC2) or an user outside our AWS account who need temporary access.
* Role contains a list of IAM policies
* AWS service can have only one role attached.
* You should never/pass access credentials to an EC2 instance => use role.

# Roles & custom policies - lab
* Step 1: create EC2 instance:
  * Use US-EAST-1 region
  * Don't attach any role (default)
  * Ensure port 22 (SSH) is open in selected security group or open it
  * In terminal:
    - `chmod 400 MyKP.pem`
    - `ssh ec2-user@IP -i MyKP.pem`
    - `sudo su`
    - `aws s3 ls` (missing credential error)
* Step 2: create user:
  * Choose programmatic access (allow to get access key ID and secret access key for AWS CLI, SDK, API).
  * Add user to a group. Group must have policies which allow read in S3.
* Step 3: on created EC2 in step 1:
  * Select instance and attach role created of previous chapter (=can attach role at any time and role effect is immediate)
  * In terminal:
    - `aws s3 ls` (no error and buckets of both regions are listed)
    - `aws s3 cp /home/xxx/myfile.txt s3://mycustombucket-useast1` (access denied: no write permission)
* Step 4: update policies to allow write on S3 (take effect immediately)
* Step 5: on created EC2 in step 1:
  * In terminal:
    - `aws s3 cp /home/xxx/myfile.txt s3://mycustombucket-useast1` (no error)
    - `aws s3 ls s3://mycustombucket-useast1` (file listed)
    - `aws s3 cp /home/xxx/myfile.txt s3://mycustombucket-euwest1` (no error with different region)
    - `aws s3 cp /home/xxx/myfile.txt s3://mycustombucket-euwest1 --region eu-west-1` (sometimes, the region must be specified !)

# MFA & reporting with IAM - lab
* In IAM: activate MFA for root account:
  * Choose virtual device. Others options: hardware device / U2F (universal second factor: USB...) device
  * Scan QR code with smartphone (Android app: google authenticator)
  * Provide two codes (6 digits) provided by QR scan to AWS.
* In AIM: activate MFA for user account
  * Create an user
  * Add user in group with policy "AdministratorAccess"
  * Click on "Assign MFA device" and scan QR code as for root account
* On EC2 instance of previous chapter:
  * Detach assigned role (no more role for EC2)
  * In terminal:
    - `aws s3 ls` (unable to locate credentials)
    - `aws configure` (provide user access key id & user secret access key & default region)
    - `aws s3 ls` (success)
    - `aws iam create-virtual-mfa-device --virtual-mfa-device-name EC2-User --outfile /home/ec2-user/QRCode.png --bootstrap-method QRCodePNG` (create QRCode.png file on disk)
    - `aws s3 cp ./QRCode.png s3://mycustombucket-euwest1` (QRCode.png can be sonsulted with browser and scanned with smartphone)
    - `aws iam enable-mfa-device --user-name EC2-User --serial-number arn:aws:iam::"USER_NUMBER_HERE":mfa/EC2-User --authentication-code-1 "CODE_1_HERE" --authentication-code-2 "CODE2HERE"` (MFA device is assigned to the user account)
  * If you use MFA with AWS CLI:
    * You must create a temporary session: `aws sts get-session-token --serial-number ARN_OF_THE_MFA_DEVICE --token-code CODEHERE`. This command provide temporary tokens with expiration. These tokens must be exported thanks to `export` Linux command. Finally, you can work normally.
    _Note_: STS = Security Token Service.
    * If you use `aws configure` with permanent tokens: MFA will never required.
* On AWS console, you can download a "Credential Report": can see lot of information about users (creation date, last password usage, is MFA is activated or not...)

# Security token service (STS)
* Grants users (could be external users to AWS, no IAM required) limited and temporary access to AWS resources. Users can comes from 3 sources:
  * Federation (typically Active Directory: used by company when you log-in in a Windows computer)
    * Use SAML (Security Assertion Markup Language)
    * Grant temporary access based to users of Activate Directory credentials
    * Single sign on allows users to log in AWS console without assigning IAM credentials
    * Provide token valid between 15min and 36h
  * Federation with Mobile Apps
    * Use OpenId provider (Facebook/Amazon/Google...) credentials
  * Cross account access (delegation)
    * Let's users from one AWS account access resources in another
    * Provide token valid between 15min to 1h (default: 1h)
* Term definition:
  * Federation: combining/join users list from one domain (IAM) to another (Activate Directory)
  * Identity broker: service which allow you to take identity from point A and join (federate it) to point B.
  * Identity providers/store: service like Active Directory, Facebook, Google...
  * Identities: a user of a service like Facebook...
* Scenario: you host a WebSite on EC2. This WebSite allow users of a company XYZ to log-in into based on the Active Directory of the company XYZ. Once connected to WebSite, the user can access to his own S3 bucket. Solution:
  * Step 1: an employee enters his username and password
  * Step 2: the WebApp call the "Identity Broker" (identity broker should be developed by your own)
  * Step 3: Identity Broker validates the credential (or not) against the Active Directory (alternative: Facebook, Google)
  * Step 4: Identity Broker calls GetFederationToken() function using an IAM credentials. The call must include an IAM policy, a duration (15 min to 36h) and a policy that specifies permission to be granted to the temporary token.
  * Step 5: Security Token Service confirms that policy of the IAM user making the method have permissions to create new tokens. This STS also return to the Identity Broker: access token, secret access key, a token and a duration (token lifetime).
  * Step 6: Identity Token returns the temporary credentials to the WebApp.
  * Step 7: the WebApp use the temporary credential to make request to S3.
  * Step 8: S3 uses IAM to verify that temporary credential are allowed to perform the request.
* IAM policy "IAM:PassRole" allows to pass role to another AWS account and/or pass role to an AWS service to assign temporary permissions to this service
* Temporary access is global (not by region) as like IAM
* The endpoint to request the token is in N.Virgina: https://sts.amazonaws.com/. Additional point can exist in others regions.
* With Amazon Cognito user pool, you can create and maintain a user directory, and add sign-up and sign-in to your mobile app or web application

# Security & Logging
* Main logging services in term of security:
  * AWS CloudTrail
  * AWS Config
  * AWS CloudWatch logs
  * VPC Flow Logs (network traffic across your VPC)
* Prevent un-authorize access to your logs. Several ways to proceed:
  * Use IAM user, group, role, policy
  * S3 bucket policies
  * MFA (MFA will be requested to delete files)
* Add alerts when log files are created or fail to be created:
  * CloudTrail notification. CloudTrail SNS notifications only point to log file location (goal: don't divulge details)
  * AWS Config rules
* Prevent modifications to logs:
  * IAM and S3 controls and policies
  * CloudTrail log file validation: allow to validate the log files where CloudTrail delivered them via a hash
  * Use CloudTrail log encryption

# AWS Hypervisors, isolation of AWS resources and AWS firewalls
* Hypervisors (also called Virtual Machine Monitor) is a software, firmware or hardware which create and run virtual machine. A computer which run several virtual machines is a host machine. Virtual machine is the guest machine.
* EC2 use Xen Hypervisor. Two types of virtualization:
  * Hardware Virtual Machine (HVM): full virtualization. Guest machines are not aware that they share processing time with others VMs. Most of the instances use HVM.
  * Para-virtualization (PV): lighter form of virtualization. It offers better performance. However, it is not recommended anymore by AWS as performance between HVM and PV are now almost the same. Only available for Linux instance.
    * Detail: CPU has four privilege modes (0 (most) to 3 (least)). Host machine use mode 0, the guest machine use mode 1 and application use mode 4.
* Isolation (not in exam tips): offer complete isolation between customers (e.g.: me). Levels from 1 to 6:
  * 1: Physical interface (network card)
  * 2: Firewall. In charge to redirect the traffic to the correct customer of the next step.
  * 3: Customer1 Security Group, Customer2 Security Group, etc.
  * 4: Virtual interface
  * 5: Hypervisor
  * 6: Customer1, Customer2, etc.
* Access:
  * Hypervisor access for AWS employees:
    * AWS administrators can access to administrative host if they need to access to management plan. Administrative hosts are hardened to protect the management plan of the cloud. Administrators must use MFA to access to administrative host and once access is not needed: it can be revoked (=cancel). All access are audited.
  * Guest access (EC2):
    * Controlled by the customer (e.g.: me)
    * You have full root access. AWS has no access right to EC2 machine / guest OS.
* Scrubbing (=clean) between customers:
  * EBS blocks are reset each time a free block is allocated to a new customer
  * Disk memory and RAM are also reset to 0 by the hypervisor before assigned to a new customer.

# EC2 dedicated instances vs dedicated hosts
* When create EC2 instance, you can choose "Tenancy":
  * **Shared hardware instance**: default
  * **Dedicated instances**: run on VPC on dedicated hardware (isolated at hardware level). Dedicated instances can share hardware from same AWS account that are not dedicated instances.
    * Price: on demand, reserved instances (save up to 70%), spot instances (up to 90%).
    * Paid per instances
  * **Dedicated hosts**: also dedicated hardware but you have more control. You can re-use same hardware over time. It is useful for some software licenses, address corporate compliance and regulatory requirements.
    * Paid per hosts
    * Can have more visibility on sockets, cores, host ID, etc.

# AWS system manager: EC2 run command
* Scenario: you are an administrators of a lot of EC2 instances. You would like to automate setup of applications, configuration changes, apply patches.
* In AWS Console:
  * Create role with policies having permission "AmazonEC2RoleForSSM".
  * Create Windows EC2 instance with this role.
  * Go in System Manager and click "Run Command":
    * Select command to run. Example: "AWS-ConfigureCloudWatch"
    * Select instance (manually or through tags)
    * Choose parameter values. Example: disable or enable CloudWatch
    * Can activate SNS to get notified when command is run.
    * Can find the AWS CLI command of "Run command" setup done through AWS Console
* All available commands are defined in a System Manager Document.
* SSM agent must be installed on the EC2 instances. It is installed by default only on Windows Server, Amazon Linux and Ubuntu. Moreover, you need role allowing SSM.
* Run command can be used through AWS Console, AWS CLI, AWS SDK, AWS tools for Windows PowerShell, System Manager API.

# AWS systems manager parameter store (SSM parameter store)
* Scenario: you working at a bank and you want to pass confidential information to EC2 instance (database connection string, password, license key...). Solution: use AWS system manager parameter store.
* Parameter are accessible in EC2 Run Command, EC2 State Manager, AWS CloudFormation, lambdas...
* In AWS Console:
  * Parameter store: available in EC2 menu in left-hand menu.
  * Create parameter: provide name, value and type: string, string list or secured string (use a KMS key to encrypt value)

# S3 bucket policies
* Example (granting permissions to multiple accounts)
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"AddCannedAcl",
      "Effect":"Allow",
      "Principal": {"AWS": ["arn:aws:iam::111122223333:root", "arn:aws:iam::444455556666:root"]},
      "Action":["s3:PutObject","s3:PutObjectAcl"],
      "Resource":["arn:aws:s3:::examplebucket/*"]
    }
  ]
}
```
* **Important**: an explicit deny always override an explicit allow
* Principal can be "\*" to make bucket public

# Securing S3 using pre-signed URLs
* Can access to private S3 file via pre-signed URL via SDK or CLI.
* In AWS Console:
  * Create role with permission "AmazonS3FullAccess"
  * Create EC2 with role just created and with a security group which allow SSH/port 22.
  * In SSH of EC2:
    * `sudo su`
    * `aws s3 ls` (list all buckets)
    * `aws s3 mb s3://mybucketpresigned` (mb = make bucket)
    * `echo "hello" > hello.txt`
    * `aws s3 cp hello.txt s3://mybucketpresigned`
    * `aws s3 ls s3://mybucketpresigned` (displaye hello.txt)
    * `aws s3 presign s3://mybucketpresigned/hello.txt --expires-in 300` (generate URL with a token which allow to access to file during 5 minutes. Default expire time is 1 hour)

# AWS Config with S3
* Important rule for S3 in AWS Config:
  * s3-bucket-public-write-prohibited & s3-bucket-public-read-prohibited: if an S3 bucket policy or bucket ACL allows public write/read access, the bucket is not compliant

# AWS Inspector
* Help to improve security and compliance of deployed applications. Check if application deviate from best practices or have vulnerabilities. Report is produced by priority order and are available in console or via API.
* In AWS Console:
  * Create EC2 instance
  * Go to Inspector (under "_Security, Identity & compliance_" category)
  * Create role for inspector. AWS propose a default role: ec2:DescribeInstances. Use this default role.
  * Add tags to concerned instances to inspect.
  * Install inspector agent (info: cannot be done on RDS because you don't manage the OS). Several ways to proceed. Example: execute a `wget` on EC2 instance or use system manager run command as proposed by AWS doc.
  * Create "**Assessment target**": provide a name and filters EC2 based on tags.
  * Create "**Assessment template**": provide a name, duration (default: 1 hour) and select rules packages among:
    * Security best practices
    * Runtime behavior analysis
    * Common vulnerabilities and exposures (=> select this one: provide a report with all CVE/CPE vulnerabilities found)
    * CIS OS Security Configuration Benchmarks (check if your OS is hardened according to rules defined by CIS: Center for Internet Security)
  * **Assessment Run**: select the template and click on "Run" button
  * After 1 hour: you can view the number of finding and download the report
  * /
  * Create a master template including all four rules packages. Run the template. Duration: 24 hours
  * After 24 hours: you can view the number of finding and download the report
  * _Note_: these security analyzes can be fully automated by API
* Rules packages can monitor network, file system, and process activity
* Different level of severity for rules/finding: high, medium, low, informational

# Trusted advisor
* Available under "_Management Tools_" category
* Help to **reduce cost**, improve **performance**, improve **fault tolerance** (=ability for a system to still working after a fail), improve **security** by optimizing your AWS environment and see the **service limits**.
* Don't require to setup agent on EC2 instances
* Give some advises according to your current environment. Example for security: warn on unrestricted access (0.0.0.0/0) for some security groups, MFA required for root user...
* Two levels of services: basic (free) and full trusted advisor (Business and enterprise companies only. Mainly to have advises on cost optimization and fault tolerance).
* Difference between trusted advisor and inspector:
  * Trusted advisor: optimize performance and security
  * Inspector: analyze application security

# Shared responsibility model (**important for exam**)
* AWS manage the security of the cloud, while security in the cloud is the responsibility of the customer (e.g.: me). Customer have control of what security they choose.
* AWS responsibilities:
  * Physical server and below (physical infrastructure)
  * Physics environment and physical security + protection (fire, power, climate)
  * Storage device decommission (=not use anymore) according to industry standard
  * Hardware, software they install depending of service (e.g.: OS for RDS), networking and facilities
  * Some managed services (see after)
  * API access use SSL for secure connection
  * Network device security and ACLs (only backside, not cloud side)
  * DDoS protection
  * EC2 instances cannot send spoofed data
  * Personal access to facilities
  * EC2 instance hypervisor isolation
  * _From AWS_: compute, storage, database, networking, regions, AZ (=what you don't have access)
* Customers responsibilities:
  * EC2 update, security group configuration
  * IAM
  * MFA
  * Password/key rotation
  * Access advisor (on IAM user, you can see his access and restraint if some are not used)
  * Trusted advisor (cost optimization, security, performance, fault tolerant, service limits)
  * Security groups: you are responsible to block port 22, etc.
  * Access control list (NACL, Bucket policies...)
  * VPC
  * _From AWS_: encryption, https, firewall configuration, application, access management, customer data
* The responsibility model changes for different service types:
  * **Infrastructures services**: the computes services like EC2, EBS, Auto Scaling and VPC. You control the OS, you configure and operate identity management system which provide access to users. You are responsible for AMIs, OS, applications, data in transit, data at rest, data stores, credentials, policies and configuration.
  * **Container services:**: RDS, DynamoDB, EMR (Elastic Map Reduce - Hadoop...) and Elastic beanstalk. You generally don't manage the OS or the platform layer (see important note). AWS is responsible to patch the OS, the MySQL/Oracle servers, PHP updates with Elastic Beanstalk, etc. AWS provides a managed service for these application "containers". You are responsible for firewall rules, identity and access management (IAM).
    * _Important notes_:
      * Elastic beanstalk: you can manage OS update or activate an option to automate the update
      * EMR: you must manage the OS for instances running for a long time. For short term instance (common case): the OS is up to date at each start.
      * OpsWork: OS is up to date at start. If OS stay up for a long time: you must manage the OS update.
      * ECS: there is two modes. EC2 launch type: you manage the update/patch. AWS Fargate: Amazon manage all for you.
  * **Abstracted**: High-level of storage, database and messaging service: S3, Glacier, DynamoDB, SQS (Simple Queue Service), SES (Simple Email Service). Service accessible through endpoint (AWS APIs). AWS manage the underlying service/application as well as the OS. However, your are, for example, responsible for S3 buckets policies.

# Other security aspects (**important for exam**)
* Security groups:
  * Security group are stateful !
  * When define security group open ports: they are open for ingress traffics (opposite to egress). Example: to access to DB, you must open port 3306 in your security group in front of you RDS.
  * Be able to read security group:
    * SGXXXXXX 22 0.0.0.0/0 (port 22 open for all instances)
    * SGXXXXXX 22 10.0.124/32 (port 22 open for a particular instance)
  * Important ports: 22 (SSH), 80, 1433 (SQL server), 3306 (MySQL)
* If you want to find who is provisioning EC2 instances in a big company: CloudTrail will insert logs in S3 (make sure CloudTrail is on) and you can read these logs. You can also use Athena to query S3 to easy the process.
* AWS artifact:
  * AWS artifacts provide on demand downloads of AWS security and compliance documents: AWS ISO, AWS Payment Card Industry (PCI), Service Organization Control (SOC). You can send these documents (A.K.A.: audit artifacts) to your auditors to show security and compliance of AWS infrastructure/services that you use.
