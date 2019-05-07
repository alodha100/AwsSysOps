# VPC overview (**important for exam**)
* VPC: virtual private network or virtual data center in the cloud. VPC is a virtual and isolated network where you can launch AWS resources. You control: selection of IP range, creation of subnet, configuration of route tables and network gateways.
* In each subnet, you can have security group and network access control list (NACL). You can create private subnet (=no internet access) for your backend service like database and public subnet for WebService API.
* VPC are linked to a specific region and subnet in an AZ inside this region.
* You can create VPN between AWS and your own data center
* VPC Structure:
  * VPC is inside a region and you can have several VPC by region. Good practice: have several VPC to isolate test from prod, etc.
  * Two possibles entry access to a VPC:
    * **Internet gateway** (=internet, IGWs)
    * **Virtual private gateway** (=VPN, see "AWS VPN" chapter)
  * Entry access goes to a router and then to a route table chose by the router
  * After route table, the traffic goes into a Network ACL
  * Network ACL hit the private/public subnet. A network ACL can be linked to several subnets but a subnet can have only one Network ACL.
  * Subnet contain one or several security groups
    * _Note_: Subnet are linked to one Availability Zone only
    * Example of IP range authorized by AWS for subnet:
      * 10.0.0.0 - 10.255.255.255 (10/8. Number of address: 2^(32-8))
      * 172.16.0.0 - 172.31.255.255 (172.16/12. Number of address: 2^(32-12))
      * 192.168.0.0 - 192.168.255.255 (192.168/16. Number of address: 2^(32-16))
  * Security groups contain a list of EC2 instances with similar function
* What can we do with VPC?:
  * Launch instances in specified subnet
  * Assign custom IP range to subnet
  * Configure route tables between subnets (can decide which subnet can communicate with others subnets)
  * Create Internet gateway and attach it to our VPC.
    * _Important point_: only one Internet gateway by VPC. Internet gateways are HA over several AZ.
  * Much security control over AWS resources (subnet can block some IP addresses via network ACL)
  * Network ACLs. **Network ACL are stateless** (open port 80: you must specify IN or/and OUT). Can have Allow or Deny rules.
  * Security group. **Security group are stateful** (open port 80 means open traffic for IN and OUT). Can have only Allow rules.
* Difference between default and custom VPC
  * Default VPC: user friendly, allow to deploy instances immediately. The two default subnets are public and have a route out to the Internet (IGW).
  * If you delete the default VPC, the option "Create default VPC" allows you to recreate it.
* VPC wizard: when you create a new VPC, you can choose VPC wizard with default VPC configured for different scenarios:
  * VPC with single public subnet
  * VPC with a public and a private subnets
  * VPC with a public and a private subnets and connection to your corporate data center
  * VPC with a private subnets and connection to your corporate data center (most secure)

# VPC lab part 1
* Create VPC:
  * Provide name and IPv4 CIDR block: 10.0.0.0/16 (65531 IPs)
  * Click on "Create" VPC: automatically create default route tables, network ACLs and default Security Groups
* Create subnet:
  * Define name, select VPC, select AZ and IPv4 CIDR block: 10.0.1.0/24.
    * _Note_: 10.0.1.0/24 is equals to 2^(32-24)=256 IP addresses. AWS reserve 5 IP addresses: 251 remaining IP addresses.
    * _Note_: CIDR are specified at VPC level and also at subnet level. Subnet must have equals or less IP than VPC. Example: VPC with 10.0.0.0/16 (65531 IP) and subnet with 10.0.0.0/24 (251 IP). For VPC and subnet, the min is /16 and max is /28.
* Create subnet in another AZ: 10.0.2.0/24
* Create internet gateway (IGW):
  * Provide name and attach it to a VPC
* Check default route table created by AWS:
  * Don't allow Internet access and it's called the "main" route. If subnet is not explicitly associate to a route table: it is automatically associate to this "main" route.
* Create a route table (in addition of default one)
  * Provide name and VPC
  * Edit "routes" tab: allow Internet access for all IPv4 (0.0.0.0/0) and IPv6 (::/0) by selecting the newly created IGW.
  * Associate one subnet (10.0.1.0/24) to this new route table to allow access to Internet.
    * _Note 1_: the second subnet (10.0.2.0/24) is not associate and therefore is redirected to the "main" route (no Internet access).
    * _Note 2_: you can associate several subnet to a route table
  * Select the subnet used as public subnet in previous step (10.0.1.0/24) and active the "auto assign IPv4" action to assign public IPv4 to the public subnet.
* Create public EC2 instance:
  * Select VPC
  * Select public subnet 10.0.1.0/24
  * Auto-assign public IP: select "Use subnet setting (Enable)". Public IP will be assigned to EC2 instance.
  * Security groups don't span over VPC. Create new security group for SSH and port 80.
* Create private EC2 instance:
  * Select VPC
  * select private subnet 10.0.2.0/24
  * Auto-assign public IP: select "Use subnet setting (Disable)". No public IP will be assigned to EC2 instance.
* Info:
  * Try to connect on public EC2 instance via SSH: it works
  * Both EC2 instances cannot communicate together because they are in different subnet

# VPC lab part 2
* Create new instance security group for instance in private subnet (10.0.2.0/24):
  * Choose name
  * Select VPC
  * Allow HTTP(s) (80/443), Mysql/Aurora (3306) and SSH (22), ICMP (ping). Source for all: 10.0.1.0/24 (=public subnet)
  * Attach this new security group to the instance
* Connect in SSH to instance in public subnet:
  * Ping private instance 10.0.2.143: ping OK
  * Copy private key of instance in private subnet into instance in public subnet. Never do that in production: use bastion host !
  * Connect in SSH to instance in private subnet: success
    * Execute: `sudo su`
    * Execute: `yum update -y`: timeout because no connection to Internet.

# NAT instances & NAT gateways
* Goal of this lecture: add possibility to perform `yum update` (access to Internet) of previous course without make subnet public via IGW.
* Create **NAT instance** on AWS Console:
  * Create EC2 instance
    * Choose AMI having "VPC NAT" integrated (called: NAT instances)
    * Choose custom VPC and the public subnet (10.0.1.0/24). NAT instance must always be inside a public subnet.
    * Choose security group with HTTP, HTTPS and SSH
    * Once created, select instance and execute "Change Source/Dest. Check " action to disable the check. This check ensure that either the source or the destination is the instance itself. This is not the case for NAT instances.
  * On main/default route table:
    * Add route with destination: 0.0.0.0/0 and target: the NAT instance. It create a route rules from NAT instance to outside world.
  * Connect in SSH to instance in public subnet:
    * Execute `ssh` to private subnet. Success (as in previous course)
    * Execute `sudo su`
    * Execute `yum update -y`: now it works !
  * Terminate the NAT instance
  * Connect in SSH to instance in public subnet:
    * Execute `ssh` to private subnet. Success (as in previous course)
    * Execute `yum install mysql -y`: timeout because NAT instance is terminated
* Create **NAT gateway** on AWS console:
  * _Note_: two types: "NAT gateway" for IPv4 and "Egress Only Internet Gateway" for IPv6.
  * Create NAT gateway (under VPC menu):
    * Select the public subnet
    * Select or create an EIP
  * On main/default route table or create new route table:
    * Add route with destination: 0.0.0.0/0 and target: the NAT gateway
    * Select private subnets which need an Internet access
  * Connect in SSH to instance in public subnet:
    * Execute `ssh` to private subnet. Success (as in previous course)
    * Execute `sudo su`
    * Execute `yum install httpd -y`: now it works !
  * _Result_: now, the instance in private subnet has access to public subnet through the NAT gateways which is himself in public subnet. This allow internet connection to the instance.
* Comparison between NAT instance and NAT gateways:
  * NAT gateways can be HA through AZ and failover is automatically handled by AWS. This is not the case for NAT instances where you need to create manually scaling group + script for failover + create instances in multi-AZ + update OS.
  * Good bandwidth with NAT gateways up to 10Gbps. Amount of traffic for NAT instance depends of the instance type. Can increase instance size to improve performance of NAT instance.
  * NAT gateways don't support Bastion host => cannot connect to NAT gateway from Bastion host
  * NAT instances are behind security group. It's not the case for NAT gateways.
  * NAT gateways is more secure: no SSH to access to gateway
  * Really prefer NAT gateway over NAT instance
  * NAT gateways have automatically a public IP address assigned

# Network access control lists vs Security group
* Default network ACL created during creation of VPC allows all IN and OUT traffic.
* One subnet can be associate to only one network ACL. A network ACL can handle several subnets.
* In AWS console:
  * Create new network ACL:
    * Provide name and select the VPC
    * _Note_: default rules deny all IN and OUT traffic by default
  * On instance in public subnet:
    * Install/setup httpd and create index.html
    * Find public IP of instance and access to it: content of index.html is displayed.
  * Update the created network ACL (reminder: it is stateless):
    * Add inbound rule: order id: 100, HTTP, source: 0.0.0.0/0, Allow. Do same for HTTPS and SSH. Note that we keep default deny rules but they are applied after the new created ones.
    * Add outbound rule: order id: 100, HTTP, destination: 0.0.0.0/0 Allow. Do same for HTTPS.
    * Add outbound rule: order id: 300, TCP, port range (also called: ephemeral port): 1024-65535, destination: 0.0.0.0/0 Allow.
      * _Note_: generally, you have inbound rule for SSH (22) and to allow answering, you must open ephemeral ports (1024-65535) in outbound rules.
    * Associate the network ACL to public subnet (10.0.1.0/24). _Note_: this will automatically un-associate this subnet from the default network ACL as only one is allowed.
    * Try to access to public IP of instance: content of index.html is displayed.
  * Update the created network ACL:
    * Add inbound rule to deny HTTP with an order id 101 => nothing change as rule are evaluated in numerical order
    * Change deny inbound rule order id to 99 => now, it is impossible to access to index.html
* You can block IP address with network ACL, not with security group
* You need at least 2 public subnet to deploy an application load balancer

# EC2 bastion: ping others instances for monitoring
* Goal: from bastion host (in public subnet): ping others instances in private subnet for monitoring
* In security group of private instances:
  * Allow ICMP IPv4. Source can be CIDR (x.x.x.x/x), IP (IP of bastion host) or security group. Use security group of bastion host for this example.
* In security group of bastion instance:
  * Allow ICMP IPv4 for all instances to ping. Source: security group of instances to ping. ICMP is a two way protocol: request + ACK.
* In Network ACL of instances to ping:
  * Allow ICMP for inbound and outbound with source: 10.99.0.0/16.
* In Network ACL of bastion host:
  * Allow ICMP for inbound and outbound with source: 10.99.0.0/16

# VPC endpoints
* Current situation: the instance in private subnet can access to NAT gateway and then the NAT gateway can communicate with (for example) S3. Unfortunately, the link between API gateway and S3 goes though Internet to access S3 instead of stay an internal call. Let see how to fix that. Moreover, IAM role must be handled.
* In AWS console:
  * Create EC2 role with permission "AmazonS3FullAccess".
  * On instance in private subnet: attach the new role.
  * On Network ACL: associate both subnets (10.0.1.0/24 & 10.0.2.0/24) to the default network ACL to keep things simple and allow all IN/OUT requests.
  * Connect in SSH to instance in public subnet:
    * Execute `ssh` to private subnet. Success
    * Execute `sudo su`
    * Execute `aws s3 ls`: display list of buckets but it is going out to the Internet. If you shutdown the NAT gateway: it won't work anymore. Of course, it's the same if you remove route to NAT gateway in route tables.
  * Create an endpoint (via VPC menu). Endpoint allows to securely connect your VPC to another service.
    * Select the concerned service in the list: S3 (Gateway)
    * Select the VPC
    * Associate the endpoint with main route table. A route is automatically created with destination=S3, target=VPC endpoint (meaning: when destination is S3, the request is transferred to the VPC endpoint). Info: the private IP address of the instance will be used to access to S3.
  * Connect in SSH to instance in public subnet:
    * Execute `ssh` to private subnet. Success
    * Execute `sudo su`
    * Execute `aws s3 ls`: display list of buckets using only the private network
* Amazon MQ doesn't support a VPC endpoint connection.

# VPC flow logs
* Capture IP traffic IN and OUT from your network interfaces in your VPC. They are stored using CloudWatch logs.
* Flow logs can be created at 3 levels:
  * VPC: catch all IN and OUT traffic
  * Subnet
  * Network interface level
* In AWS console:
  * Go in "VPC" menu
    * Select the VPC
    * Click on "Create Flow Logs" action
      * Select the filter: all, accept (only logs accepted traffic), rejected (only logs rejected traffic)
      * Create flow logs IAM role used by CloudWatch
      * Specify the log group of CloudWatch (pre-requisite: create log group in CloudWatch just by providing a name)
  * In CloudWatch: you can stream the logs into AWS lambda to re-actively react to some logs. You can also export logs in S3.
* You cannot enable flow logs on VPCs which are peer with your VPC except if these VPCs are also in your account.
* You cannot tags flow logs
* You cannot change flow logs configuration after his creation (e.g. cannot associate a new role)
* Not all IP traffic are monitored:
  * Traffic generated by instances when they contact the Amazon DNS server. For other DNS server, it can be logged.
  * Traffic for Windows license are not logged
  * Traffic to and from 169.254.169.254 for instance metadata
  * DHCP traffic
  * Traffic to reserved IP address for the default VPC router (5 IP address mentioned in previous courses)

# Elastic IP (EIP) & Elastic network interface (ENI)
* EIP info:
  * Definition: public IP address which can be "moved"
  * EIP are region specific
  * IPv6 not supported
  * When associate EIP to EC2 instance, the previous public IP is released as well as the DNS (e.g. ec2-25.123.587.123.compute1.amazonaws.com)
  * EIP can associate to an EC2 instance or to an ENI (ENI is attached to an EC2 instance).
  * EIP are charged only when:
    * EIP is not associate to an EC2 instance
    * Several EIP are associate to one EC2 instance (through ENI)
* ENI info:
  * Virtual network card associate to a subnet which offer:
    * Primary and secondary private IP address
    * An EIP (IPv4 only) or public IPv4 or public IPv6
    * MAC address
  * ENI must be linked to a existing security group (generally the security group of the EC2 instance where ENi will be attached on)

# VPC peering
* Allows you to connect one VPC with another via direct network route using private IP address. Example: one EC2 in a subnet/VPC can communicate with another instance in another subnet/VPC via private IP address.
* You can also peer VPC between different AZ or regions in same AWS account or between different AWS accounts.
* If VPC A is peering with VPC B and VPC B is peering with VPC C: the VPC A cannot reach VPC C (no transitive peering)
* VPC peering can improve network performance (> bandwdith) and create a reliable connection (no single point of failure). Moreover, data don't go through Internet.
* AWS console:
  * In "VPC > Peering Connections": create a peering connection by specifying the requester VPC (OhioPeer) and accepter VPC (VirginaPeer).
  * In "VPC > Peering Connections" of accepter: the request appears in pending status. You must accept the request.
  * In "VirginaPeer" VPC, update the route table. Add: destination: 10.101.0.0/16 (IP range of VPC OhioPeer) and target: select the peer connection.
  * In "OhioPeer" VPC, update the route table. Add: destination: 10.99.0.0/16 (IP range of VPC VirginaPeer) and target: select the peer connection.
  * In N.Virgina region (where "VirginaPeer" VPC exist): update Security Group of bastion to add ping (All ICMP - IPv4). Source is: 10.101.0.0/16. Same for inbound and outbound rules of NACL.
  * In Ohio region (where "OhioPeer" VPC exist): update Security Group of bastion to add ping (All ICMP - IPv4). Source is: 10.99.0.0/16. Same for inbound and outbound rules of NACL.
  * Bastion host of Ohio can ping bastion host of N.Virgina and the opposite
  * _Note_: it is possible to transfer key between regions. On server of region 1, execute: `more ~/.ssh/authorized_keys`. On region 2: import displayed key via AWS console. On region 2: use this imported key in your bastion host. Now, bastion host can perform `ssh` to the other bastion host.
    * This solution is not very secure. If key of one region is compromise, both regions are compromise.

# Cloud Front
* Global content delivery network (CDN). Low latency and high transfer speed from the origin (=EC2 instance, S3 bucket...)
* Components:
  * **Origin**: origin of your content (=EC2 instance, S3 bucket...)
  * **Distribution**: configuration of logging, HA and limitations
  * **Edge location**: location of cached objects over the globe (total: 158 in 29 countries)
  * **Regional edge cache**: location of cached objects not frequently accessed (total: 11)
* How it work: when edge location receive a request, it check first his cache, then original edge cache (if exist) and finally, the origin.
* Change content: wait content reach expiration or invalidate the content before expiration (cost more).

# AWS VPN
* Connection between VPC and your own infrastructure. Connection throughput depends of your Internet connection.
* Provide good security thanks to VPN
* Three components:
  * **Virtual Private Gateway**: entry point in the VPC
  * **Customer Gateway**: allow to specify the IP of your on-prem environment to AWS
  * **VPN connection**: connection between "Virtual Private Gateway" and "Customer Gateway". Routing type are:
    * Dynamic: it is require Border Gateway Protocol (BGP)
    * Static
* Other possibility: use custom VPN connection if required instead of AWS VPN connection.

# Direct Connect
* If you have a VPN between on premise network and AWS VPC: performance can vary a lot according to your Internet connection. Recommended solution by AWS is to use "AWS Direct Connect":
  * You get a **dedicated network connection**
  * It is a network service that provides an alternative to using the Internet to utilize AWS cloud services
  * It does not involve the Internet
  * It can reduce costs in some situation
  * Speed:
    * **1Gbps and 10Gbps ports are available** (=between AWS services and AWS Direct Connect)
    * Between 50 Mbps to 500 Mbps (offered through an APN partner) between you and AWS Direct Connect
* Two main components:
  * **Direct Connect**: make the link between customer and AWS. Available in a lot of city around the world
  * **Virtual interface**: give access to AWS services. Public Virtual Interface give access to EC2, S3 and Private Virtual Interface give access to the VPN.
* This solution offers less security than AWS VPN. More sure to configure correctly security group and NACL.

# VPC CIDR calculations
* Computation: http://cidr.xyz/
* 10.0.1.0/28: 2^(32-28) = 16 IP addresses (min for AWS subnet) | 10.0.1.0/16: 2^(32-16) = 65536 IP addresses (max for AWS subnet)
* AWS reserve 4 IP addresses + broadcast: 5 IP addresses
* For exam, you must know how many IP addresses for:
  * /24: 256 IP addresses, 251 in AWS world
  * /25: 128 IP addresses, 123 in AWS world
  * /26: 64 IP addresses, 59 in AWS world
  * /27: 32 IP addresses, 27 in AWS world
  * /28: 16 IP addresses, 11 in AWS world

# DNS 101
* IPv4: 32 bits, 4 billion, IPv6: 128 bits
* Top level domain name in "xyz.com": it's .com. Top domain name are defined by Internet Assigned Numers Authority (IANA). Second level domain name in "xyz.co.uk": it's .co
* Domain must be unique and therefore are handled by registrar (GoDaddy.com, 123-reg-co.uk, ...). These domains are registered with interNIC (a service from ICANN) to enforce uniqueness.
* DNS operate on port 53 and famous US route is 66. Combination of both give Route53.
* Start Of Authority (SOA) record is information stored in DNS zone for that zone. It contains:
  * The name of the server which provide the data for the zone
  * The administrator of the zone
  * The current version of data file
  * The TTL in seconds of the file (example: cloudguru.com point to 254.123.12.27. It will take x seconds to have a change of IP address propagate over Internet)
* DNS steps
  * 1) User provide domain name in browser: acloud.guru (.guru is a top domain name)
  * 2) IPS (Internet Provider Service) asks to ".guru" servers if it knows the "acloud" domain. Answer returned is: `acloud.guru 172800 IN NS ns.awsdns.com` (172800 is a TTL in seconds and "ns.awsdns.com" is a NS record).
  * 3) IPS asks to "ns.awsdns.com" if it knows "acloud.guru". Answer is yes as the domain is registered to AWS. In addition of "**SAO**" and "**NS**" records, there are:
    * **A record**: mapping between domain name and IP address
    * **CNAME**: mapping between two domain name (http://m.acloud.guru & http://mobile.acloud.guru)
    * **Alias record**: same as CNAME but allow naked domain names (naked: acloud.guru, not naked: m.acloud.guru). Common use case with AWS ELB: www.example.com > elb1234.elb.amazonaws.com). Always use alias record against CNAME if you have the choice.
    * **MX record**: mail server record
* ELBs don't have pre-defined IPv4 addresses. You must resolve to them using a DNS.

# Register a domain name - lab
* Go in "Route 53" menu (not region specific)
  * Click on "Registered domain" and provide name: "helloroute53guru.com"
  * Provide all admin info: name, first name, address, country...
  * Go in hosted zone to see records: NS records point to 4 domain over the world to ensure availability. There is also a SOA defined.
* Create 2 EC2 instances with httpd server in N.Virginia region
* Create 1 EC2 instance with httpd server in Sydney
* Create 1 EC2 instance with httpd server in South America

# Simple routing policy - lab
* Simple: one record with multiple IPs. All values are sent to user in random order. No health check possible. Of course, IP can be a load balancer.
* In AWS console:
  * Create A record:
    * Name: _empty_
    * No alias
    * Value: 4 IP addresses of the EC2 instances
    * Routing policy: simple
  * _Info 1_: TTL are 60sec for A, 172800sec for NS and 900sec for SOA
  * _Info 2_: Wait about 5 minutes to propagate the update performed over the world
  * Access to your Web site: one of 4 EC instances is displayed.
    * _Note_: After refresh: it is always the same server displayed due to DNS server cache. You can use your phone to test if you can reach another IP address.

# Weighted routing policy - lab
* Weighted: you can specify that 20% of requests go to US-EAST-1 and 80% go to US-WEST-1.
  * If you specify 50% and 150%: first one will have 25% (= 50/(50+150)) and second one will have 75%.
* In AWS console:
  * Remove the A record of previous course (not possible to have 2 simple routed A record)
  * Create A record (=> repeat 4 times for the 4 EC2 instance):
    * Name: _empty_
    * No alias
    * Value: 1 IP address of the Virgina EC2 instance
    * Routing policy: weighted, Weight: 25%, ID: WebServer1 N.Virgina
    * TTL: 60sec
  * Test (no performed): about every 1:30min you should get different result when access to your Web site.

# Latency routing policy - lab
* Latency: select the best IP based on latency of the final user compare to a region.
* In AWS console:
  * Remove the 4 A records of previous course (not possible to have weighted and latency records)
  * Create A record:
    * Name: _empty_
    * No alias
    * Value: 2 IP addresses of the Virgina EC2 instances
    * Routing policy: latency, Region: us-east-1 (automatically selected based on provided IP. Can be updated), ID: WebServer1&2 N.Virgina
    * TTL: 60sec
  * Create A record (=> repeat for South America EC2 instance):
    * Name: _empty_
    * No alias
    * Value: 1 IP address of the Sydney EC2 instance
    * Routing policy: latency, Region: ap-southeast-2, ID: WebServer Sydney
    * TTL: 60sec
  * Access to your Web site: the region with best latency is selected by Route 53.
    * _Note_: with a VPN (e.g.: NordVPN), you can connected to Brasil network and check that South America server is used.

# Failover routing policy - lab
* Failover: always the primary server (=active) will be selected if health check is positive. Otherwise, secondary server (=passive) will be selected.
* In AWS console:
  * Remove the 4 A records of previous course
  * Create Health Check (in left menu of Route 53)
    * Name: MyHealthCheckUS_EAST_1
    * What to monitor: Endpoint
    * Endpoint info:
      * Specify endpoint by: IP address (Could be by domain name. Useful for elastic load balancer)
      * IP address: 1 IP address of the Virgina EC2 instance
      * Host name (No mandatory. Used by Route53 to pass the Host header. Seems to be used by load balancer): helloroute53guru.com
      * Port: 80
      * Path: /index.html
    * Can create an alarm to receive SNS notification
  * Create A record:
    * Name: _empty_
    * No alias
    * Value: 1 IP address of the Virgina EC2 instance
    * Routing policy: failover, failover record type: primary, Id: Primary
    * Health check: select one create before (MyHealthCheckUS_EAST_1)
    * TTL: 60sec. AWS recommend a low TTL (60sec) when we use Health Check.
  * Create A record:
    * Name: _empty_
    * No alias
    * Value: 1 IP address of the Sydney EC2 instance
    * Routing policy: failover, failover record type: secondary, Id: Secondary
    * TTL: 60sec
  * Access to Web site: primary one (Virgina) is displayed.
    * _Note_: clear your browser history can help to clear local DNS cache
  * Stop Virgina EC2 instance
  * Wait couple of minutes to have health check fail
  * Access to Web site: secondary one (Sydney) is displayed.

# Geolocation routing policy - lab
* Geolocation: select server based on location of your user (=DNS query originate). Scenarios for European users: redirect to server which display price in Euro or have only European languages.
* In AWS Console:
  * Restart the stopped server of previous course (attention: the IP address can be different)
  * Remove the A records of previous course
  * Create A record:
    * Name: _empty_
    * No alias
    * Value: 1 IP address of the Virgina EC2 instance
    * Routing policy: Geolocation, location: North America, Id: NorthAmerica
      * _Note_: you can choose the "Default" location.
    * TTL: 60sec
  * Create A record:
    * Name: _empty_
    * No alias
    * Value: 1 IP address of the Sydney EC2 instance
    * Routing policy: Geolocation, location: Oceania, Id: Sydney
    * TTL: 60sec

# Multivalue answer routing - lab
* Similar to simple but return up to 8 IP addresses maximum and health check can be attached. With health check, Route53 doesn't return IP addresses which are not healthy. Route53 gives different answer to different DNS resolver.
* In AWS Console:
  * Remove the A records of previous course
  * Create A record (=> repeat 4 times for the 4 EC2 instance):
    * Name: _empty_
    * No alias
    * Value: 1 IP address of the Virgina EC2 instance (not possible to provide more than one IP address)
    * Routing policy: Multivalue answer, Id: Virgina
    * TTL: 60sec
