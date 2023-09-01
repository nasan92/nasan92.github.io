---
title: "Securing Resources in Azure VMs with Microsoft Entra Private Access in a Hub-and-Spoke Architecture"
author: Nathanael Santschi
date: 2023-08-31T06:10:21+01:00
draft: false
tags:
  - GlobalAccess
  - PrivateAccess
  - Microsoft
  - Entra
  - MicrosoftEntra
  - VPNAlternative
  - HubSpoke
categories:
  - Azure
  - MicrosoftEntraPrivateAccess
  
---

I did a little **Microsoft Entra Private Access** Test setup.  
My goal was to test access to some **private Resources** hosted on **Azure Virtual Machines** with **Microsoft Entra Private Access** instead of VPN. 

## Overview
The test setup is illustrated below:
![Overview Test Setup:](/images/MicrosoftEntraPrivateAccess-AzurVM.drawio.png "Preview")


I have one Virtual Machine (VM) with a **Windows File Share** that I wish to access from my endpoint and I also want to be able to access this VM via **RDP.**  
Additionaly in another Spoke VNET I have a simple **Web Server** which I also would like to access via **Private Access**. 

Following the Microsoft Documentation: [How to get started with global secure access](https://learn.microsoft.com/en-us/azure/global-secure-access/how-to-get-started-with-global-secure-access#microsoft-entra-private-access)

There are **four main points** which needs be configured from **Microsoft Entra Private Access** side: 
1. The **App Proxy connector** and connector group
2. **Quick Access** for private resources OR a **private Global Secure Access Application**
3. Private Access **traffic forwanding profile**
4. **Global Secure Access Client** on end-user devices


## 1. Configure App Proxy connector
Read the docs for more details: [How to configure App Proxy connectors for Microsoft Entra Private Access](https://learn.microsoft.com/en-us/azure/global-secure-access/how-to-configure-connectors)
### Prepare new Windows Azure VM
I prepared the following:
- New **Windows Server VM** on Azure in a new **Hub VNET**

### Download the Connector

After successfully creating the VM I downloaded the **Connector service** from the **Entra Admin Center**:

![Download connector service:](/images/download-connector-service.png "Preview")

### Install the Connector
Connect to the new created VM and install the **connector:**
![Installer-connector:](/images/installer-connector.png "Preview")

start the installation
![Install-connector-start:](/images/start-appproxy-install.png "Preview")

during the installation a login with your **global admin** is required.
![Install-connector-start:](/images/setup-successfull-appproxy.png "Preview")

### Verify Connector Status in the Entra Portal

![connector status](/images/connector-status.png "Preview")

## Prepare Spoke Vnet 1 and Windows Server File Web Server
Now the appproxy VM is prepared and I'm gonna create two further VMs in two different VNETs which both will serve as separate spokes. 
In order to be able to work as spoke the VNETs will be peered to the Hub VNET.

### Create new VM with a Web Server
create a new vm:
![New VM](/images/New-webservervm.png "Preview")

this VM will be placed into a new Spoke VNET:
![Vnet-webserver](/images/webserver-vnet.png "Preview")

### Create VNET Peering
After successfully creating the VM and VNET, I established peering between the newly created spoke VNET and the hub VNET.

![Vnet-peering](/images/VNET-peering.png "Preview")

### Install IIS on the Web Server
After successful peering of the VNETs, I installed **IIS** on the new VM: 

````powershell
Install-WindowsFeature -name Web-Server -IncludeManagementTools
````

and verified that the default site was accessible locally.
![iis-default](/images/iis-default.png "Preview")
 
### Prepare Windows Firewall on IIS
In order to be able to access this site from external I created inbound **Windows Firewall Rules** to allow Ports **80** and **443**: 
![iis-default](/images/windows-firewall-443-80.png "Preview")

### Prepare NSG Rules
And I created as well a rule on a **NSG** which is assigned to the Web Server to allow port 80 and 443 from the App Proxy VM: 

![nsg-rule-webserver](/images/nsg-rule-appproxy-webserver.png "Preview")

## 2. Configure per app Access
Note: There are two Options to configure Access to resources: 
1. per App Access
2. Quick Access

Further information [here](https://learn.microsoft.com/en-us/azure/global-secure-access/concept-private-access)

In this first example I tried the per App Access.
To configure that I created a new application: 
![per-app-access-app](/images/new-per-app-access-application.png "Preview")

and added a application segment which includes the IP of the Web Server VM and the necessary ports: 
![create-application-segment](/images/create-application-segment.png "Preview")

and now I need to configure which users or groups can access this application. For now I just added myself: 
![app-users-groups](/images/enterprise-app-users.png)


## 3. Enable Private Access traffic forwarding profile 
Enable the traffic forwarding profile for Private Access:  
![Traffic-Forwarding](/images/Traffic-forwarding.png)

and now I'm able to link Conditional Access Policies to my application: 
![ConditionalAccessPolicy-Privateapp](/images/CA-PrivateAccess.png)

## 4. Install Global Secure Access Client
And now as a last step the installation of the Global Secure Access Client is necessary which is pretty straightforward:  
[The Global Secure Access Client for Windows - Installation](https://learn.microsoft.com/en-us/azure/global-secure-access/how-to-install-windows-client)

![client-download](/images/client-download-screen.png)

## Test Access to Website
Now my **Global Secure Access Client** is running an I can test to access my default **IIS Website:**  
![IIS Website](/images/IIS-website.png)

wohoo it's working.

## Prepare Spoke Vnet 2 and Windows Server with a File Share / RDP
Here I repeated the following steps for a second test:
- created a second **Windows Server** as Azure VM
- Created a second **Spoke VNET**
- **VNET peering** between Spoke 2 VNET and Hub VNET
- Inbound **Windows Firewall Rules** for **445 (SMB)** and **3389 (RDP)**
- Inbound **NSG Rules** from **AppProxy** VM to **Fileserver VM**


## Configure Quick Access
This time instead of configuring a **per app access** (which didn't made until now much sense because I had just one app) I created a **Quick Access**:  
![Quick Access](/images/QuickAccessConfiguration.png)  

Also for the **Quick Access** it is necessary to add users and groups should be able to use it:  
![QuickAccess-Users](/images/QuickAccess-Users.png)

Now I can already test **Quick Access** because:
- **Traffic forwarding profile** is enabled -> I could only create an additional Conditional Access Policy for Quick Access
- **Global Secure Access Client** is installed

## Test Access via RDP and SMB
I tested the **RDP** connection to the private IP of the VM:
![RDP-Testconnection](/images/RDP-Connection-Test.png)  

The login window appeared, which is a good sign, and I was able to successfully log in. 
After logging in, I created a test share and verified access from my endpoint.  
![Test Share](/images/test-share-creation.png)

and finally test if I'm able to access this share from my endpoint.   
After entering credentials of a user account which has perrmission to the previously created share. I'm able to access it from my local endpoint:   
![ShareTest](/images/ShareTest.png)

## Conclusion
Private Access offers a secure VPN alternative, allowing access to specific resources without the need to connect the entire network. However, it requires running a VM with App Proxy installed, and for high availability, additional Connectors may be needed.