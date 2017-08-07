# [VPC](https://aws.amazon.com/vpc/faqs/)

* 5 per region per account
* World &rarr; Internet _or_ Virtual Private Gateway &rarr; Router &rarr; Route Table &rarr; NACLs &rarr; Subnets &rarr; Security Group &rarr; Entity
    - Virtual Private Gateway: VPN termination
    - SGs, NACLs, Route Tables can span multiple subnets
    - _Subnets cannot span AZs!_
* Only _one_ Internet Gateway per VPC
* Default VPC
    - All public subnets
    - Route out to internet
    - Each EC2 instance has public and private address
    - Can delete but if you do, have to email support to restore
* No transitive peering. No, seriously.
* When new VPC is created, these are automatically created and nothing else
    - NACLs
    - Route Tables
    - Security Groups
* Flow logs capture IP address data flowing through network interfaces

## VPC Wizard

- Understand the defaults!
    + Main route table w/private subnet
    + Custom route table w/public subnet
    + Running NAT instance

## Subnets

* Largest network that Amazon gives us is `/16`
* When you create any subnet, AWS reserves 5 addresses in your block
    - `.0` and `.255` are not routable anyway...
    - `.1` for the router
    - `.2` for DNS
    - `.3` for some indeterminate future purpose
* "Auto-assign Public IP" is not enabled by default

## Route Tables

* _Via_ "Target" _to_ "Destination"
    - E.g. via IGW to `0.0.0.0/0` for route out to internet
* Don't use the main route table!
    - Every subnet you create is automatically associated with it
    - So if you add an IGW to the main one, you risk exposing intended private subnets to the internet. YUGE security problem.

## NAT

* Provide a secure way for private subnets to reach internet (or vice-versa)
* Always run in public subnet
* Need to add route from private subnet to NAT instance/gateway
* Two types
    - NAT Instance
        + An AMI provided by Amazon; run as EC2
        + So bandwidth depends on instance type
        + Disable source/destination check (!)
        + Must apply SGs to this!
        + Have to manage: failover, scaling, etc across multiple subnets
        + Must assign public IP
    - NAT Gateway
        + Managed by Amazon; scales up to 10GBps
        + Cannot apply SGs to this entity
        + Automatically assigned public IPs
* NATs _must_ be in public subnet

## Security Groups

* "First layer of defense"
    - Since it operates at the instance level
* Stateful!
    - So no need to add explicit outgoing for each incoming (e.g. and esp. ICMP)
* Default: block all incoming, allow all outgoing
* Cannot block specific IP with SG!
    - Use NACL for that

## NACL

* "Second layer of defense"
    - Since it operates at the subnet layer
* Stateless
    - Need to specify both inbound and outbound rules
* Block IPs using NACLs
* Default NACL comes with VPC and it allows all in/outbound traffic
* NACL to subnet is one to many
* Subnet to NACL is one to one (!!!)
* _Must_ associate subnet with custom NACL else it will be associated with default
    - This is not a good thing...
    - Subnets need some NACL
* NACL rules are evaluated in order
    - Lowest # to highest
    - If rule is matched, evaluation stops

## Peering

* Can be done across different accounts _but not across different regions_
* Cannot be done with VPCs that share the same CIDR block
* Cannot do transitive peering!
* Cannot do edge to edge routing via Gateway

## Cross Account Access

Let's say you want user `jane` to access a resource (like an S3 bucket). `jane` is in DEV account and the bucket is in PROD account.

* Prepare the PROD account
    - Create an IAM policy that specified access to the resource
    - Create a "Cross Account" IAM role
        + Specify the account ID of the DEV account as the "Trusted Entity"
        + Attach the IAM policy you created to this role
* Prepare the DEV account
    - Create an STS Trust Policy for `jane`
        + `sts:assumeRole` with the resource being the ARN of the role you created in the PROD account
* Profit

---

# [Storage Gateway](https://aws.amazon.com/storagegateway/faqs/)

* It's a _software_ appliance you download as a _VM image_ and run in your datacenter
    - Can run on VMWare ESXi or Microsoft HyperV
    - Has three 'proxies' for storage: file, volume, and tape
* File Gateway
    - NFS only!
    - Store stuff in S3
    - Provides a file-based interface to S3
* Volume Gateway
    - See as an iSCSI block device
    - Can take snapshots (compressed, incremental) and save to S3
    - Stored
        + Store locally, back up async to S3
        + 1GB to 16TB
    - Cached
        + Cache locally, store in S3
        + 1GB to 32TB
* Tape Gateway
    - Library
        + Instantaneous retrieval
    - Shelf
        + Stored in Glacier
        + Not instantaneous; can take 24 hours

---

# [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)

* It's the AWS CDN. You create *distributions* with it
* *Origins* for a single distribution can be
    - S3
    - EC2
    - Route53
    - ELB
* The _very first_ user/request to hit CloudFront distribution won't see the advantages. All subsequent users/requests will.
* Cached for TTL (Default 24 hours)
* Use *Invalidations* to clear cache (charged for this)
* Common distributions are web or [RTMP](https://en.wikipedia.org/wiki/Real-Time_Messaging_Protocol)
* Edge locations
    - About 50 at the time of this writing. They're separate from regions and AZs
    - They're not read-only. For example, S3's Transfer Acceleration uses Edge Locations to... accelerate uploads to S3.
* Can restrict by country (99.8% accuracy)
* Can use custom error pages
* Max delivery size is 20GB per file
* Secure objects in CloudFront or S3: use *presigned URLs* or *signed cookies*
* Firewalls: WAF or Web ACLs

---

# [Snowball](https://aws.amazon.com/snowball/faqs/)

* Petabyte-scale way to move data quickly into AWS. Used to be called "AWS Import/Export Data"
* Three kinds
    - Snowball
        + Just 80TB of storage
    - Snowball Edge
        + 100TB of _compute_ + storage
    - Snowmobile
        + A fucking _truck_ with 100PB of storage
* AES-256 encryption
* Full chain-of-custody audits possible
* Uses Kindles for the interface
* Import to and export from *S3*

---

# [Elastic Transcoder](https://aws.amazon.com/elastictranscoder/faqs/)

* Transcoder in the cloud
    - E.g. S3 -> Lambda -> ElasticTranscoder -> S3
    - Will even do thumbnails!
* Pay for _duration_ and _resolution_ of transcoding

---

# [Route53](https://aws.amazon.com/route53/faqs/)

* There's a limit of 50 domain names per account
    - Can be raised tho
* Know basic stuff about SOA, A, CNAME, NS, MX: all these supported by Route53
* AWS introduced "Alias Records"
    - Because ELBs and CloudFront don't always resolve to the _same_ IPv4 address
    - Another way to look at them is that they're not _pre-defined_.
* [BFD](https://www.urbandictionary.com/define.php?term=BFD). Why can't I just use ye olde CNAMEs in AWS?!
    - Because $$$, foo. You get charged for each CNAME resolution. Ridiculous.
    - CNAMEs don't allow you to map the zone apex ("naked domain") to another domain

## Policies

They're exactly what you'd guess them to be. Can adjust each of these on the fly, BTW.

* Simple
* Weighted
* Latency
* GeoLocation
* Failover

---

# [Kinesis](https://aws.amazon.com/kinesis/streams/faqs/)

## Kinesis Streams

- A _shard_ is a unit of data capacity
    + A way to horizontally partition a database for performance
    + Linear scaling of reads leads to exponential scaling of latency
- Shard limits
    + Read: 5 transactions per second or 2MB whichever comes first
    + Write: 1,000 transactions per second or 5 MB whichever comes first
- Producers -> Shards in Kinesis Streams -> Consumers (e.g. EC2)
- Keep for 24 hours (default) to 7 days
    + Increase with `IncreaseStreamRetentionPeriod`

## Kinesis Firehose

- Streams without shards or retention or, importantly, _consumers_
- Analyze with Lambda within Firehose (optional)
- Send to RedShift, S3, DynamoDB
- Just a highly performant data collection resource

## Kinesis Analytics

- Use a SQL-like language to analyze data in both Streams and Firehose
- Send to DynamoDB, RedShift, Elasticsearch Cache, S3

---

# [Glacier](https://aws.amazon.com/glacier/faqs/)

* 1kb to 40TB in "archive"
* Archives are immutable
* Can download 10GB per month for free

---

# [Direct Connect](https://aws.amazon.com/directconnect/faqs/)

* Dedicated connection to AWS datacenter
    - Bypass ISPs!
    - Customer Datacenter &rarr; (TelCo Circuit) &rarr; Line to Direct Connect facility &rarr; Private Rack (Circuit Terminated) &rarr; AWS Rack &rarr; (Dark Fiber) &rarr; AWS Datacenter
    - VPN is less reliable
        + But can be set up in minutes
        + OK for low to moderate bandwidth
    - A dance between _resiliency_ and _bandwidth_
        + AWS bandy the latter as the primary reason to use DC
        + Makes sense, since you can configure multiple VPCs
* 802.1q VLANs, so partition the dedicated connection into virtual interfaces
    + So access public instances like S3 using public IP space
    + So access private resources like inside a VPC using private IP space
* 10Gbps and 1GBps
* DC is the connection between the Direct Connect Facility and Datacenter!

---

# [IAM](https://aws.amazon.com/iam/faqs/)

* _Everything_ is global: users, groups, policies, roles
    - ARN-wise, that is
* "Power User" policy: can do everything except manage IAM
* Root account signin URI is not 'pretty' like the IAM one (e.g. `mycompany.signin.aws.amazon.com/console`)
* Can attach roles to running EC2 instances (this wasn't the case before)
* Strange (to me) thing with instance in one region and bucket in another: _you must specify the `--region` flag_
    - Recommendation is to always use this flag
* 100 Groups
    - 10 groups per user
    - _Cannot_ nest groups!
* 5,000 users
    - Use Temporary Security Credentials if more needed
* 500 roles

---

# [Security Token Service](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html)

* Use for limited and temporary access
* User comes from three sources
    - Federation via AD
    - Federation via Identity _Stores_ like Facebook, Google, LinkedIn
    - Cross-account access
* Identity Brokers: take identity from A and join (federate) to B
    - You develop this yourself (Gluu is an example)
* Steps
    - App gives credentials to Identify Broker
    - Identity Broker uses these to auth against store like LDAP, gets token
    - Takes token to STS
    - STS hands four values back to app (via Broker)
        + Access key
        + Secret Access key
        + Token
        + Lifetime of Token
    - App then uses these values to talk to IAM and get access
    - Auth against LDAP first, STS next!
* Another scenario
    - Identity Broker uses roles instead of user credentials
        + It gets the roles associated with user from LDAP
        + Application assumes the role and interacts with AWS
    - This was confusing to me... :/ Don't fully understand it.

---

# [CloudWatch](https://aws.amazon.com/cloudwatch/faqs/)

* Has Standard and Detailed Monitoring
    - Each depends on kind of instance
    - EC2: 5 min Standard and 1 min Detailed
    - RDS, ELB, OpsWorks, Route53: no charge for detailed billing
* Custom metrics for _any_ instance: cannot go below 1 minute
* Metrics
    - Average
    - Min
    - Max
    - Sum
    - Data Samples
    - If you collect a lot of logs within the 1min period, you have to "bunch" them up using these and then send to CloudWatch
* Uses UTC by default or if zone unspecified, else can specify local timezone
* CloudWatch Alarms
    - Only send to email
* Events
    - Send to Lambda, some API, etc
* Logs
    - Need to install an agent on the EC2 instance
    - Stored in S3
    - JSON and text-based common log format (no XML)
* You only get CPU, Disk, Network, Status
    - _No Memory yo_! Memory is a _custom metric_
    - Does _not_ do disk usage! Only IOPS!

## EC2 Monitoring

* There's a difference between a _System_ and _Instance_ status check
    - Former is the physical server
        + All sorts of issues with it: network loss, power, hardware and software issues
        + Best thing to do is to restart VM. This will launch it on another host.
    - Latter is your instance, yo
        + Just reboot. Fix any problems first.
* For Custom Metrics (like disk usage), download and install AWS tools
    - Need role for EC2 to publish to CloudWatch!
    - Zip file containing `mon` Perl scripts (yuck)
    - Put this in User Scripts... burn an AMI... OpsWorks script... whatever
    - Run in `crontab`
    - See in CloudWatch
    - Profit

## EBS Monitoring

* `VolumeQueueLength`: Number of RW operations waiting to be completed
* Volume Status
    - `ok`: all is well
    - `warning`: degraded or severely degraded
    - `impaired`: stalled or not available
    - `insufficient_data`: what it says

## RDS Monitoring

* Can get metrics from outside RDS instance
    - `DatabaseConnections`: Number of active connections to DB. Is your app closing connections after opening them?
    - `DiskQueueDepth`: number of requests pending/in queue. Scale up if non-zero
    - `FreeStorageSpace`: what it says
    - `ReplicaLag`: latency between master and read-replicas
    - `Read|WriteIOPS`
    - `Read|WriteLatency`
    - Many, many more
* Can get events from inside RDS
    - _Subscribe_ to events (like failover) for a bunch of instances or a specific one

## ELB Monitoring

* Send metrics to CloudWatch
    - _Only when_ requests flowing and
    - _Only_ in 60s intervals
* Lots of them...
    - `HealthyHostCount`
    - `UnHealthyHostCount`
    - `Latency`: time from when a request leaves the ELB until response is received
    - `RequestCount`: total number of completed requests received and sent to backend
* Important ones
    - Request waiting in ELB: `SurgeQueueLength`
    - Requests rejected because queue is full: `SpilloverCount`

## ElastiCache Monitoring

* Monitor four things
    - CPU Utilization
        + Memcached
            * Multi-threaded
            * Threshold is 90%; exceed? Add more nodes (use alarm)
        + Redis
            * Single-threaded
            * Threshold is (90/cpu_count)%
    - Swap Usage
        + Like ye olde swap
        + Memcached
            * Threshold is 50Mb; exceed? Increase `memcached_connections_overhead`
        + Redis
            * Not applicable
    - Evictions
        + Replace old with new
        + Memcached
            * No recommendation
            * Scale up (more memory) or scale out (more nodes)
        + Redis
            * No recommendation
            * Scale out (more nodes (read-replicas))
    - Concurrent Connections
        + No recommendations
        + Set an alarm on concurrent connections!
        + Your app might not be releasing connections

## Centralizing Monitoring

* Make sure that the appropriate ports are open
    - ICMP, 3306, etc.
* You can set another Security Group as the source in any given SG
    - Remember that SGs are `DENY ALL` when created

---

# [OpsWorks Stacks](https://aws.amazon.com/opsworks/stacks/faqs/)

* Provisioning and Configuration Management powered by Chef.
* OpsWorks Stacks is _configuration_ management
    - Beanstalk is _application_ management... it's higher level
    - CloudFormation is lower level, more foundational, and broader
* It's just automation with more control for DevOps-types
    - Creates IAM policies for users
    - Creates SGs for ports between layers
* **Stacks**
    - A collection of AWS instances like ELBs, EC2s, RDS, etc that are required by your app(s)
* **Layers**
    - Contained within a stack and are a way to separate purpose. E.g. you can have a web application layer composed of EC2s. A database layer composed of RDS. A load balancing layer composed of ELBs. And so on.
* An instance _must_ be assigned to at least one layer.
* You have a lot of preconfigured layers.
* Updates to your stack are targeted at one or more layers.
* When you attach/add an ELB to a layer, all the instances registered with the ELB are _removed_ since OpsWorks manages the ELB for you.
* There's Chef 11 and Chef 12 stacks. The former is all pre-baked recipes. The latter is BYOB.
* OpsWorks manages these kinds of instances
    - 24/7 - Run all the time
    - Time-based - what you think it is
    - Load-based - what you think it is (CPU, Memory, App load)
* You can run recipes based on _Lifecycle Events_. You can mostly guess what these might be
    - Setup
    - Configure
    - Deploy
    - Undeploy
    - Shutdown

## A Word on [OpsWorks for Chef Automate](https://aws.amazon.com/opsworks/chefautomate/faqs/)

* This is a _managed Chef server_. That's it.
    - Backups et all are managed for you.
* OpsWorks Stacks, however, does all that _and_ runs stuff based on lifecycle events using a [Chef Solo](https://docs.chef.io/chef_solo.html) client that is installed on the EC2 instances _on your behalf_ (automation again.)

---

# [CloudFormation](https://aws.amazon.com/cloudformation/faqs/)

* _Minimum required_ is the `Resources` section
* Need Resource Conditionals? Use `WaitCondition`
* TODO: Deep topic... more notes

---

# Elastic LoadBalancer ([Classic](https://aws.amazon.com/elasticloadbalancing/classicloadbalancer/faqs/) and [Application](https://aws.amazon.com/elasticloadbalancing/applicationloadbalancer/faqs/))

* Two types w.r.t. DNS
    - External ELB with external DNS
        + Go across AZs
        + Cannot go across VPCs... or different regions
    - Internal ELB with internal DNS
        + E.g. use for a web application tier in a public or private subnet
* Single ELB cannot have multiple SSL
* Understand timeouts!
    - Interval x Un/Healthy Threshold before something is marked un/healthy
* Planning ahead with load-testing
    - Need to submit ticket to AWS
    - Given them the
        + Window when you expect/will simulate increase
        + Expected request rate
        + Total size of a typical request + response
    - They'll "pre-warm" the ELB for you
* "Evaluate Target Health": confirm a connection
* Connection draining: will hold in-flight requests for 300s default
* Instances run if ELB is deleted
    - `InService` if OK, `OutOfService` if not
* AutoScaling & Termination priority
    - Oldest Launch Config &rarr; Closest billing hour &rarr; Random
* You _have_ to enable cross-zone loadbalancing!
        Only balances across AZs by default
            To balance across _all_ instances in _all_ AZ, enable this!

## Stickiness

* Requests are sent to the same instance behind the ELB
    - Use if you're doing server-side sessions
        + Really, you'd use ElastiCache or something else... right?
* Two kinds of "stickiness"
    - ELB/Duration-based
        + Need to understand this more
    - Application-based
        + Need to understand this more
    - You can have problems when managing stickiness!
        + E.g. ELB + AutoScaling. The latter will work but requests will be routed to the same instances due to high stickiness

---

# [Elastic Block Store](https://aws.amazon.com/ebs/faqs/)

* _Must be in same AZ as EC2_
* Max 16TiB; min varies depending on type
* Max 5,000 volumes per account, with 10,000 snapshots
* Here's the deal: you can't mount one EBS to many EC2s
    - Use EFS
* Delete on termination is turned on by default
* SSD - gp2 - General Purpose: 3 IOPS per GB
    - Use if require < 10,000 IOPS
* SSD - io1 - Provisioned IOPS
    - Use if need > 10,000 IOPS
* HDD - ST1 - Throughput Optimized
    - _You can't boot from these!_
    - Use for Big Data, Data Warehousing, etc
* HDD - SC1 - Cold
    - _You can't boot from these!_
    - Cheap
    - Used for infrequent access
* HDD - Magnetic
    - Cheapest
    - _Can_ actually boot from these
* No IOPS are guaranteed for magnetic storage...
* _Total_ (not individual) volume storage across all classes is 20TB
* Use `lsblk` to list mounts on EC2 instance (can use `mount` but the former command shows up on tests for some weird reason.)
* Basic flow: Shutdown EC2 -> Snapshot volume -> Create new different/larger volume from snapshot -> Boot up EC2 -> Attach to EC2
* Can modify type of volume by using snapshots
* Can change size on the fly _except_ for Magnetic
    - For Magnetic: stop, snapshot, create volume
    - Have to wait 6 hours if you make another change
    - Change change size up only
    - Best practice tho is to stop, detach, snapshot, change, reattach, etc

## IOPS and Bursting

* You get 3 IOPS/GB
    - Capped at 10,000 IOPS on a gp2
    - Upgrade to io1 to get provisioned IOPS
* You can burst up to 3,000 IOPS
    - You can burst up to (3000 - (3 * volume_size)) IOPS
* Need more? That's where the IOPS Credit comes in
    - 5.4M IOPS in credit per volume
    - Enough to sustain 3,000 IOPS for 30 minutes
    - There is a recovery time:
        + Earn back credits if not bursting
* More IOPS? Get a larger volume
    - AWS incentivizes you to get larger ones

## Initialization, AKA "Pre-warming"

* Not necessary anymore
* _However_, you need to initialize snapshots
    - Do this by reading all blocks

## Instance Stores and EBS

* Former is ephemeral, the latter isn't.
    - Historically, all volumes were ephemeral
    - Then added EBS for persistence
* There are root and additional volumes
    - Instance Stores and EBS can be attached to either
    - Root Volume
        + Instance Store: 10GiB
        + EBS Volume: 1-2TiB depending on instance
            * Terminated automatically by default!
            * Unless you uncheck "Delete on Termination"
            * Or set the `deleteontermination` flag
    - Additional volumes
        + Instance Store: Deleted on termination
        + EBS: Persist automatically
* Stopping
    - Instance stores cannot be stopped: Only reboot or terminate
    - EBS can be stopped
* Upgrading
    - EC2 with root as Instance store: can't do shit.
    - EC2 with root as EBS: stop and upgrade kernel, RAM, disk, etc
* Data
    - Instance Store
        + Data persists on reboot
        + Data lost when stopping or terminating (or host failure)
    - EBS
        + Persists on root volume if "Delete on termination" unchecked
        + Persists automatically on additional volumes
* Big point: _Instance Store cannot be stopped_!
    - Reboot: Data exists
    - Terminate: * Poof! *

---

# Support

* Basic, Developer, Business, Enterprise levels
* Enterprise: they'll get back to you in 1 hour guaranteed

---

# [Workspaces](https://aws.amazon.com/workspaces/faqs/)

* A VDI in the cloud
* Don't need an AWS account
* `D:\` is backed up every 12 hours
* User is local admin on the machine

---

# [Relational Database Service](https://aws.amazon.com/rds/faqs/)

## Backups

- Automated
    + Full daily snapshot + Transaction logs
    + Can do *1 second* Point in Time recovery
    + Retention period is 1 - 35 days
    + You experience latency during backups
    + Deleted if you hose RDS instance
- Snapshots
    + These are _user-triggered/manual_
    + Events for snapshot: creation, deletion, notification, restoration
    + Present even after you hose RDS
- Restoring both implies a _different_ RDS instance
- Maintenance Window
    + Weekly
    + A time for AWS to perform security patches and updates
    + Select 30 minute window
        * Else, AWS will select a random day in the week
        * Each region has its own "time blocks" within which maintenance windows are created

## Encryption

- Only for certain instance types: mediums are not supported, look for larges
- Supported for all except Aurora (?)
- All your backups and snapshots are automatically encrypted
- Uses AWS KMS for encryption key management

## Redundancy

- Read Replicas
    + _Asynchronous_ snapshots
    + Use for performance
    + Need Automatic Backups turned on
    + You get 5 per RDS instance
    + You can do read replicas of read replicas _only_ for MySQL
    + No SQL Server or Oracle support :(
    + No MySQL < 5.6
        * _Only_ InnoDB for Read Replicas
    + Can have replica in another region: only MySQL and MariaDB :/
* Set up a multi-AZ failover
    - Backups _and_ restores are taken from secondary to prevent IO suspension
    - Can _force_ failover with `RebootDBInstance` call (i.e., reboot your instance)
    - Synchronous Replication and Failover are automagic
    - This is for disaster recovery and _not_ for scalability!
    - Use Read Replicas for that!
* Read Replicas
    - Take with `CreateDBInstanceReadReplica` call
    - If Multi-AZ enabled, secondary snapshotted, no hit
    - If not enabled, up to 1 min IO suspension
    - Can have them in different regions _only_ with MySQL
    - Read Replicas of Read Replicas _only_ with MySQL
    - They are tied to an AZ! Cannot do Multi-AZ with Read Replicas
    - Can have a Read Replica in another region!
    - Need a snapshot or Automated Backups enabled
* "Replica Lag" is the key CloudWatch metric
    - _Most_ of the time, just recreate the Read Replica

## Scaling up

- Not as easy (on-the-fly) as DynamoDB
- Take snapshot -> Restore to larger instance type -> Adjust code etc.

## Miscellaneous

* To check for failure, look for an _error node_ in the RDS SDK response
* Can get reserved RDS instances!

## Specific Databases

### SQL Server

* Cannot increase storage on an active instance
* 10GB database size
* 300GB max instance size (so 30 DBs max on an instance)
* Doesn't use AWS' failover, HA, Multi-AZ; uses its own called "SQL Server Monitoring" because Microsoft.

### Aurora

* MySQL compatible and supposedly up to 5x faster
* Scales in 10GB increments up to 64TB
    - 32vCPUs with up to 244GB of memory
* Redundancy
    - 2 copies of data in each AZ
    - Minimum 3 AZs
    - So minimum 6 copies of data
    - Self-healing
    - Can handle loss of 2 copies transparently
* Replication
    - Two kinds
        + Aurora Replicas (15 max.)
        + MySQL Read Replicas (5 max.)
    - Key difference is failover: happens automagically to Aurora Replica and not to the MySQL one

---

# [Simple Workflow Service](https://aws.amazon.com/swf/faqs/)

* Task-based API as opposed to SQS which is message-based
* Retention period is 1 year!
* Elements/Actors
    - Starters: initiate the workflow
    - Deciders: control workflow
    - Workers: Actually do the task defined in workflow
    - All these do not need to be AWS resources (e.g. can be a human)
* You can group together related workflows in a _domain_

---

# [RedShift](https://aws.amazon.com/redshift/faqs/)

* Data warehousing solution
    - About operations on columns (sums, averages, other stats) and not single rows
    - Understand the difference between OLTP and OLAP
        + OLTP (transaction) is better with RDBMS
        + OLAP (analytics) is better with RedShift
* Block size is 1,024KB
* Leader and task nodes
* Can use a *single node* of 160GB
* Case use a *multi-node* with a _leader_ node and _compute_ nodes
* Fast because
    - Columnar storage
    - Data stored sequentially on disk
    - Massively Parallel Processing (MPP): distribute data and query across all nodes
    - Compression is great: it samples data and automatically chooses the best compression scheme. Doesn't require indexes

---

# [DynamoDB](https://aws.amazon.com/dynamodb/faqs/)

* All SSD storage
* Across three different facilities (not the same as AZs...)
* Understand _eventually_ and _strongly_ consistent reads
    - The former means that data is consistent across all facilities in usually under a second
    - Just think about what your application needs
* Up to 35 levels of nesting for a single document/item
* First 25GB of data in DynamoDB is free
* `BatchGetItem` can get 100 items not exceeding 16MB
* `BatchWriteItem` can put 25 items not exceeding 16M
* 400KB max item size
* RCU and WCU
    - 1 read/write per second
    - Read: up to 4kb then additional RCU
    - Write: to 1kb then additional WCU
    - Start with strongly for base
    - Both are limited to 400kb max item size
* Figure out RCU and WCU per item, then multiply by desired # of reads/writes to get total RCU and WCU
    - Leave alone if you want Strongly Consistent Reads
    - Divide RCUs by 2 if you want Eventually Consistent Reads
* LSI (Partition Key + Sort key) **only** at table creation
* GSI (Any Key + Sort Key) can be created any time

## Querying

* Search by primary key/Partition ID attribute
* Returns the whole damn item
    - _Unless_ you supply `ProjectionExpression`
* Returns stuff sorted in ascending order (ASCII char code for letters)
    - _Unless_ you set `ScanIndexForward` to `false`
* Default is eventually consistent

## Scan

* Search by any attribute
    - Will dump the _whole_ table, then you supply a filter to limit results, then a `ProjectionExpression` to limit attributes in results. Whew!
* Obviously less efficient than Query
* Can chew up provisioned RCUs!
* Returns the whole damn item
    - _Unless_ you supply `ProjectionExpression`

`ProvisionedThroughputExceededException` when you exceed limits.

## Web Identity Providers

* Authenticate to Provider (like Facebook) -> Get Web Identity Token
* Send `AssumeRoleWithWebIdentity` request to the AWS Security Token Service
    - Web Identity Token
    - App ID of Provider
    - ARN
* STS will then send back temporary credentials that last **one hour**
    - 15 mins to an hour... but one hour is default
* FIRST need to create a role to use all this

## Operations

* Can do conditional updates which are idempotent _or_ use atomic updates/counters which are not
* `BatchGetItem` can get 1MB of data of up to 100 items.

## Misc

`ConditionExpression`:

* Boolean functions: ATTRIBUTE_EXIST, CONTAINS, and BEGINS_WITH
* Comparison operators: =, <>, <, >, <=, >=, BETWEEN, and IN
* Logical operators: NOT, AND, and OR.

---

# [Simple Queue Service](https://aws.amazon.com/sqs/faqs/)

* Used to decouple components
    - Producer makes data faster than consumer can process
    - Consumer might be offline
* Understand "at least once" delivery
    - Understand [visibility timeouts](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html#configuring-visibility-timeout) (30s default, 12hr max)
    - Can change _per message_
* Understand long-polling
    - Send `ReceiveMessage` to get the message but add `WaitTimeSeconds`
    - Or, set the queue attribute `ReceiveMessageWaitTimeSeconds`
* 256KB size of message in queue
    - Minimum 1Kb
    - One request can have 1-10 messages (still within 256KB)
    - Still billed at 64KB chunks
    - Each 64KB of a request is counted as a single request
* No guarantee that messages are delivered in order! No FIFO
    - Only for standard queues... they introduced FIFO queues
    - FIFO queues limited to 300 transactions/s
    - Client needs to process and delete messages
* Messages kept for 4 days (default)
    - Range is 60s to 14d
* SQS never pushes; clients always pull
* Keeps messages for 14 days max (4 days default, 60s minimum)
* Delete queue? **Poof** go the messages...
* AutoScaling groups can see size of queue and spin up or spin down instances: basis of some of the largest websites
* Consumer can longpoll (max 20s)
    - Save money
* `ChangeMessageVisibility`

---

# [Simple Notification Service](https://aws.amazon.com/sns/faqs/)

* Push-based (instead of SQS which is pull-based)
* Core concept is a *topic*
    - Can have multiple subscribers (10 million subscriptions)
    - Can have multiple formats
    - Publish once to a topic and let SNS handle format
* 100,000 topics/account, 10M subscribers/topic
* All messages are stored redundantly across AZs
* Send to
    - HTTP/S
    - Email/Email-JSON
    - SQS
    - Some application (REST endpoint)
    - Lambda
* TODO: write down the components of an SNS message
* TODO: describe the "fan-out" pattern of SNS + SQS

---

# [Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/faqs/)

* Stores app and logs in S3
* Have access

**Preconfigured**

* IIS .NET
* .NET
* Java
* Python
* PHP
* Node.js
* Go
* Tomcat
* Ruby
* Packer

**Docker, Preconfig**

* Go
* Python
* Glassfish

**Raw Docker**

* Docker
* Multi-container docker

---

# [Lambda](https://aws.amazon.com/lambda/faqs/)

* Prices based on
    - Number of requests and
    - Duration of execution
* First 1M are free! $0.20 for next 1M (!!!)
* Duration is start of function to return or termination; rounded to nearest 100ms; charged for every GB-second used (!!!)
* 1-300s execution time
* TODO: This is an obviously deep topic... more notes!

---

# [S3](https://aws.amazon.com/s3/faqs/)

* Understand the consistency models!
    - Read after Write Consistency for PUTs (new objects... why is this not a POST?)
    - Eventual consistency after update PUTs or DELETEs (existing objects)
* Key-value store, not block-based
* Components of an S3 object
```
{
    Key: Value,
    Metadata: ...,
    VersionID: ...,
    SubResources: {
        ACL: ...,
        BucketPolicy: ...
    }
}
```
* Unlimited _objects_, but restricted to 100 buckets per account
* New buckets private by default
    - Use *Bucket Policies* to set bucket-level policies
    - Use ACLs for fine-grained access policies per object/key
* Eleven 9's durability
    - Only RRS has 99.99
    - Availability same 99.99 across all three
    - 99.9% availability SLA guarantee by Amazon
* Single PUT max 5GB
* `>` 100MB? use multipart
* `>` 100 `PUT`s and `>` 300 `GET`s per second? Make sure you add some randomness for performance
* Minimum storage duration of 30 days for S3-IA and 90 days for Glacier
* Transfer Acceleration: User -> Edge Location -> Bucket
    - URIs look like `https://bucket.s3-acceleration.amazonaws.com`
* Encryption
    - Use AES-256
    - At rest: Client side
        + [AWS is not responsible for this](http://i0.kym-cdn.com/photos/images/newsfeed/000/284/529/e65.gif)
    - In transit: Client side is using SSL/TLS
    - At rest: Server side is
        + *SSE-S3 Managed Keys*: Each object encrypted with a key employing strong MFA which is in turn encrypted with a rotating master key. Amazon manages these keys for you.
        + *SSE-C*: Customer-provided keys. You bring the encryption keys. Amazon manages _only_ the encryption _process_. Only when versioning is enabled on bucket.
        + *SSE-KMS*: Uses AWS' Key Management Service. Kind of the same as SSE-S3 but has a few more benefits (e.g. audit who is decrypting what and when, specify your own envelop key)... no need to go into depth about these for now
* Static assets and serving them
    - Bucket: `https://s3-us-east-1.amazonaws.com/bucket`
        + This has changed is simpler now: `https://s3.amazonaws.com/bucket`
    - Website: `https://bucket.s3-website-us-east-1.amazonaws.com`
    - Route53 and S3 - bucketname must be same as domain!!
* Once versioning is turned on, it can be disabled but not removed entirely
* Versions are NOT incremental! New versions of the object occupy their real space
* If restoring deleted object, delete the "Delete Marker" to restore. Yep.
* Cross-region replication
    - Versioning must be enabled on source and destination
    - Regions must be different (duh...)
    - Only new objects and updated objects are replicated to the new regions
    - Permissions are replicated
    - _Cannot_ chain replication events between three or more buckets: one-to-one only
    - Delete markers are replicated
* "Expiration" does _not_ mean deletion: it means that you attach a "Delete Marker" to the object. It's an invisibility cloak. The object's still there, the versions (if enabled) are still there, and you're paying for it all.
* S3 Analytics: Analyze access pattern

---

# [EC2](https://aws.amazon.com/ec2/faqs/)

* DR MCGIFT PX
* Understand the states: `running`, `stopped`, `terminated`
    - Billed only when `running`
* Metadata available at `http://169.254.169.254/latest/meta-data`
* Four classes/pricing models: On-demand, Spot, Reserved, Dedicated
* Spot
    - If the spot price goes up and instance is terminated (by AWS), you get that hour for free
    - If _you_ terminate this, you pay, sucka
* 1 or 3 years for reserved instance
* Reserved vs. Dedicated
    - Reserved guarantees computing power saying nothing of underlying hardware
    - Dedicated
        + Host: single tenant + hardware
        + Instance: hardware
* EIP same after reboot

## CLI

* `aws ec2 run-instances --instance-ids ...` will create new instance(s)
* `aws ec2 start-instances --instance-ids ...` will start stopped instance(s)
* `aws ec2 describe-instances --instance-ids ...` will list all running instance(s)
* `aws ec2 terminate-instances --instance-ids ...` will terminate instance(s)

## [Placement Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)

* Collection of homogenous (recommended) EC2 instances
* Why? Guarantees low, 10Gbps network latency. Think Hadoop cluster.
* Can only be in a single AZ
* Can only use certain instance types: general, compute, storage, memory, accelerated
* Cannot merge placement groups
* Cannot add _existing_ instance; create AMI and boot up

## [AMIs](https://aws.amazon.com/amazon-linux-ami/faqs/)

* If they're S3-backed, the root volume goes **poof** if the instance is rebooted (?)

---

# Costs and Billing

* Visualize with **Cost Explorer**

## EC2

* Understand spot, on-demand, reserved, dedicated instances
    - Depending on app, maintain a few reserved instances and use autoscaling to recruit on-demand as load rises
* Understand Heavy versus Medium utilization
    - Going from xlarge &rarr; large &rarr; medium &rarr; small:

## [Consolidated Billing](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html)

* Associate up to 20 accounts with a paying account
    - Can add more by contacting AWS
    - Don't deploy anything there!
    - Top level account cannot access anything in child account
* Using the "AWS Organizations" tool
    - Create/invite other accounts
    - Create policies
    - Create OUs
    - Now create associations!
* Nice in terms of volume discounts BTW; consolidations save $$$
    - S3
    - EC2, esp. with unused reserved/dedicated instance quotas
* Auditing with CloudTrail
    - Enable in root/paying account
    - Enable cross-account access via Bucket Policy
    - Store all logs in root/paying account
* Can set billing alarm (for the whole account)

---

# [ElastiCache](https://aws.amazon.com/elasticache/faqs/)

* In-memory key/value store
* Durability considerations...
* Uses memcached or Redis
    - Redis can be Multi-AZ
* TODO: Deep topic... more notes!

---

# [Elastic File System](https://aws.amazon.com/efs/faq/)

* NFS in the cloud, basically
* Can scale like crazy to petabytes
* Region-level: Spread across multiple AZs!
* An EFS instance must be in the same Region as the machines

## Snapshots

* Stored in S3
* Incremental
* Encrypted EBS = Encrypted Snapshot
* Unencrypted EBS => Cannot do encrypted Snapshot
* _Cannot share encrypted snapshots_
    - Key is associated with your account
* For RAID snapshots, you _must_ stop I/O in some way

## Roles

* Can attach to running EC2 instances
* Since they're under the aegis of IAM, they're _global_
    - No need to create a role per region. IAM, yo.

---

# [Risk Management](https://d0.awsstatic.com/whitepapers/compliance/AWS_Risk_and_Compliance_Whitepaper.pdf)

* Bi-annual evaulation of risks by Amazon
* They do vulnerability scans themselves
    - You can too but must let them know and not violate Acceptable Use Policy

---

# [Disaster Recovery](https://media.amazonwebservices.com/AWS_Disaster_Recovery.pdf)

* Understand Recovery Point Objective (RPO) and Recovery Time Objective (RTO)
    - Balance between the two for costs and resilience
* Spectrum of RTO/RPO: Backup & Restore > Pilot Light > Warm Standby > Multi Site
    - High/High > Med/Med > Low/Low > None/None
* "Tiered Backup Solutions"
    - E.g. Backup to S3 and then to Glacier (or VTL)
* Backup & Restore
    - Sort of mirrors traditional datacenter operations
    - Set retention policy
    - Set security policy
* Pilot Light
    - Like in a gas heater; small light keeps going and when required can light the whole furnace
    - Keep a minimum core that's always running
    - Figure out the core
        + Active Directory
        + RDS
    - All other services come up (automation) around the Pilot Light
    - Use pre-allocated EIPs, ENIs w/MAC addresses!
    - Use ELBs and DNS CNAMEs (ALIASes?) to switch over traffic
* Warm Standby
    - More resources, still minimal, running all the time
    - More extended than Pilot Light
    - More involved
        + Use automation for config management
        + Use AMIs
        + Use both horizontal and vertical scaling to bring up to production capacity
* Multi Site
    - You have a full copy active/active stack on-prem and in AWS
    - Requests don't go to just one; send to both
    - Change DNS weighting
    - Change application logic for failover
* Restoring
    - Backup and Restore
        + Freeze changes to DR site
        + Take a backup
        + Restore data to primary
        + Point users to primary
        + Unfreeze DR
    - All others
        + Reverse mirroring from DR -> Primary
        + Freeze DR
        + Point users to primary
        + Unfreeze DR

## Automated Backups

* RDS
    - Store in S3
    - See note about Multi-AZ and IO suspension
    - Delete instance => All _automated_ backups deleted
        + Manual snapshots are safe
    - On restore, you can change medium _and_ engine type
* ElastiCache
    - Store in S3
    - Redis only
    - Whole cluster
    - Performance hit unavoidable
* RedShift
    - Store in S3
    - Automatic: 1 day retention
    - Incremental
* EC2
    - Store in S3
    - _No_ automation available
    - Incremental
    - Performance hit unavoidable

---

# High Availability Notes

* "Elastic" and "Scalable" are two slightly different things
    - Elastic is short term; 'bursts'; can come back to original capacity
    - Scalable is long term
* When you're growing, you want a balance between Scaling Up and Scaling Out
    - _Automated Elasticity_: Lambda is a perfect example
    - Elasticity + Autoscaling
        + Scaling up easy; scaling down not so much
        + Scaling out easy; scaling back easy
* If network bottleneck, most likely a scaling up problem

---

# [Shared Model of Responsibility](https://aws.amazon.com/compliance/shared-responsibility-model/)

* Understand what Amazon is responsible for and isn't
    - Use common sense...
* Three levels: Infrastructure Services, Container Services, and Abstract Services
    - Figure out which services are managed; security patches for these are managed by AWS. E.g. RDS. You still secure the database with SGs, strong passwords, etc.
    - Moving from LTR: more and more is managed by AWS
* Container: AWS takes care of OS and platform
* Abstract: S3, DynamoDB, Lambda
    - AWS takes care of everything except for customer data and client-side encryption; that's your headache.

## Security

 * Storage Decommissioning
     - Zero out and destroy all disks
     - DoD 5220.22M and NIST 800-88 standards
 * AWS provides all sorts of Network Monitoring and Protection
     - DDoS
     - MitM
     - Port Scanning
     - Packet Sniffing
     - `arp` poisoning
     - Does not allow IP Spoofing
     - Prevented using an AWS-managed host firewall
* Use _Xen_ as hypervisor
    - Firewall betweek hypervisor's physical and VM's virtual interfaces
        + Network > Physical Interface > AWS Firewall > Customer Security Groups > Virtual Interface > Hypervisor > Instance
        + So the AWS firewall takes care of 'filtering' traffic to customers' security groups? Easy to visualize but incorrect
    - RAM is separated off similarly as well
    - Cannot access disk on Hypervisor
        + Disk is zeroed out before reallocated to another customer
        + RAM is zeroed out as well
* Compliant with ISO, SOC 1,2,3, PCI DSS Level1 (can't start taking credit card details... need more), blah, blah
    - HIPAA, MPAA, Cloud Security Alliance
* Can do port scans but _must request beforehand_
* Can secure stuff in CloudFront using x509 certificates
* 24x7 security with datacenters
* Principle of Least Privilege w.r.t. access
* Elastic LoadBalancer supports SSL termination
* Trusted Advisor can be used for audits but not in a completely exhaustive sense

## [Trusted Adviser](https://aws.amazon.com/premiumsupport/ta-faqs/)

* High-level information
    - Performance
    - Cost Optimization
    - Security
    - Fault Tolerance
* Supplies this information
    - Security
        + Security Groups - Specific Ports Unrestricted
        + IAM Use
        + MFA on Root Account
        + EBS Public Snapshots
        + RDS Public Snapshots
    - Performance
        + Service limits

## Services that allow `root` access

* EC2
* EMR
* Elastic Beanstalk
* OpsWorks

---

# [AutoScaling](https://aws.amazon.com/autoscaling/faqs/)

* Free of cost
* Two conflicting schedules? ERROR
* Can it reboot instances? No
* Failed to launch instances... what happens?
* Detailed monitoring enabled by default if and only if launch config is through CLI or API (else basic monitoring)
* TODO: Deep topic... more notes

---

# Miscellaneous

* Can pick AZ for RDS instance
* Read-after-write for all S3 buckets except US-STANDARD (this is a redundant term)
* Placement Group <-> AZ
* NAT instance degradation? Bump up NAT instance type
* Can transfer reserved instance from one AZ to another
* S3 has MFA delete policy as additional security to delete stuff
* ELBs only within region: peak load
* 20 EC2 instances per region soft limit
* No charge for public datasets
* Review IP addressing...
* What is free tier?
    - S3 with 5GB
    - 750 hours/mo EC2
    - 750 hours/mo ELB

## Questions

* For SSH to a bastion host: do you open up a NACL or a Security Group or both? I think it's both. Sample question made it seem like it was one but not both.
* Does an ELB have _only_ a one-to-one relationship with an AutoScaling group?
* How are fault tolerance and high-availability related?
* `t2` instances: do you _only_ run these using an EBS-backed paravirtualized AMI? Or is HVM applicable?
* How do you handle network 'bursts' to an instance or set of instances in a subnet? Upgrade instance types? Load-balance a NAT Gateway? Attach ENIs?
* How do you do an EBS RAID snapshot?
    - Best practices and suspend IO?

