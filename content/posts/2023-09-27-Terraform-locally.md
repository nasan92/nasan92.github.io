---
title: "(2/2) How to use terraform from local Machine additionally to Azure DevOps Pipeline (2 Part Series)"
author: Nathanael Santschi
date: 2023-09-26T06:10:21+01:00
draft: false
tags:
  - terraform
  - AzureDevOps
  - Microsoft
  - WorkloadIdentityFederation
  - MicrosoftEntra
  - InfrastructureAsCode
categories:
  - Azure
  - AzureDevOps
  
---
In the latest blog post [(1/2) Setting up Azure workload identity federation with Terraform in Azure DevOps pipelines (2 Part Series)](https://nasan.ch/posts/2023-09-23-azuredevopsterraform/) we learned how to setup Azure DevOps using Workload Identiy Federation

Because we are using a managed identity and not a service principal with a secret that has a certain lifetime, we are not directly able to run terraform from locally. 

But what about using a service principal for local activities whose secret will expire after a few hours instead of months?  
This wasn't practical when using a service principal in the service connection. Typically, the secret had a longer lifetime, so it didn't need to be renewed every time within the service connection.

And there is another challenge. What about the provider? I don't want to change the provider locally everytime?.  

My Workaround to use terraform locally:

## Prerequisites
- You worked through [Series 1](https://nasan.ch/posts/2023-09-23-azuredevopsterraform/)
- [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) installed 
- [Git](https://git-scm.com/downloads) installed 
- [Microsoft Graph Powershell Module](https://learn.microsoft.com/en-us/powershell/microsoftgraph/installation?view=graph-powershell-1.0) installed
- [Azure Powershell Module](https://learn.microsoft.com/en-us/powershell/azure/) installed

## Overview - Initial Setup Steps
1. Create an app and a service principal, and add contributor permissions to the subscription (this service principal will be used to connect from locally)
2. Create a .gitignore file (to ignore secrets and override file)
3. Create secrets file (here we will store temporarily valid secrets)
4. Define variables
5. Create a provider override file
6. Add helper scripts 


![image](/images/terraform-overview-Demo-tf-local.png "Preview")

The directory structure will look like this:
````
Terraform-Demo
└── Helper-Scripts  
    ├── update-secrets.ps1  
└── terraform  
    ├── 00_provider_override.tf
    ├── 00_provider.tf  
    ├── 01_mainexample.tf
    ├── 02_variables.tf
    ├── secrets.auto.tfvars 
└── .gitignore 
└── azure-pipeline.yaml  
````

## Initial steps
The following steps need to be prepared only initialy. 

### 1. Create an app and service principal, and add contributor permissions to the subscription
- initial creation of an app registration and service principal with permission to specific resources in my example the subscription.

````powershell
$AppName = 'Terraform-Demo-2609'
$subscriptionName = "SUB-Terraformdemo-01"

# connect with mgraph and azaccount
Connect-MgGraph -Scopes 'Application.ReadWrite.All'
Connect-AzAccount

# Set Azure Subscription
Set-AzContext -SubscriptionName $subscriptionName

# Create new App
$App = New-MgApplication -DisplayName $AppName -SignInAudience 'AzureADMyOrg'
$AppId = $App.AppId

# Create a new service principal object
$ServicePrincipalID=@{
    "AppId" = $AppId
    }
$ServicePrincipal =  New-MgServicePrincipal -BodyParameter $ServicePrincipalId 

# Assign permission to the service principal
$SubscriptionId = (get-azsubscription -SubscriptionName $subscriptionName).Id

New-AzRoleAssignment -RoleDefinitionName Contributor -Scope "/subscriptions/$SubscriptionId" -ObjectId $ServicePrincipal.Id
`
````


### 2. Create a .gitignore file 
Let's create a gitignore file. For our example are two ignorations specifically important:
- secrets.auto.tfvars
- override.tf


The secrets file will include the client id and secret of our app registration. We will have one override file which basically will override the provider file from the repo with details to the backend storage account, where the terraform state file is stored.   
So we don't want to have those files in the repository. 

.gitignore:
```git
`# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Secrests
secrets.auto.tfvars

# Crash log files
crash.log

# Exclude all .tfvars files, which are likely to contain sentitive data, such as
# password, private keys, and other secrets. These should not be part of version 
# control as they are data points which are potentially sensitive and subject 
# to change depending on the environment.
#
#*.tfvars

# Ignore override files as they are usually used to override resources locally and so
# are not checked in
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Include override files you do wish to add to version control using negated pattern
#
# !example_override.tf

# Include tfplan files to ignore the plan output of command: terraform plan -out=tfplan
# example: *tfplan*

# Ignore CLI configuration files
.terraformrc
terraform.rc
```

### 3. Create secrets file 
Within the terraform folder create a new file secrets.auto.tfvars with the following content: 

```terraform
client_id          = "addheretheclientidofyourapp"
client_secret      = "leavethatemptynow"

azure_tenant_id = "add-here-the-customer-tenant-id"
platform_subscription_id = "add-here-subscription-id"
```

### 4. Define variables
We'll later use some variables so create a new file 02_variables.tf with the following variables: 
````terraform
variable "client_id" {
  type    = string
  default = "none"
}
variable "client_secret" {
  type    = string
  default = "none"
}
variable "azure_tenant_id" {
  type    = string
  default = "none"
}
variable "platform_subscription_id" {
  type    = string
  default = "none"
}
````



### 5. Create a provider override file
I have this idea from here: [run terraform locally and in pipeline](https://bzzzt.io/post/2021-10/2021-10-04-run-tf-locally-and-in-pipeline/)

simply copy the 00_provider.tf file and add override at the end of the file name
In this file we need now to add the backend storage account with all necessary information. 

```terraform
`terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-backend-demo"
    storage_account_name = "tfdemo23092314462014"
    container_name       = "terraform01"
    key                  = "terraform.tfstate"
    access_key           = "addaccesskeyhere"
  }

  required_version = "1.4.0" // remember to update azure-pipelines.yml file too (when updating version)
}

provider "azurerm" {
  alias           = "platform"
  tenant_id       = var.azure_tenant_id
  subscription_id = var.platform_subscription_id
  client_id       = var.client_id
  client_secret   = var.client_secret

}
```


### 6. Add helper scripts 

create a new directory "Helper-Scripts" for example and add the following script

````powershell
# Retrieve Client ID from secrets file
#-------------------------------------------------------------------
# Define the path to the secrets.tf file
$secretsFilePath = ".\terraform\secrets.auto.tfvars"

# Read the contents of the file
$fileContent = Get-Content -Path $secretsFilePath -Raw

# Define a regular expression pattern to match the client_id value
$pattern = 'client_id\s*=\s*"([^"]+)"'

# Use the Select-String cmdlet to find the pattern in the file content
$match = $fileContent | Select-String -Pattern $pattern

# Check if a match was found
if ($match) {
    # Extract the captured value (client_id) from the match
    $client_id = $match.Matches.Groups[1].Value

    # Print the client_id value
    Write-Host "client_id value: $client_id"
} else {
    Write-Host "client_id not found in $secretsFilePath"
}


# Create a new secret
#-------------------------------------------------------------------
Write-host "Login with a user which has permissions to read/write applications"
Connect-MgGraph -Scopes 'Application.ReadWrite.All'



$appObjectId = (Get-MgApplication -Filter "AppId eq '$client_id'").Id 

$expirationDate = (Get-Date).AddHours(8)
$credentialName = "TFTempLocal_EndDate_$($expirationDate.ToString('yyyy-MM-dd_HH-mm'))"
$passwordCred = @{
    displayName = $credentialName
    endDateTime = $expirationDate
}

$secret = Add-MgApplicationPassword -applicationId $appObjectId -PasswordCredential $passwordCred
$secret | Format-List
$secretText = ($secret).SecretText

# Save new secret into secret file
#-------------------------------------------------------------------


# Define the new client_secret value with the $secret variable
$newClientSecret = 'client_secret = "' + $secretText + '"'

# Use regular expressions to find and replace the client_secret line
$fileContent = $fileContent -replace 'client_secret\s*=\s*"[^"]*"', $newClientSecret

# Write the updated content back to the file
$fileContent | Set-Content -Path $secretsFilePath

# Display a success message
Write-Host "Updated $secretsFilePath with the new client_secret value."
``
````
## Recurring steps
Now you prepared so far everything to be able to use terrafrom from you local machine. 
Now everytime you need to use terraform from your local machine you can run the following steps:

### 1. Create new secrets 

run the script:
**.\Helper-Scripts\update-secrets.ps1**

this will force you to login with an account which has enough permissions to create new secrets on an application and then it will create a new secret.

The script stores the secret directly in the secrets.auto.tfvars file.

I hardcoded an expirationtime of 8 hours - so you can use terraform for 8 hours locally.

![image](/images/update-secrets.png "Preview")

![image](/images/update-secrets2.png "Preview")

### 2. Go and use terraform locally
![image](/images/Pastedimage20230926174850.png "Preview")

![image](/images/terraform-plan.png "Preview")

Using Terraform locally is especially useful for troubleshooting or if you want to manipulate the state.

## Conclusion
With this solution, we still need a service principal with permissions on the subscription, which is in addition to the managed identity we are using within the pipeline.    
However, we don't have any long-valid secret that could be misused. The only place where your temporarily valid secret is stored is on your local machine.  
So, we can use the same repository locally and in the Azure DevOps Pipeline without manually changing the provider every time