---
title: "Azure - Create a Custom RBAC Role to allow Stop - Start of all Azure Virtual Machines in a Subscription with Terraform"
author: Nathanael Santschi
date: 2022-09-08T10:51:21+01:00
draft: false
tags:
  - terraform
  - Azure
  - RBAC
categories:
  - Azure
  
---

Yes... its annoying there is **no built in role** to only allow restarting of Azure Virtual Machines...
The **Virtual Machine Contributor Role** allows to much. With this role you are able to destory and create VMs..

So what I want to do in this case is creating a **custom role which only allows to start / stop / restart Virtual Machines.** 
And I want to do that with **terraform** because I'm doing the whole Azure Resource Deployment with terraform anyway. 

## Prerequisites
- You need to have your basic terraform config ready (provider setup etc.)


## Creating the Role Definition
First we need to create the **Role Definition**. Because I want to create the role on subscription level, I first need to get id from the subscription. 
So what I do is creating a terraform **data source** with the subscription id.
After that I can create the **role definitions** with the permissions which I need and assign that role to the scope of the subscription: 


````
data "azurerm_subscription" "primary" {
    provider     = azurerm.platform
    subscription_id  = "346b5d52-b6b0-478c-87d4-b0c6f75adae2"

}

resource "azurerm_role_definition" "VM_Operator" {
  provider     = azurerm.platform
  name        = "Virtual Machine Operator"
  scope       = data.azurerm_subscription.primary.id
  description = "Start / Stop virtual machines"

  permissions {
    actions     = [
        "Microsoft.Compute/*/read",
        "Microsoft.Compute/virtualMachines/start/action",
        "Microsoft.Compute/virtualMachines/restart/action",
        "Microsoft.Compute/virtualMachines/deallocate/action"
    ]
    not_actions = []
  }

  assignable_scopes = [
    data.azurerm_subscription.primary.id
  ]
}
````


## Creating the Role Assignment
To assign our newly created role, we need a **Azure AD Group** to which we can assign it. 
You can either create a Azure AD Group with terraform using the Azure AD provider or you can retrieve the data of an existing azure AD Group with the terraform data source. 

In this example I create first a **group**: 

```
resource "azuread_group" "sub_vm_operator" {
  provider         = azuread.ad
  security_enabled = true
  display_name     = "my-amazing-vm-operator-group"
}
```

and in a second step I create the **role assignment**:

```
resource "azurerm_role_assignment" "sub_vm_operator" {
  provider           = azurerm.platform
  scope              = data.azurerm_subscription.primary.id
  role_definition_id = azurerm_role_definition.vm_operator.role_definition_resource_id
  principal_id       = azuread_group.sub_vm_operator.id
}
```

Now I can browse my **Subscription** in the **Azure Portal** and on **Access control (IAM) -> Role assignments** I can find my new Role **"Virtual Machine Operator"** assigned to my specific group. 


So that's it. The users in this group are now able to start / stop / restart all Virtual machines in this subscription. 

## Links
https://techcommunity.microsoft.com/t5/itops-talk-blog/step-by-step-enabling-custom-role-based-access-control-in-azure/ba-p/363668
https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/role_assignment
