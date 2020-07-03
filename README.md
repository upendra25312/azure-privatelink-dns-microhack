# **Azure Private Link DNS MicroHack**

# Contents //update this last//

[Lab Overview and Pre-req](#lab-overview)

[Challenge 1 : Connect to Azure SQL](#challenge-1--deploying-the-sap-s4-hana-landscape)

[Challenge 2 : Deploy service endpoints](#challenge-2--sap-parameter-tuning)

[Challenge 3 : Deploy private endpoint for SQL](#challenge-3--sap-hana-backup-using-azure-native-tools)

[Challenge 4 : Implement Azure DNS Private Zones integration](#challenge-4--securing-fiori-access-from-internet)

[Challenge 5 : Implement custom DNS integration (Windows DNS on Windows Server 2019)](#challenge-5--setup-dashboards-for-the-sap-environment)

[Challenge 6 : Implement On-Premises access using conditional forwarding](#challenge-6--hana-performance-validation)

[Challenge 7 : Optional, hard! Implement On-Premises access within using existing Azure hosted custom DNS server](#challenge-6--hana-performance-validation)

[Appendix](#appendix)

# Scenario

Contoso group is a consumer electrical retail company. The company works within a regulated industry and would like to secure their use of Azure PaaS services. As part of the evaluation the IT team has started to look at using **Azure Private Link**. 

## Context

This MicroHack scenario walks through the use of Azure Private Link with a focus on the required changes needed for DNS. Specifically this builds up to include working with a an existing custom DNS infrastrucutre (I.e. a customer using their own Virtual Machines for Internal DNS Resolution). In this lab we use Microsoft DNS running on top of Microsoft Windows Server 2019, but your customer could be using another DNS solution such as BIND on Linux, or Infoblox.

# Pre-requisites

## Overview

In order to use the MicroHack time most effectively, the following tasks should be completed prior to starting the session.

With these pre-requisites in place, we can focus on building the differentiated knowledge in Private Link that is required when working with the product, rather than spending hours repeating simple tasks such as setting up Virtual Networks and Virtual Machines. 

At the end of this seciton your base lab build looks as follows:

![image](images/base.png)

In summary:

- "On-Premises" environment simulated by Azure Virtual Network
- On-Premises contains a client VM and a DNS Server VM
- On-Premises is connected to Azure via a Site-to-Site VPN
- Azure contains a simple Hub and Spoke topology, with a client VM and DNS Server VM

## Task 1 : Deploy Template

We are going to use Terraform to deploy the base environment. It will be deployed in to your subscripiton, with resources running in the Azure West Europe region.

To start the terraform deployment, follow the steps listed below:

- Login to Azure cloud shell [https://shell.azure.com/](https://shell.azure.com/)
- Clone the following GitHub repository 

`git clone https://github.com/carlsyner/privatelink-dns-microhack.git`

- Go to the new folder ./privatelink-dns-microhack and initialize the terraform modules and download the azurerm resource provider

`terraform init`

- Now run apply to start the deployment (When prompted, confirm with a **yes** to start the deployment)

`terraform apply`

- Choose a suitable password to be used for your Virtual Machines

- Wait for the deployment to complete. This will take around 30 minutes (the VPN gateways take a while).

## Task 2 : Explore and verify the deployed resources

- Verify you can access the On-Premises Virtual Machine via RDP to the Public IP using the following credentials

Username: AzureAdmin

Password: <see step above>

- Verify that you are able to hop from the jumpbox to all Virtual Machines, using their Private IP addresses and RDP access. This step also proves that the Site-to-site VPN is online. 

### :point_right: Hint 

**Desktop shortcuts exist for easy RDP access to other virtual machines**

## :checkered_flag: Results

- You have deployed a basic Azure and On-Premises environment using a Terraform template
- You have become familiar with the components you have deployed in your subscripiton
- You are now be able to login to all VMs using the supplied credentials
- On-Premises VMs can contact Azure VMs

Now that we have the base lab deployed, we can progress to the Private Link challenges!

# Challenge 1 : Connect to Azure SQL

### Goal 

The goal of this exercise is to deploy a simple Azure SQL Sever and observe the default network behaviour when connecting to it. 

## Task 1 : Deploy an Azure SQL Server

Within the resource group named private-link-microhack-hub-rg, deploy a simple Azure SQL Server in the West Europe. Example config shown below.

![image](images/1.png)

How do we connect to this database by default, what networking information is needed, where do we find this?

## Task 2:  Test default connectivity to Azure SQL

Using the FQDN obtained in the previous step, confirm that your Azure Management Client VM can establish a connection to your SQL Server. Launch SQL Server Management Studio (SSMS) and input your SQL Server details and credentials.

![image](images/2a.png)

- Why does this connection fail?

## Task 2:  Modify SQL server firewall

- What settings on the Azure SQL server firewall do you need to modify?

- How can you verify which source Public IP address your Azure Management Client VM is using when accessing the Internet?

- How can you verify which destination Public IP is being used when connecting to your SQL Server FQDN?

![image](images/2.png)

## :checkered_flag: Results

- You have deployed a basic Azure SQL Server and connected to it from your Azure client VM. You have confirmed that you are accessing it via the "Internet" (This traffic does not leave the Microsoft backbone, but it does use Public IP addresses). The traffic is sourced from the dynamic NAT address on your client VM, and is destined to a public IP address sitting in front of the Azure SQL Service. 

# Challenge 2 : Implement Service Endpoints to restrict access to your Azure SQL Server

### Goal

The goal of this challenge is to implement Service Endpoints to restrict access to your Azure SQL Server; turning off inbound connections from Public IPs (The Internet) and only allowing access from the subnet within which your Azure Client VM Resides.

## Task 1: Remove public IP address from SQL Server firewall

Within the previous step we added the public NAT IP address used by our client VM to the SQL Server firewall. Please remove this IP address and save the firewall settings. This ensure that no inbound connecitons are permitted from any public IP address.

## Task 2: Enable Service Endpoints on your subnet

On your Infra subnet, within the Azure Spoke VNet, enable Service Endpoints for Azure.SQL. 

![image](images/3.png)

## Task 3: Enable Virtual Network access within SQL Server Firewall

Create a new Virtual Network rule within your SQL Server Firewall allowing access from the InfrastructureSubnet within which your Azure Client VM resides. Notice how it recognises that you have enabled the service endpoint for Azure.SQL in Task 1.

## :checkered_flag: Results

- Your SQL Server is no longer accepting connections from any Public IP address. It only accepts connections from your specific Subnet where service endpoints are enabled. Verify that you are still able to connect to your server via SSMS.

### :point_right: Hint 

**Even with service endpoints enabled, we are still sending destination traffic to the Public Interface of our SQL Server. The difference is how the traffic is sourced; now utilisaiton a special "VNET:Subnet" tag, rather than the Public IP address in Step 1**

# Challenge 3 : SAP HANA Backup using Azure native tools

### Goal

SAP HANA, like other database, offers the possibility to backup to files. In addition, it also offers a backup API, which allows third-party backup tools to integrate directly with SAP HANA. Products like Azure Backup service, or Commvault are using this proprietary interface to trigger SAP HANA database or redo log backups. SAP HANA offers a third possibility, HANA backup based on storage snapshots. The focus of this challenge is to configure **Azure Backup for SAP HANA**.
 


## Task 1 :  Prepare HANA VMs for Azure Backup

Contoso Group have asked you to prepare SAP HANA virtual machines for Azure Backup using BACKINT API.

You need to perform all pre-requisites and setup network connectivity to allow Azure Backup to connect to SAP HANA database as per this document.

[https://docs.microsoft.com/en-us/azure/backup/tutorial-backup-sap-hana-db](https://docs.microsoft.com/en-us/azure/backup/tutorial-backup-sap-hana-db)


### :point_right: Hint 


Your HANA database is running over a 2 node highly available cluster. For this hack **configure it only on the active node**. 

Also, you may need to register your HANA virtual machines to access SUSE repos and install package unixODBC, if not already installed.

Run following commands as root to install unixODBC

`zypper install unixODBC`

You will need to create SAP HANA HDBUSERSTORE Key named SYSTEM for user SYSTEM for password less authentication.

Run following commands as tstadm on HANA virtual machine

Syntax

`hdbuserstore -i SET <KEY> host:port <USERNAME> <PASSWORD>`

Test the connection using following command.

`hdbsql -U SYSTEM`

On the hdbsql prompt, type \s to check the status and ensure you can connect.



## Task 2 : Run backup for HANA using Azure Backup

Contoso Group have asked you to create a backup configuration to meet their requirements and to start the first backup right away.

Contoso deem it sufficient to take full backups once a week over the weekend, but want to optimize the cost and time required for backup and restore by using an optimized backup and retention policy. In the scenario where a restore becomes necessary, they want to be able to restore with no more than 20 minutes of loss of data.

Create an appropriate backup and retention policy and backup your SAP HANA database using Azure Backup.


### :point_right: Hint 

The key steps are - create a recovery services vault, configure backup, create a backup policy and backup.

[https://docs.microsoft.com/en-us/azure/backup/tutorial-backup-sap-hana-db#create-a-recovery-service-vault](https://docs.microsoft.com/en-us/azure/backup/tutorial-backup-sap-hana-db#create-a-recovery-service-vault)


## :checkered_flag: Results

- You have configured and run Azure backup for HANA database











  
# Challenge 4 : Securing Fiori access from internet

### Goal

Expose specific SAP endpoints to internet without adding public IPs to SAP VMs.


## Task : Make SAP Fiori launchpad accessible from Internet 

Contoso group wants to expose their SAP Fiori launchpad to internet. They would like to do this in a secure way without exposing any other parts of SAP.  Security team at Contoso wants proof that only Fiori launchpad can be accessed and any attempt to access other urls will result in error. 

URL for Fiori launchpad

http://hostname:port/sap/bc/ui2/flp


You can use Azure app gateway for this exercise. Below diagram explains how application gateway works. 

![App gateway image](https://docs.microsoft.com/en-us/azure/application-gateway/media/how-application-gateway-works/how-application-gateway-works.png)

### :point_right: Hint 

- You will need an additional subnet within the existing VNET (by adding a new address space) for App gateway creation
- Activate SAP ports HTTP (8000) and HTTPS (8001) using transaction SMICM.
- Note that the FQDN for SAP hostname doesn't resolve within the jumpbox. Use the IP address or just the hostname. 
- SSL is not in scope for this lab. Disable HTTPS switch for Fiori launchpad by changing the properties of service **/sap/bc/ui2/flp** in SICF. Changes need to be made in the Logon data tab and Error Pages tab (System Configuration). 
- Pay attention to NSG rules necessary for Application Gateway and your networking setup


## :checkered_flag: Results

- You have now demonstrated how SAP endpoints can be securely published to internet with the help of Azure App gateway.












# Challenge 5 : Setup dashboards for the SAP environment

### Goal 

Use Azure monitor to check performance of the SAP environment and setup dashboards.

## Task : Setup Availability and Performance dashboard for SAP 

The IT leadership team has requested for an availability and performance dashboard for the SAP environment. The dashboard must contain key performance metrics and availability for all the SAP VMs that have been deployed. 


### :point_right: Hint 


 Use Azure monitor for this exercise. Check what metrics are collected by default. There are several ways to visualize log and metric data stored in Azure Monitor
 It is sufficient to setup monitoring dashboard using **any one** of the following methods.

- Workbooks
- Azure dashboard
- PowerBI dashboard
- Grafana dashboard


[https://docs.microsoft.com/en-us/azure/azure-monitor/visualizations](https://docs.microsoft.com/en-us/azure/azure-monitor/visualizations)

See example dashboards below

![azuremonitor](images/azuremonitor.png)

![pbi](images/pbi.png)


## :checkered_flag: Results

- You now have centralized dashboards reporting performance and availability of your SAP S/4 HANA.














# Challenge 6 : HANA performance validation


### Goal

The goal of this exercise is to validate the setup of HANA infrastrcture.

## Task :  Check HANA parameters and performance 

You want to ensure that the HANA infrastructure is setup correctly and is performing as expected without involving the testing and functional teams to run tests.



### :point_right: Hint  

Use HANA mini checks SQL to verify the parameters and metrics of HANA. See SAP note # 1999993 - How-To: Interpreting SAP HANA Mini Check results

To check if infrastructure is setup correctly use HWCCT or HCMT tool. Both the tools are downloaded and available in the directory `/hana/backup/HANA_perftools` of HANA node tst-hana-vm-0

Note that some of the metrics will fail as we are using smaller VMs without write accelerators for our test deployment.




## :checkered_flag: Results

- You now successfully verified that SAP HANA system is setup correctly for optimum performance.




# Finished? Delete your lab

- Go to the new folder Private-Endpoint-Hack and run the following command

`terraform destroy`












# Appendix

### Generating SSH keypair

- Login to Azure cloud shell [https://shell.azure.com/](https://shell.azure.com/)
- Generate SSH keypair using the command `ssh-keygen` as shown in the screenshot below

  ![keygen](images/keygen.jpg)

- Use the public key (id_rsa.pub) for the deployment. 
- Download the private key to your desktop using the download button at the top. You will be using this file for logging into the linux VMs 


