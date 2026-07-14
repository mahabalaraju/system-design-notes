AWS Certified Developer Associate:

https://aws.amazon.com/search/?searchQuery=%20Exam%20Prep%20Official%20Question%20Set%3A%20AWS%20Certified%20Developer%20-%20Associate

Step 3: Your 2-Hour Daily Study Blueprint (30-Day Plan):

Phase             Days                             Daily 2-Hour Strategy                                                                                       Focus Area
Phase 1        Days 1–10         Watch Stephane's videos at 1.25x or 1.5x speed. Do the IAM, EC2, and S3 labs.                                       Core Infrastructure & Security
Phase 2        Days 11–20        Focus deeply on the Serverless Section (Lambda, API Gateway, DynamoDB, Cognito). Build the hands-on projects.       Serverless & Application Dev
Phase 3        Days 21–25        Learn CI/CD tools (CodeCommit, CodeBuild, CodeDeploy, CodePipeline) and AWS CloudFormation.                         Deployment & Monitoring 


 How do you choose AWS region ?  
 
 compliance: with data governance and legal requirements: data never leaves a region without your explicit permission 
 
 proximity to customers : reduced latency 
 
 available services within a region : new services and new features aren't available in every region 
 
 pricing : pricing varies region to region and is transparent in the service pricing page. 
 
                                                                                      ---------
                                                                                         IAM:
                                                                                      ---------


IAM --> Identity and access management, global service
Root account created by default, shouldn't be used or shared 
users are people within your organisation, and can be grouped
groups contains only users and not other groups
users don't have to belong to a group and user can belong to multiple groups

IAM permissions

users or groups can be assigned json documents called policies
these policies define teh persmissions of the users 
In aws you apply the least privilege principle : don't give more permissions than a user needs

IAM policies 

inline policy : for single person 
group policy: will apply for all the developers. they will inherit that. 
              Even if the same person is in other group he'll also get the access of other 


IAM policies structure 

consists of 
version : poliy language version, always include "2012-10-17"
Id : an indentifier for the policy(optional) 
statement: one or more individual statements(required)

statements consists of : 
sid: an identifier for the statement(optional)
effect: wheather the statement allows or denies access(allow, deny) 
principal: accoutn/ user / role to which this policy applied to 
action: list of actions this policy allows or denies 
resource : list of resources to which the actions applied to 
condition : conditions for when this policy is in effect(optional) 

test-developer :  https://115154236565.signin.aws.amazon.com/console
username       : test-developer
Password       : Hmbr@1997


coming to the password policy you have default and custom as well where you can set your own like number of characters , symbols , length , expiration time and you have dashboard to configure --> account settings and password policy 

and you can enable MFA by IAM -> security credentials 

How you can access AWS ?

AWS management console (protected by password + MFA) 
AWS commandline interface (CLI) : protected by access keys 
AWS software developer kit (SDK) : for code : protected by access keys 

access keys are generated through the aws console 
user_name ~= accesskey Id 
password ~= SecretAccess key 


AWS cli : its a tool that enables you to intract with aws services using commands in your command line shell 
direct access to the public apis of AWS services 
its open source . alternative to the aws management console where you can develop the scripts to manage your resources . 


AWS SDK : (software development kit) 

language specific APIs (set of libraries)
enables you to access and manage aws services programmatically 
embedded within your application 

AWS cloudshell: 

its region specific : its just a terminal on aws cloud . 

try : aws --version 
clear 
aws iam list-users 
aws iam list-users --region
ls

IAM roles for services

 some AWS service will need to perform actions on your behalf
 to do so, we will assign permissions to aws services with IAM roles 
 
 common roles : 
  ec2 intance roles 
  lambda function roles 
  roles for cloudformation 
  
 IAM security tools (Audit)
 
 IAM credentials report (account-level) 
  a report that lists all your accounts users and the status of their various credentials 
  
 IAM access advisor(user-level)
  access advisor shows the service permissions granted to a user and when those services were last accessed
  you can use this information to revice your policies.
  
  shared responsbility model for IAM : 
  
  means what you are responsible and aws for ? 
  
            AWS                                             You 
  infrastructure (global network security)        users, groups, roles, plicies management and monitering 
  configuration and vulnerability analysis    enable MFA on all accounts 
  compliance validation                       rotate all your keys often 
                                              Use IAM tools to apply appropriate permissions 
									                                 		  analyse access pattern and review permissions 
											  
											  

Billing and cost management:
Budget: we have budget so if it exceeds it will send the mails we added.
Bills: we have sofiscated dashboard to view the bills on what and which . 


EC2 (Elastic compute cloud): infrastructure as a service 
  is one of the most popular of AWS offering 
  
  Its mainly consists in the capability of : 
  renting virtual machines(EC2) 
  storing data on virtual drives(EBS) 
  distributing load across machines(ELB)
  scaling the services using an auto-scaling group(ASG). 
 
 knowing EC2 is fundamental to understand how the cloud works . 
 
 EC2 sizing and configuration options 
 operating OS : mac, linux and windows 
 CPU, RAM 
 storage space : Network-attached(EBS and EFS)
                 hardware (EC2 instance store) 
 network card: speed of the card, public IP address 
 firewall rules : security group 


 bootstrap script (configure at first launch) : EC2 user data
 Its possible to boostrap our instances using an EC2 user data script . 
 bootstrapping means launching commands when a machine starts 
 that scripts is only run once at the instance first start 
 EC@ user data is used to automate the boot task such as: 
 installing updates 
 installing softwares 
 downloading common files from the internet 
 anything you can think o files
the EC2 user data script runs with the root user 


Notes: 
GB (Gigabyte) = Base 10 (Decimal)
How humans count.
1 GB = 1,000,000,000 bytes (Exactly 1 Billion bytes).

GiB (Gibibyte) = Base 2 (Binary)
How computer hardware and AWS operate.
Uses multiples of 1,024.1 GiB = 1,073,741,824 bytes (2³⁰ bytes).

1 KiB (Kibibyte) = 2^10 bytes = 1,024 bytes
1 MiB (Mebibyte) = 2^20 bytes = 1,048,576 bytes
1 GiB (Gibibyte) = 2^30 bytes = 1,073,741,824 bytes
1 TiB (Tebibyte) = 2^40 bytes = 1,099,511,627,776 bytes
1 PiB (Pebibyte) = 2^50 bytes = 1,125,899,906,842,624 bytes

EC2 instance types - general purpose 

great for a diversity of workloads such as web servers or code repositories 
balance between : 
compute , memory and networking 

m5.2xlarge 
m: instance class 
5: generation ( AWS improves them over time)  
2xlarge : size within the instance class 
  
ex. t, m , a

compute optimized 
 great for compute intensive tasks that require high performance processors: 
 bathc processing workloads 
 media transcoding 
 high performance web servers 
 high performance computing(HPC) 
 scientific modeling and machine learning 
 dedicated gaming servers
 
Ex. C
 
memory optimized 

Fast performance for workloads that process large data sets in memory 
use cases : 
high performance , relational / non relational databases 
distributed web scale cache stores 
in memory databases optimized for BI( business intelligence) 
applications performing real time processing of big unstructured data

EX. r, x and z

storage optimized 

great for storage-intensive tasks that require high, sequential read and write access to large data sets on local storage

use cases : 
high frequency online transaction processing(OLTP) systems 
relational and nosql databases 
cache for in-memory databases (ex. redis)
data warehousing applications 
distributed file systems 

ex. i1 d, h1 

go the ec2instances.info
 
Introduction to security groups 

security groups are the fundamental of network security in aws 
they control how traffic is allowed into or out of our EC2 instances . 
security group only contain allow rules and they can be reference by IP or by security group 
they are acting as a firewall on EC2 instances 
they regulate : 
  access to ports 
  authorized IP ranges - IPv4  and IPv6 
  control of inbound network (from other to the instance)
  control of outbound network (from the intstance to other) 
  Its good to maintain one seperate security group for ssh access
  
 classic ports to know : 
 
 22 = ssh (secure shell) - log into a linux instance 
 21 = Ftp (file transfer protocol) - upload files into a file share 
 22 = Sftp (secure file transfer protocal) - upload files using ssh 
 80= http - access unsecured websites 
 443 = https - access secured websites 
 3389 = rdp - remote desktop protocol - log into a window instance 

SSH : Secure Shell. 
 It is a secure way to remotely control your AWS cloud servers from your own computer.
 PuTTY is a free ssh client. 
 Pem if window < 10 
 .ppk file if window >10
 
 EC2 instances purchasing options 
 
 On-demand instances - short workload, predictable pricing, pay by second 
  pay for what you use : 
  linux or windows (billing per second) other are per hour basis . 
  highest cost but no upfront payment , no long term commitment . 
  recommended for short term and un-interrupted workloads, where you can't predict how the application will behave . 
 
 reserved instances( 1 and 3 years) 
  reserved instances - long workloads  
  
  upto 72% discount compared to on-demand 
  you reserve a specific instance attributes (instance type, region, tenancy, os) 
  payment options - no upfront (+) , partial upfront (++) , full payment (+++). 
  reserved instances scope - regional or zonal (reserve capacity in AZ)
  recommended for steady-state usage application ( think database)
  you can buy and sell in the reserved instances marketplace 
  
  convertible reserved instances - long workloads with flexible instances
  can change the ec2 instance type , instance family , os , scope and tenancy
  upto to 66% discount . 
   
 saving plans ( 1 and 3 years) - commitment to an amount of usage, long workload 
   get a discount based on long-term usage ( upto 72% - same as RIs) 
   commitment to certain type of usage (10$/hr for 1 or 3 years) 
   usage beyong ec2 savings plans is billed at the on demand price . 
   locked to specific instance family and aws region (ex. m5  in us-east1 ) 
   flexible accross : 
     instnace size (ex. m5.xlarge, m5.2xlarge)
	 os(ex. linux , windows)
	 tenancy(host, dedicated, default)
 
 
 spot instances - short workloads, cheap, can loose instances (less reliable) (batch jobs , data analysis , image processing, any distributed workloads  , workloads with a flexible start and end time. not suitable for critical jobs or databases. 
 
 dedicated hosts - book an entire physical server, control instance placement. 
 
 A physical server with EC2 instance capacity fully dedicated to your use 
 allows you address compliance requirements and use your existing server bound software licenses (per socket, per core , pre - vm software licenses) 
 purchasing options : 
 on demand - pay per second for active dedicated host
 reserved - 1 or 3 years ( No upfront, partial upfront, all upfront) 
 the most expensive option 
 useful for software that have complicated licensing model (BYOL- bring your own licence) 
 or for companies that have strong regulatory or compliance needs 
 
 
 dedicated instances  -  no other customer will share your hardware 
 
 Instances run on hardware thats dedicated to you (multi-tenant). 
 may share hardware with other instances in same account 
 no control over instance placement ( can move hardware after stop / start) 
 
 capacity reservations - reserve capacity in a spacific AZ for any duration. 
 
 reserve on demand instances capacity in a specific AZ for any duration 
 you always have access to EC2 capacity when you need it
 no time commitment ( create/cancel anytime) no billing discounts 
 combine with regional reserved instances and savings plans to benifit from billing discounts 
 you're charged at on-demand rate wheather you run instances or not 
 suitable for short term , uniterrrupted workloads that needs to be in  a specific AZ

 Whats an EBS volume ? 

An EBS (Elastic block store) volume is a network drive(consider USD flash drive) you can attach to your instance while they run 
It allows your instance to persist data, even after their termination 
they can only be mounted to one instance at a time( at a ccp level)
they are bound to a specific specific availability zone

Analogy : think of them as a "network USB stick". 
	
Note : ccp - certified cloud practitioner - one EBS can be only mounted to one EC2 instance associate level(solution architect, developer, sysops):  EBS Multi-Attach. This specific feature allows you to attach a single, specialized Provisioned IOPS (io1 or io2) EBS volume to up to 16 AWS Nitro-based EC2 instances simultaneously within the same availability zone.

EBS volume 

its a network drive (i.e., not a physical drive)
 it uses the network to communicate the instance, which means there might be a bit of latency 
 it can be detached from an ec2 instance and attached to another one quickly 

its locked to an availabilityzone (AZ) 
 An EBS volume in us-east-I a cannot be attached to us-east-Ib
 To move a volume across, you first need to snapshotit 

have a provisioned capacity( size in GB's and IOPS) 

 you get billed for all the provisioned capacity 
 you can increase the capacity of the drive over time
	 
	EBS Snapshots -- ( like git commits )

	 Make a backup (snapshot) of your EBS volume at a point in time 
	 Not necesary to detach volume to do snapshot, but recommended 
	 can copy snapshots across AZ or Region
	 
	EBS Snapshots features 

	EBS Snapshots Archive (Git Cold Storage)
	 move a Snapshots to an "archieve tier" that is 75% cheaper . 
	 takes within 24 to 72 hours for restoring the archive 
	 
	Recycle bin for EBS snapshots 
	 setup rules to retain deleted snapshots so you can recover them after an accidental deletion 
	 specify retnetion ( from 1 day to 1 year) 
	 
	fast snapshot restore (FSR) 
	force full initializaation of snapshot to have no latency on first use ($$$) . 
	
	the snapshots are very important in retaining the deleted volumes and instances and completely create new replica of those .. in few clicks . 
	
	AMI
	
	AMI = amazon machine image 
	AMI are a customization of an EC2 instance 
	 you add your own software, configuration, operating system, monitering .. 
	 faster boot/ configuration time because all your software is pre-packaged. 
	AMI are built for a specific region(and can be copied across regions) 
	you can launch EC2 instances from : 
	a PUblic ami  :  AWS provided. 




 
 
