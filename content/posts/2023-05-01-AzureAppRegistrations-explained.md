---
title: "AzureAD App Registrations explained"
author: Nathanael Santschi
date: 2023-05-01T05:51:21+01:00
draft: false
tags:
  - MindMap
  - Microsoft
  - AppRegistration
categories:
  - Azure
  - Security
  
---
# Azure AD App registration
Recently I had some talks with developers which made me to realize that I didn't fully understand how **App Registrations** are working. 
I was aware that we are registering an app and allowing the app certain permissions but in detail I didn't understand it. 

## Why Azure AD App registration?
Basically for every app where you want to use the Microsoft Identity Platform, you need to register your app. 
So you want to login into a certain webapp with your Microsoft Account? This app needs to be registered in Azure AD. After you logged in into your application, this app maybe also need some data of your Microsoft Account and you maybe need to grant perrmissions.

There are two main concepts behind that:

### 1. Authentication
- The process of proving who you say you are. 
- Verification of the identity of a person or device. 
- short: **AuthN** 
- used protocol:
    -  [OpenID Connect](https://openid.net/connect/)
        - commonly cloud apps
    - SAML
        - Enterprise apps - with ADFS
  
### 2. Authorization
- Granting an authenticated party permission to do something. 
- short: **AuthZ**
- used protocol: [OAuth 2.0](https://oauth.net/2/)


## Basics of OpenID Connect (OIDC) - OAUTH2
To understand how App registrations are working I started with this video of John: [John Savill - Azure AD App Registrations, Enterprise Apps and Service Principals](https://www.youtube.com/watch?v=WVNvoiA_ktw&ab_channel=JohnSavill%27sTechnicalTraining)  
He first started to explain th basics of **OpenID Connect** and **OAUTH2**.  

From this video I created this picture to understand how OpenID Connect - Oauth2 is working: 
![OIDC-OAUTH](/images/OAUTH2-AzureADAppRegistration.png "Preview")

Watch the Video of John to get a better idea how this is working. 

## Register an App
So now we switch to **Azure AD**. A developer has for example his **HikingApp** where he would like to use **Azure AD** as **Identity Provider**. The **HikingApp** will be running as **SaaS** in his own tenant. So the developer now needs to **register his app** in his Azure Tenant:
![Register an App](/images/AzureAppRegistratin-registerintenant.png "Preview")

1. Developer wants to register his app in his Tenant
2. He creates an **App Registration** which creates a globally unique **Application Object**
3. During the creation of the App Registration it's necessary to select:
   1. **supported Account types**
   2. **Redirect URI**
      1. (place where authorization server redirects the user)
4. The App Registration process in the portal automatically creates a **Service Principal** (Listed in Enterprise Apps)

## What happens when a user is opening this HikingApp for the first time?
![User opening App](/images/AzureAppRegistration-firstopening.png "Preview")

1. User 1 navigates to the **HikingApp.com** Webpage and signs in with his **Microsoft Account**
   1. At the first login attempt the users needs to **consent permissions**
2. This will create a **Service Principal** in the user tenant
   1. which references to the **globally unique** **Application Object** 

## Authentication Concepts - MindMap
This MindMap tries to give an Overview over some core Authentication Concepts related to Azure AD: 

![Authentication Concepts - MindMap](/images/Authentication-Concepts.svg "Preview")