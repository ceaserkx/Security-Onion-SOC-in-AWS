# Security Onion SOC in AWS

A SOC environment hosted in the cloud is very achievable with a platform such as Security Onion. Security Onion is a free and open-source Linux distribution designed for network security monitoring, intrusion detection, and log management, providing a platform for security teams to monitor network traffic, detect potential threats, and investigate incidents through tools like Suricata, Zeek, and Kibana, all within a user-friendly interface; essentially acting as a comprehensive security monitoring solution for organizations of all sizes. 

Project credits go to [Nicole Enesse](https://www.linkedin.com/in/nicole-enesse-koch-b1878689/) for crafting the project setup procedures.  The Procedures section (up until subsection "8. Testing using Test My NIDS") of this repo is a slightly shortened version of the original procedure by Enesse. For verbatim, step-by-step procedure of the original project, visit Enesse's reference [Youtube video](https://www.youtube.com/watch?v=cwhvndEfuRw). 

## Contents
- [Scope](#scope)
- [Components](#components)
- [Procedure](#procedure)
- [Conclusion](#conclusion)

## Scope

In this project, I configured a SOC environment with Security Onion through AWS, leveraging various cloud features such as Security Groups for cloud firewall management, and Traffic Mirroring. A VPC was created to house the project components. Two EC2 instances were configured, one for the Security Onion 2 image, and one target host, an Amazon Machine Image (AMI). Elastic IP Management was utilized for managing public IP addresses for project components. SSH and Key Management was utilized to generate key pairs, assisting with securely remoting into AWS resources. After all components are configured, Test My NIDS by 3CORESec will be utilized to generate malicious traffic. This project aims to foster growth in configuring and deploying a secure (kinda) cloud network, and configuring and deploying Security Onion and
its toolset.

## Components
### Infrastructure
- AWS cloud platform
- VPC
- (2) EC2 Instances - (1) Security Onion 2, (1) Amazon Machine Image
- Security Groups
- Traffic Mirroring Session
- Elastic IP Management
### Security Tools
- Security Onion 2
  - IDS/IPS - i.e. Suricata, Zeek
  - Log Manager - ELK (Elasticsearch, Logstash, Kibana) stack
  - Alert Management - i.e. TheHive, Cortex
  - Packet Capture and Analysis - i.e. Wireshark, NetworkMiner
- [Test My NIDS](https://github.com/3CORESec/testmynids.org) by 3CORESec for attack simulation

## Procedure

### 1. Create VPC and Security Group
- Navigate to AWS console, then Services > VPC
- Create VPC, name it SO_Demo
- Input the IPv4 CIDR for the public subnet as 10.0.100.0/24, and click Create VPC.
- Navigate to VPC > Security Groups
- Click on the default security group and make necessary changes based on your requirements. Apply the following for your IP addresses.
  
  | Type       | Protocol | Port Range |
  | -----------|----------|------------|
  | SSH        | TCP      | 22         |
  | HTTP       | TCP      | 80         |
  | HTTPS      | TCP      | 443        |
  | ALL Traffic| ALL      | ALL        |
  
- Allow all traffic from your IP specifically (your home computer or laptop), as this will allow you to login to security onion through the web gui.
  
### 2. Configure and Deploy Security Onion 2 Image as an instance
- Launch an EC2 instance
- Search for Security Onion in the AWS marketplace
- Read and accept terms as required
- Select instance type t3a.2xlarge
- Create a new keypair, name it SO_demo
- Apply the VPC we created before (SO_demo) to this instance
- Set a public subnet
- Disabled Auto-Assign Public IP (will be using elastic IP for assignment)
- Under Advance Network Configuration, add a network interface
- In the Security Group section, select default group
- Launch instance
  
### 3. Elastic IP association
- Under Network and Security, navigate to IP options
- Allocate Elastic IP
- Choose Amazon's pool of ipv4 addresses
- In the EC2 dashboard, select the Security Onion instance, navigate to Network Interfaces and click eth0
- Clicked on interface ID (hyperlinked)
- Actions > Manage IP addresses
- Allocate an Elastic IP, ensure Amazon's pool of addresses is selected, then allocate
- Navigate to Network Interfaces tab, Actions > Associate Address, choose one of the Elastic IP addresses we just allocated to the network interface, ensuring to check for Allow reassociation.

### 4. Security Onion Configuration 
#### 4.1 Remoting into Security Onion
- Use Putty to ssh into Security Onion using the .ppk file (the keypair we downloaded before "SO_demo")
- Open Putty
- Navigate to Category (on the left) > Connection > SSH > Auth > Credentials
- Click the first "Browse" and select the .ppk file SO_demo
- Navigate to "Session" on the left. Enter the IP of the Security Onion EC2 instance, then select Open at the bottom of the Putty window.
- This should open the Security Onion setup page for the next steps
#### 4.2 CLI Setup of the Security Onion image
- Standalone mode
- Yes to "Would you like to continue"
- Choose standalone mode
- Write Agree
- Yes if more space is requested
- Choose hostname, can be something like "security onion demo"
- Leave description blank
- Yes for "network install"
- "Yes"
- Select Eth0; this is the management NIC
- "OK"
- Choose Direct for how you would like to connect to the internet
- Preflight checks will run
- The monitor interface will be sniffing all of the traffic. Choose eth1
- Choose Automatic
- Click Okay for home networks
- Choose Basic for type of manager you would like to choose
- Choose first option
- Choose Emerging threats Open. Various rules are available, but we will do Open for this project
- For optional services, the default is fine. this includes osquery, Wazuh, playbook, and strelka
- Click Yes for keeping the docker image
- Type in an email address/ and password to login. DO NOT LOSE THIS
- For how would you like to access the web interface, **choose OTHER. Because this is in the cloud.**
- Choose a name
- Set password for soremote user
- Choose basic for what type of configuration would you like to use
- Chose 1 for zeek & suricata process
- Choose Nodebasic for what type of config your would like to use
- Choose yes for So-Allow via web interface
- Type 0.0.0.0/0. This allows all traffic. But because we are in the cloud, AWS will be taking care of the firewall rules
- Setup will take about 30 minutes; follow next steps while this runs

### 5. Configuring Traffic Sniffing Security Group (eth1)
#### 5.1 Create a New Security Group
- Create a new security group, name it something like security onion sniffing
- Edit inbound/outbound rules as needed
  - If you want all traffic inbound/outbound, specify (0.0.0.0/0) for the ip range
- Assign the security group to a network interface
#### 5.2 Apply Security Group to Network Interface
- Select your Security Onion instance, navigate to networking tab
- Apply the created security group to the eth1 interface
- Keep in mind in a real situation you will want to be strategic with what you allow into the network. For the learning purposes of this project, we allowed all traffic.

### 6. Create a EC2 Instance to be Monitored (sniffed)
- Create an AMI machine as normal, ensure necessary settings below
  - t3 nano instance
  - public subnet
  - Enable Auto-Assign Public IP (only one interface on this instance, this is okay)
  - Choose the sniffing security group
  - Create a key pair just like the first instance we made before
  - Allocate and Associate Elastic IP as before

### 7. Create a Mirror Session
#### 7.1 Create Traffic Mirroring Filter first
- Navigate to Mirror Filter on left panel AWS features
- "Create Traffic Mirroring Filter"
- Give name and description, such as Security Onion Filter
- Allow all inbound/outbound traffic
#### 7.2 Create Traffic Mirroring Target
- Navigate to Mirror Target on left panel AWS features
- Choose eth1 interface as the sniffing target
#### 7.3 Create Traffic Mirroring Session
-  Navigate to Mirror Session on left panel AWS features
-  For the Mirror Source, select the network interface of the EC2 instance to be sniffed
-  For the Mirror Target, select the security onion sniffing interface we set up before
-  For Mirror Filter, select the filter we created.
-  Specify a session #

### 8. Generate Alerts with [Test My NIDS](https://github.com/3CORESec/testmynids.org) by 3CORESec
- Open security onion terminal, run the following
```
  ssh -i /path/my-key-pair.ppk onion@publicIpaddress
```
- Login to the target AMI terminal, run the following
```
    ssh -i /path/my-key-pair.ppk ec2-user@publicIpaddress
```
- Capture network on the target AMI by running the following on the security onion terminal
```
sudo tcpdump -i eth1 host linuxprivateipaddress
```
- On the target AMI, running the following to generate alerts from Test My NIDS
```
curl -sSL https://raw.githubusercontent.com/3CORESec/testmynids.org/master/tmNIDS -o /tmp/tmNIDS && chmod +x /tmp/tmNIDS && /tmp/tmNIDS
```
- Select any or all options. The CHAOS option will generate a satisfying amount of alerts and traffic.
- Refer back to the security onion web console after logging in, and check for alert (can take up to 10 minutes) 

## Conclusion

From a high level, created was a target instance monitored by a security onion instance in AWS. Security Groups (acts a firewall) were created for various network interfaces and can be modified depending on testing requirements. IP addressing was streamlined by the Elastic IP function of AWS. Alerts were generated by a simple network intrusion tool. Further exploration might include routing alerts through a messaging system, like slack or email, doing a deep dive of various alert types and experimenting with alert escalation in security onion, creating custom categorization for certain alert types (i.e. Remote Code Execution) by severity.  

Some steps may have been skipped to shorten the length of the Procedure section for simplicity. For other projects, you can go [here](https://github.com/ceaserkx/Project-List-2024). 
