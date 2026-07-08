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

IAM policies inheritance 

group policy: will apply for all the developers. they will inherit that. 
Even if the same person is in other group he'll also get the access of other 

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
 
