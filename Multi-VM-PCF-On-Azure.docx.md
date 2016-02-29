---
title: Multi-VM PCF On Azure
layout: post
author: jgammon
permalink: /multi-vm-pcf-on-azure.docx/
source-id: 1XiqZcd7DHOcTN7_SrfhkVLfE7dwSUkjcrlUbL63H9kE
published: true
---
![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_0.png)

Multi-VM PCF installation on Azure

(Single VM OSS CF document available [here](https://docs.google.com/a/pivotal.io/document/d/1gq5wbVCuZI_RQUTXBC2wlfDpbyXfa2YUS4vJ4--b8ls/edit?usp=sharing))

 **THIS IS A DRAFT !!!**

[[TOC]]

# Overview

This document describes how to deploy BOSH, Cloud Foundry OSS and PCF on Azure

# Planning and Pre-Requisites

### Azure Resources

Bosh and CloudFoundry will use the following resources:

*  Azure Resource Manager Resource Groups, Virtual Networks, Subnets

* Azure DNS (wildcard CNAME and/or A records)

* AD Client ID and Application Service Principal

* Storage Account(s) – containers for block, tables

* Virtual Machines

* Load Balancer & Availability Sets

* (mult-VM only) 100+ Core & VM Quota (Standard Dv2 Instances)

* Azure Resource Manager ExpressRoutes and/or Site-to-Site VPN

### Azure Security Object Creation

The following security objects will be created as part of the Cloud Foundry installation (these are automated through an Azure Resource Manager template, or can be done manually via the Azure CLI or Azure Portal):

*  Client ID

* Application Service Principal

These objects are used by BOSH, Pivotal's integrated installation & provisioning system, to perform its VM and storage lifecycle management duties automatically.

### Cloud Foundry DNS and SSL Requirements

The Cloud Foundry API and applications require at least one wildcard DNS entry and wildcard TLS certificates.  Two subdomains are configured, one for system hosts, and a default domain for applications (example: **_*.system.cloud.com_** and **_*.dev.cloud.com_**).  All deployed applications will dynamically receive a hostname under **_*.dev.cloud.com_** at deployment time, whereas Cloud Foundry system hosts are located under the system domain, e.g. **_api.system.cloud.com_**. 

There is no restriction on how many subdomains may be associated with Cloud Foundry, and they may be available either globally to all developers, or to specific developers assigned to an organization.

One or more TLS certificate(s) may be generated that correspond these wildcard DNS names using x.509 Subject Alternative Names for multiple hosts.   Non-wildcard TLS certificate(s) are also possible if an Azure Load Balancer is used.    Either **self-signed** or **CA-signed** TLS certificates are usable, depending on your preference.

### Load Balancing Requirements

Cloud Foundry for Azure automatically creates an Azure Load Balancer for use with interacting with applications and the API.  

Cloud Foundry comes with an HAproxy load balancer out of the box, for deployments that do not wish to use the Azure Load Balancer pools.   HAproxy still utilizes an Azure Load Balancer object for network access, but sidesteps the the Azure load balancer pools in favour of using automatically Inbound NAT rules to HAproxy.   This configuration is simpler for smaller, non-HA Cloud Foundry deployments, but is not recommended for production deployments as it is a single point of failure.

For a production deployment, Pivotal recommends the Azure Load Balancer pools to be configured to load balance the Cloud Foundry Router VMs in one or more availability sets.

### Azure Resource Requirements

* Cloud Foundry requires an Azure Resource Manager **Subscription**,

* A typical **multi-VM** Pivotal Cloud Foundry install requires upwards of 25 VMs or more.

* **This requires Increased Azure quotas**, preferably within a **region** with premium storage (e.g East US 2), with the following quotas as safe for a small installation:

    * Total Regional Cores      	100

    * Standard-A Family Cores	50

    * Standard-D Family Cores	50

* Via ARM template or Azure CLI / Portal**, an Azure Resource Group**, including:

    * **Virtual Network** and **Subnet**

    * **One or more storage accounts** (subject to I/O performance requirements); one is sufficient for dev performance

    * Within the storage account,

        * **Two storage account containers (bosh **and** stemcell)**

        *  **One storage account table** (**stemcells**)

    * A **Static IP address** &** load balancer** for accessing Cloud Foundry applications and the API

### Pivotal Cloud Foundry Installation Process

1. *Pivotal is happy to walk any individuals through the installation process in person or over WebEx*

2. Pivotal Cloud Foundry (PCF) **_cannot be manually installed_**, it must be installed **automatically** through **BOSH**, an integrated VM/storage provisioning, configuration, packaging, and health management system for composite software releases. ([http://bosh.io](http://bosh.io))

3. **Pivotal Cloud Foundry Operations Manager** provides a Web GUI for managing BOSH and Cloud Foundry.  Operations Manager is projected to be available for the Azure platform in mid-2016.  Until then, **_the command-line BOSH tools_** must be used to install and maintain PCF on Azure.

4. A small BOSH Client VM (vanilla Ubuntu Linux) is created initially via **ARM template** or **manually** to install and administer BOSH and CF

5. A BOSH Director VM, image provided by Pivotal, is then provisioned via **command-line** to manage the cluster. 

6. **_The BOSH command line and BOSH Director VM then provisions and manages all Cloud Foundry VMs, Disks, and IPs via the Azure API. _**

7. Approximately 20 VMs or more, along with 40 to 60 Virtual Hard Drives, varying on performance, availability, & service requirements, are created to host Cloud Foundry.  The size of VMs vary depending on performance, but the majority will be Standard D1 or D2.

### Azure Network Requirements

1. Cloud Foundry is installed on an **Azure Resource Group**, which is the equivalent of a **virtual private cloud** with **private subnet**.  This subnet is customizable to any private IP range desired.

2. If this Cloud Foundry instance requires access to internal data center servers, this requires an **Azure Resource Manager ExpressRoute** or **Site-to-Site** **VPN**.

3. Pivotal has an example reference architecture and implementation for an **OpenVPN** based setup to enable cross data-center connectivity to Azure in the interim; alternatively, an **Azure Resource Manager Site-to-Site VPN** is possible if preferred.

4. The long run solution would be to provision and configure an **Azure Resource Manager ExpressRoute.**

# Creating a Microsoft Azure Account

If you already have a Microsoft Azure Account, continue to the next section; otherwise click on the Portal (portal.azure.com), It will take you to login using your Microsoft account if you are not already logged in. We will assume here that you already have a Microsoft account, otherwise go ahead and create one by hitting the following link: [https://signup.live.com](https://signup.live.com)

If this is the first time you are logging in you will not have any Azure subscription in your account. To add a new one, access the SIGN UP FOR WINDOWS AZURE link ([https://account.windowsazure.com/SignUp](https://account.windowsazure.com/SignUp)). Your personal details should appear from your account info, and you will need to verify it with an SMS or call verification: 

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_1.png)

Once your account is verified, you can enter your credit card details. However, do not panic if you want a free trial or "pay as you go"; you won't get automatically signed up for any premium subscription. Accept the agreement and click on the Signup button; your card details will be validated and you will be taken to the subscriptions page, where you will be pleased to find that you already have a free trial !!! This is shown in the following screenshot:

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_2.png)

If you are only interested in deploying Cloud Foundry in a single VM, the free trial subscription will be enough and you should skip the next section. On other hand if you are interested in a multi-VM deployment, a customized subscription is required. 

# Adding a subscription

Click on the "add subscription" button to add a new subscription.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_3.png)

Chose the **Pay-As-You-Go**  subscription (or another option that best fit your needs). After which you will get a purchase confirmation on your screen, as shown in the following screenshot:

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_4.png)

Once the payment information is confirmed, you will be taken back to the **subscriptions** page, where you can see your new subscription being listed.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_5.png)

# Creating a Service Principal

Azure CPI provisions resources in Azure using the Azure Resource Manager (ARM) APIs. We use a Service Principal account to give Azure CPI the access to proper resources. This can be done using either the portal or the CLI, however as part of this exercise, we will be using the Azure CLI. Skip the next section if you already have the CLI installed.

### Install Azure CLI

To create and manage Azure resources on the command line, we need to install the Azure CLI.

1. Install and configure Azure CLI using the documentation that can be found **[HER**E](https://azure.microsoft.com/en-us/documentation/articles/xplat-cli-install/)

**NOTE: **

* It is suggested to run Azure CLI using Ubuntu Server 14.04 LTS or Windows 10.

* If you are using Windows, it is suggested that you use **command line** but not PowerShell to run Azure CLI

2. Configure Azure CLI

Because the Azure Resource Manager mode is not enabled by default, use the following command to enable Azure CLI Resource Manager commands.

> azure config mode arm

3. Login

Now let test if we can successfully login to azure using the CLI.

> azure login

info:    Executing command **login**

info:    To sign in, use a web browser to open the page https://aka.ms/devicelogin. Enter the code FR6MB95F8 to authenticate.

Use a browser to go to the website indicated, enter the code, and then enter your Microsoft account credentials when prompted. Once the credentials are verified the CLI will successfully login.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_6.png)

Set Default Subscription

      

1. **Check whether you have multiple subscriptions**

> azure account list --json

	Sample output:

	

		[

  {

    "id": "1b365ce1-10f7-46b6-a8af-c221fb314f92",

    "name": "Pay-As-You-Go",

    "user": {

      "name": "3cb98565-22e8-4ce3-94bc-4bb4086c3eb7",

      "type": "servicePrincipal"

    },

    "tenantId": "1fb8ec91-a6f7-4b90-b7b1-3efb5523fd34",

    "state": "Enabled",

    "isDefault": true,

    "registeredProviders": [],

    "environmentName": "AzureCloud"

  }

]

**	**Now, get the following values from the output, there are required for the next steps:

* **SUBSCRIPTION-ID** - the row id

* **TENANT-ID** - the row tenantId

**NOTE**: If your **TENANT-ID** is not defined, one possibility is that you are using a personal account to log in to your Azure subscription. See 1.b Configure Azure CLI on how to fix this.

2. **Set your default subscription**

	> azure account set <SUBSCRIPTION-ID>

	

Example:

	

**>** azure account set 1b365ce1-10f7-46b6-a8af-c221fb314f92

Sample output:

**info**:    Executing command account set

**info**:    Setting subscription to "Pay-As-You-Go" with id "1b365ce1-10f7-46b6-a8af-c221fb314f92".

**info**:    Changes saved

**info**:    account set command **OK**

### Create an Azure Active Directory (AAD) application

	

	Create an AAD application with your information.

**>** azure ad app create --name <name> --password <password> --home-page <home-page> --identifier-uris <identifier-uris>

Explanation:

* **name**: The display name for the application

* **password**: The value for the password credential associated with the application that will be valid for one year by default. This is your CLIENT-SECRET.

* **home-page**: The URL to the application homepage. You can use a faked URL here.

* **Identifier-uris:** The comma-delimited URIs that identify the application. You can use a faked URL here.

	Example: 

> azure ad app create --name "Service Principal for BOSH" --password "password" --home-page "http://BOSHAzureCPI" --identifier-uris "http://BOSHAzureCPI"

	Sample Output:

info:    Executing command ad app create

+ Creating application Service Principal for SYO                               

data:    **AppId**:                   0417e000-3506-4d6b-84a6-e5d97bbdd79f

data:    ObjectId:                6f861a3c-8e7c-4097-a925-54c179d811a5

data:    DisplayName:             Service Principal for SYO

data:    IdentifierUris:          0=http://BOSHAzureCPI

data:    ReplyUrls:              

data:    AvailableToOtherTenants:  False

data:    AppPermissions:       

data:                             claimValue:  user_impersonation

data:                             description:  Allow the application to access Service Principal for SYO on behalf of the signed-in user.

data:                             directAccessGrantTypes: 

data:                             displayName:  Access Service Principal for SYO

data:                             impersonationAccessGrantTypes:  impersonated=User, impersonator=Application

data:                             isDisabled: 

data:                             origin:  Application

data:                             permissionId:  57febf3d-369f-4c66-9a7a-2cd4ef131ebe

data:                             resourceScopeType:  Personal

data:                             userConsentDescription:  Allow the application to access Service Principal for SYO on your behalf.

data:                             userConsentDisplayName:  Access Service Principal for SYO

data:                             lang: 

info:    ad app create command OK

	**NOTE**: **AppId** is your CLIENT-ID you need to create the service principal.

### Create a Service Principal

> azure ad sp create <CLIENT-ID>

Example: 

	> azure ad sp create 0417e000-3506-4d6b-84a6-e5d97bbdd79f

Sample Output:

info:    Executing command ad sp create

+ Creating service principal for application 0417e000-3506-4d6b-84a6-e5d97bbdd79f

data:    Object Id:               8df3fc23-8549-4ab8-8b3e-5bdb9ac029c3

data:    Display Name:            Service Principal for BOSH

data:    Service Principal Names:

data:                             3cb98565-22e8-4ce3-94bc-4bb4086c3eb7

data:                             http

Info:     ad sp create command OK

You can get **service-principal-name** from any value of **Service Principal Names** to assign role to your service principal.

### Assign roles to your Service Principal

Now you have a service principal account, you need to grant this account access to proper resource use Azure CLI.

> azure role assignment create --spn <service-principal-name> --roleName "Contributor" --subscription <subscription-id>

Example:

> azure role assignment create --spn "http://BOSHAzureCPI" --roleName "Contributor" --subscription 1b365ce1-10f7-46b6-a8af-c221fb314f92  

Then you can verify the assignment with the following command:

> azure role assignment list --spn http://BOSHAzureCPI 

Sample Output:

info:    Executing command role assignment list

+ Searching for role assignments                                               

data:    RoleAssignmentId     : /subscriptions/1b365ce1-10f7-46b6-a8af-c221fb314f92/providers/Microsoft.Authorization/roleAssignments/b88043e3-5e7c-4bb7-aa2b-c87f629ae315

data:    RoleDefinitionName   : Contributor

data:    RoleDefinitionId     : b24988ac-6180-42a0-ab88-20f7382dd24c

data:    Scope                : /subscriptions/1b365ce1-10f7-46b6-a8af-c221fb314f92

data:    Display Name         : Service Principal for BOSH

data:    SignInName           :

data:    ObjectId             : 8df3fc23-8549-4ab8-8b3e-5bdb9ac029c3

data:    ObjectType           : ServicePrincipal

data:    

info:    role assignment list command OK

### Verify your Service Principal

Your service principal is created as follows:

* **TENANT-ID**

* **CLIENT-ID**

* **CLIENT-SECRET**

Please verify it with the following steps:

1. Use Azure CLI to login with your service principal.

You can find the TENANT-ID, CLIENT-ID, and CLIENT-SECRET properties in the ~/bosh.yml file. If you cannot login, then the service principal is invalid.

azure login --username <CLIENT-ID> --password <CLIENT-SECRET> --service-principal --tenant <TENANT-ID>

Example:

azure login --username 246e4af7-75b5-494a-89b5-363addb9f0fa --password "password" --service-principal --tenant 22222222-1234-5678-1234-678912345678

2. Verify that the subscription which the service principal belongs to is the same subscription that is used to create your resource group. (This may happen when you have multiple subscriptions.)

3. Recreate a service principal on your tenant if the service principal is invalid.

**NOTE**: In some cases, if the service principal is invalid, then the deployment of BOSH will fail. Errors in ~/run.log will show get_token - http error like this [reported issue](https://github.com/cloudfoundry-incubator/bosh-azure-cpi-release/issues/49).

# Deploying the ARM Template

The Azure Resource Manager (ARM) template allows you to quickly deploy the necessary infrastructure for deploying Bosh and CloudFoundry. This template will create everything needed from scratch, so there is no need to have network, storage or VM's pre-configured before running this.

Click this link to deploy the template:

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_7.png)

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_8.png)

### Virtual Machine Name

The template will create a jump box on the Bosh network. In this example it will be called "devbox". The VM will have 1 core, 3.5GB RAM, and 50GB of SSD storage. This jumpbox will be used to deploy the Bosh Director and CloudFoundry. The jumpbox will have a public IP that can be used to SSH, and the template form allows the username and password to be specified.

### Admin Username

A username for the jump box VM.

### SSH Key Data

Provide a public key for the jump box VM. You should generate a private/public key pair, and keep the private key securely on your local machine. The parameter sshKeyData should be a string which starts with ssh-rsa.

Use ssh-keygen to create a 2048-bit RSA public and private key files, and unless you have a specific location or specific names for the files, accept the default location and name.

> ssh-keygen -t rsa -b 2048

### Tenant ID

Specify the tenant ID for the subscription to be used. This can be found from the Azure CLI.

**> azure account list**

info:    Executing command **account list**

data:    Name        Id                                    Current  State  

data:    ----------  ------------------------------------  -------  -------

data:    Free Trial  8a1f4887-57d2-4efa-a4d0-b3457ad46110  true     Enabled

info:    **account list** command **OK**

**> azure account show 8a1f4887-57d2-4efa-a4d0-b3457ad46110**

info:    Executing command **account show**

data:    Name                        : Free Trial

data:    ID                          : 8a1f4887-57d2-4efa-a4d0-b3457ad46110

data:    State                       : Enabled

data:    Tenant ID                   : 7d5503c5-af32-4c2f-bec4-a7bb3f3fa165

data:    Is Default                  : true

### Client ID

The ARM template needs Active Directory permissions to create cloud resources as an application with "contributor" privileges. This means that an application account must have been created in Active Directory in the previous step ([Creating a Service Principal](https://docs.google.com/document/d/1gq5wbVCuZI_RQUTXBC2wlfDpbyXfa2YUS4vJ4--b8ls/edit?ts=56b21914#heading=h.i25sqeulpjgs)). The password that was specified in that step will be the Client-Secret. The Client ID can be found from the Azure CLI.

**> azure ad app list**

info:    Executing command **ad app list**

+ Listing applications                                                         

data:    AppId:                   ec864f4a-ec9d-40ce-9f40-57b58f969aee

data:    ObjectId:                704ee950-90e5-4295-a501-5c7a75a3f596

data:    DisplayName:             pcf

data:    IdentifierUris:          0=http://BOSHAzureCPI

### Auto-deploy Bosh

This is optional. Leave it disabled if you want to see more detail about the install process, or enable it to hurry things along a little. If you decide to enable this field, please skip the next section on Deploying Bosh.

### Deploy the ARM Template

After filling out the information in the previous section, specify the correct subscription and resource group and click "Create". The process should take about 35 minutes.

Once the ARM template completes, the following resources will be seen in the "All Resources" panel:

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_9.png)

There are two subnets, Bosh and CloudFoundry, and the "devbox" vm is connected to the Bosh subnet. The VM's IP address is 10.0.0.100.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_10.jpg)

### Connect to the devbox VM

Selecting the devbox VM in the All Resources panel will show information about the VM including the Public IP that was assigned. In this case the public IP is 40.114.85.194.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_11.png)

SSH to the devbox VM using the username and your private SSH key.

> ssh -i ~/.ssh/id_rsa jgammon@40.114.85.194

The home directory contains several useful files.

> ls

bosh.key  bosh-state.json  bosh.yml  **deploy_bosh.sh**  **deploy_cloudfoundry.sh**  **example_manifests**  install.log  run.log  settings

<table>
  <tr>
    <td>File</td>
    <td>Description</td>
  </tr>
  <tr>
    <td>bosh.key</td>
    <td>Private key to connect to the Bosh Director once the VM is created.</td>
  </tr>
  <tr>
    <td>bosh.yml</td>
    <td>Bosh manifest. Check this file for accuracy before doing the deployment.</td>
  </tr>
  <tr>
    <td>deploy_bosh.sh</td>
    <td>Script for deploying the Bosh director VM.</td>
  </tr>
  <tr>
    <td>deploy_cloudfoundry.sh</td>
    <td>Script for deploying CF. It will upload the stemcell and release to the Bosh director, then deploy CF with the specified manifest.</td>
  </tr>
  <tr>
    <td>example_manifests</td>
    <td>CF manifests for single or multi-VM deployments. The Enterprise manifest deploys more instances for HA.</td>
  </tr>
  <tr>
    <td>install.log</td>
    <td>Logs the creation of the devbox vm. Check this file to make sure all of the packages are installed correctly. Should say "Finish" at the bottom.</td>
  </tr>
  <tr>
    <td>run.log</td>
    <td>Debug log for Bosh Director deployment.</td>
  </tr>
  <tr>
    <td>settings</td>
    <td>Useful information about the Azure environment.</td>
  </tr>
</table>


# Deploying Bosh

It is now time to use the devbox VM as a jumpbox to deploy the Bosh Director VM. The bosh.yml file is already built by the ARM template, so everything should be ready to go.

The bosh.yml file should be auto-generated correctly, but it is useful to check it before initiating the Bosh deploy. Confirm that the tenant-id, client-id, and client-secret are correct (see [Creating a Service Principal](#heading=h.i25sqeulpjgs)). The rest of the defaults should be good.

To start the deployment, execute the deploy_bosh script from the devbox.

> ./deploy_bosh.sh

The deployment will take about 40 minutes. Once the deployment finished, there will be a Bosh Director VM as shown below.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_12.png)

SSH from the devbox to the new Bosh Director VM to confirm everything is working.

> ssh -i ./bosh.key vcap@10.0.0.4

The authenticity of host '10.0.0.4 (10.0.0.4)' can't be established.

ECDSA key fingerprint is a5:2d:98:43:ac:70:37:48:4e:76:6b:85:c5:03:78:f4.

Are you sure you want to continue connecting (yes/no)? yes

Warning: Permanently added '10.0.0.4' (ECDSA) to the list of known hosts.

Ubuntu 14.04.3 LTS \n \l

Welcome to Ubuntu 14.04.3 LTS (GNU/Linux 3.19.0-31-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

To run a command as administrator (user "root"), use "sudo <command>".

See "man sudo_root" for details.

vcap@36849489-b8cd-4214-42f6-42c9296960ff:~$ sudo -i

root@36849489-b8cd-4214-42f6-42c9296960ff:~#

Here is the updated resources view after the Bosh Director is deployed. In my environment, the VM called braavos-368(..) is the new Bosh Director VM. Yours will be named according to your unique storage account name.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_13.png)

Here is the updated network diagram showing the new Bosh Director VM (braavos-368..).

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_14.jpg)

### Assign Bosh Director Public IP

The IP address of the Bosh Director is 10.0.0.4. It can optionally have a public IP assigned. The public IP was created by the ARM template, but it was not assigned to the Bosh Director VM. Click "All Settings" on the VM, then “IP Addresses”, and assign the “devbox-bosh” public IP as shown below. Be sure to click the “Save” icon circled in red below.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_15.png)

The Bosh Director VM should now be updated with the new public IP. SSH to the public IP to confirm everything is working.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_16.png)

### Connect to Bosh

The bosh CLI is already installed on the devbox. SSH to the devbox and run the following command:  (username/password is admin/admin)

> bosh target 10.0.0.4

Target set to `bosh'

> bosh login

Your username: admin

Enter password: 

Logged in as `admin'

Once the bosh target is configured, run the "bosh status" command. Make a note of the Bosh UUID.

> bosh status

Config

  /home/jgammon/.bosh_config

Director

  Name       bosh

  URL        https://10.0.0.4:25555

  Version    1.0000.0 (00000000)

  User       admin

  UUID       768a2a54-838e-4ca5-aa34-ba58e073672a

  CPI        cpi

..

..

# Deploy Pivotal CloudFoundry and Errands

The full deployment pipeline for PCF, errands and services is available at the following Github repsoitory: [https://github.com/pivotal-customer0/azure-bosh-cpi-pipeline](https://github.com/pivotal-customer0/azure-bosh-cpi-pipeline). The repository contains the manifests used in the pipeline as well as the Concourse manifest and scripts.

![image alt text]({{ site.url }}/public/xUtrEM6WO3aobUMV8wT0hw_img_17.png)

### Download Binaries from PivNet

The first step is to get the PCF binaries from PivNet. To download binaries from the command line you will need an authorization token and the URL of the binary:

1. Go to PivNet at [https://network.pivotal.io](https://network.pivotal.io)

2. Login with your account, click your username at the top of the page and select Edit Profile

3. To obtain your authorization token, go to the bottom of the page and copy the value of API TOKEN

4. Test your API token

curl -i -H "Accept: application/json" -H "Content-Type: application/json" -H "Authorization: Token YOUR_TOKEN_GOES_HERE" -X GET [https://network.pivotal.io/api/v2/authentication](https://network.pivotal.io/api/v2/authentication)

5. Go back to PivNet home page

6. Accept the EULA for cf 1.6.15 release:

curl -i -H "Accept: application/json" -H "Content-Type: application/json" -H "Authorization: Token YOUR_TOKEN_GOES_HERE" -X POST 

[https://network.pivotal.io/api/v2/products/elastic-runtime/releases/1485/eula_acceptance](https://network.pivotal.io/api/v2/products/elastic-runtime/releases/1485/eula_acceptance)

7. To obtain the URL of the binary, choose the product, **_cf 1.6.15_***,* select the file you want to download, click the i button and copy the API Download URL

***note - ensure the quotation marks are all consistent in the copied string - test in simple text editor**

8. Use wget to get the product. Download the following binaries from Pivnet:

    1. Elastic Runtime 1.6.15

wget -O "cf-1.6.15.pivotal" --post-data="" --header="Authorization: Token YOUR_TOKEN_GOES_HERE" https://network.pivotal.io/api/v2/products/elastic-runtime/releases/1485/product_files/3888/download

		

### Upload the Releases to Bosh

Next we need to unzip and upload the releases we downloaded from PivNet. Unzip the each of the .pivotal files downloaded i.e. cf-1.6.15.pivotal. 

Each .pivotal file unzipped has a /release folder. Change dir to each /release folder for each .pivotal unzipped. Use BOSH to upload the .tgz files.

unzip cf-1.6.15.pivotal

bosh upload release releases/cf-225.12.tgz

bosh upload release releases/cf-autoscaling-28.tgz

bosh upload release releases/cf-mysql-23.tgz

bosh upload release releases/diego-0.1446.1.tgz

bosh upload release releases/etcd-release-18.tgz

bosh upload release releases/garden-linux-0.330.0.tgz

bosh upload release releases/push-apps-manager-release-397.tgz

bosh upload release releases/notifications-19.tgz

bosh upload release releases/notifications-ui-10.tgz

### Configure a PCF Manifest

Now we need to create the deployment manifest for Azure. Fortunately we have a handy tool at our disposal to do so.  The converter-azure, which is a Ruby tool, will take as input the vSphere manifest and automatically convert it (given some parameters) to a format that BOSH and the Azure CPI can understand.

Please follow the steps below to get the Azure manifest:

1. Clone the converter-azure tool from git

> cd ~

> git clone [https://github.com/pivotal-customer0/converter-azure.git](https://github.com/pivotal-customer0/converter-azure.git)

> cf converter-azure

> bundle install

>

NOTE: If you encounter an error similar to the following, 

"Bundler could not find compatible versions for gem "bundler"

Just reinstall the Ruby gem manager by running the following command (assuming you are on a Linux based platform):

> gem install bundler

2. Use the converter

	> bundle exec converter --name cf \                      --swap-ip-ranges IP-RANGE-TO-SWAP \                      --domain YOUR-DOMAIN \                      --sanitize-partition \                      --director-uuid YOUR-BOSH-DIRECTOR-UUID \                      --manifest VSPHERE-MANIFEST-FILE \                      --dns DNS-IP \                      --stemcell_version STEMCELL-VERSION \                      --network-name BOSH-SUBNET:CLOUDFOUNDRY-SUBNET \                      --output WHERE-TO-PUT-THE-CONVERTED-MANIFEST \                      --public_ip INSTALLATION-PUBLIC-IP

Replace the values of the parameters as follow 

<table>
  <tr>
    <td>Parameter</td>
    <td>Description</td>
  </tr>
  <tr>
    <td>name</td>
    <td>Release name</td>
  </tr>
  <tr>
    <td>swap-ip-ranges</td>
    <td>Replaces ips on the left with ips on the right and will accept a comma separated list of ranges to swap.</td>
  </tr>
  <tr>
    <td>domain</td>
    <td>Replace the domain in the manifest</td>
  </tr>
  <tr>
    <td>sanitize-partition</td>
    <td>When specify, the converter will remove the UUID from the Job name during the conversion.</td>
  </tr>
  <tr>
    <td>director-uuid</td>
    <td>The BOSH director UUID, This information can be found in the bosh.yml file or by simply running bosh status from inside the jumpbox.</td>
  </tr>
  <tr>
    <td>manifest</td>
    <td>The vSphere manifest that will be used as source of conversion.</td>
  </tr>
  <tr>
    <td>dns</td>
    <td>The DNS IP to set in the manifest file.</td>
  </tr>
  <tr>
    <td>stemcell_version</td>
    <td>The stemcell version to use, default to 3163</td>
  </tr>
  <tr>
    <td>network-name</td>
    <td>The BOSH and CloudFoundry subnets, comma separated</td>
  </tr>
  <tr>
    <td>output</td>
    <td>A path to a YAML file where you want to save the converted manifest.</td>
  </tr>
  <tr>
    <td>public_ip</td>
    <td>Ha Proxy IP</td>
  </tr>
</table>


Example of usage:

> bundle exec converter --name cf \                      --swap-ip-ranges 192.168.200:10.0.16 \                      --domain azure.pcflab.net \                      --sanitize-partition \                      --director-uuid YOUR-BOSH-DIRECTOR-UUID \                      --manifest ../manifest/cf-75e54624c449084052b6.yml \                      --dns 8.8.8.8 \                      --stemcell_version 3196 \                      --network-name boshvnet:cfsubnet \                      --output ../azure/cf.yml \                      --public_ip 40.76.198.83

 

3. Generate tile manifest

If you are planning to install some of our services (MySQL, SCS, etc.) , the converter can also be leverage. All you have to do is repeat the step b. using the product manifest file generated by vSphere. Be sure to change the source file name each time! Be sure to change the destination file each time!Samples of converted YMLs for Azure deployment

* [PCF](https://github.com/pivotal-customer0/azure-bosh-cpi-pipeline/blob/master/cf.yml)

* [MySQL](https://github.com/pivotal-customer0/azure-bosh-cpi-pipeline/blob/master/mysql.yml)

* [RabbitMQ](https://github.com/pivotal-customer0/azure-bosh-cpi-pipeline/blob/master/rabbitmq.yml)

* [Redis](https://github.com/pivotal-customer0/azure-bosh-cpi-pipeline/blob/master/redis.yml)

* [Spring Cloud Services](https://github.com/pivotal-customer0/azure-bosh-cpi-pipeline/blob/master/spring-cloud.yml)

**NOTE**: At the moment, the tool does not generate certificate for you, you will have to  take care of that part yourself. 

### Deploy PCF and Errands

Use the *bosh stemcells* and *bosh releases *CLI commands from the devbox to confirm that the stemcells and CF releases have been uploaded correctly. The last step is to simply do the deploy.

> bosh target <bosh-director-ip>

> bosh deployment YOUR-MANIFEST-FOR-PIVOTAL-CLOUD-FOUNDRY

> bosh deploy

The deployment process should take about 2-4 hours. Good luck! Once CloudFoundry finishes deploying, you can run the errands.

> bosh -n run errand smoke-tests

> bosh -n run errand autoscaling

> bosh -n run errand autoscaling-register-broker

> bosh -n run errand notifications

> bosh -n run errand notifications-ui

> bosh -n run errand push-apps-manager

> bosh -n run errand push-app-usage-service

### Connect to PCF

You can CF login with username/password of admin/password:

> cf login -a https://api.SYSTEM-DOMAIN --skip-ssl-validation

# Deploy MySQL

Similar to deploying PCF above, we need to **modify the manifest**,** **upload our releases, run the deployment and associated errands. For more information about getting the authorization token, see the previous section about deploying Pivotal CloudFoundry.

curl -i -H "Accept: application/json" -H "Content-Type: application/json" -H "Authorization: Token YOUR_TOKEN_GOES_HERE" -X POST [https://network.pivotal.io/api/v2/products/p-mysql/releases/1327/eula_acceptance](https://network.pivotal.io/api/v2/products/p-mysql/releases/1327/eula_acceptance)

wget -O "p-mysql-1.7.2.pivotal"  --post-data="" --header="Authorization: Token YOUR_TOKEN_GOES_HERE" [https://network.pivotal.io/api/v2/products/p-mysql/releases/1327/product_files/3630/download](https://network.pivotal.io/api/v2/products/p-mysql/releases/1327/product_files/3630/download)

> unzip p-mysql-1.7.2.pivotal

> bosh target <bosh-director-ip>

> bosh upload release releases/cf-mysql-24.tgz

> bosh upload release releases/mysql-backup-1.tgz

> bosh upload release releases/service-backup-1.tgz

> bosh deployment p-mysql-1.7.2.yml

> bosh deploy

> bosh run errand broker-registrar

> bosh -n deployment mysql.yml

> bosh -n deploy

> bosh -n run errand broker-registrar

> bosh -n run errand acceptance-tests

# Deploy Redis

> bosh target <bosh-director-ip>

> bosh -n deployment redis.yml

> bosh -n deploy

> bosh -n run errand broker-registrar

> bosh -n run errand smoke-tests

# Deploy RabbitMQ

> bosh target <bosh-director-ip>

> bosh -n deployment rabbitmq.yml

> bosh -n deploy

> bosh -n run errand broker-registrar

# Deploy Spring Cloud Services

> bosh target <bosh-director-ip>

> bosh -n deployment spring-cloud.yml

> bosh -n deploy

> bosh -n run errand deploy-service-broker

> bosh -n run errand register-service-broker

