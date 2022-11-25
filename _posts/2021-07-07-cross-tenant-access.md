---
layout: post
current: post
navigation: True
title: Cross tenant access to Azure resources
date: 2021-07-07 07:00:00
class: post-template
subclass: 'post'
author: uday
---
The Azure AD application model provides a great way for someone to build an app running in their developer tenant while allowing their customers to use that app in the customer's tenant. Most examples of this pattern showcase how an admin can consent the app in their tenant and get access to Microsoft Graph data in that tenant. This also happens to be a great pattern to also access Azure resources in the other tenant. For example, your customer may ask you to encrypt their content using a key in their keyvault. This blog post gives a walkthrough on accessing Azure resources across tenant boundaries.

## Introduction

A multi-tenant application in Azure Active Directory provides a convenient way for developers to serve multiple customers without having to run a dedicated instance for each customer. It allows developers to access the data unique to each customer, once the customer consents to that application in their tenant.

In a typical multi-tenant application, the user interaction provides all the context your application needs to determine which tenant and user information to access for that interaction. This is called "delegated access" since the application accesses the data on behalf of the signed-in user. The application doesn't even need to know which tenant it is accessing, since the signed-in user who is interacting with the application belongs to a tenant and that implies which tenant is being accessed on behalf of the user. 

Sometimes, developer scenarios require their application to run in a "daemon" mode, without a signed-in user. In this non-interactive style, there is no signed-in user interaction that provides the tenant context just in time. We have to separately provision all the tenants our application has to access and process. We will consider this "daemon" pattern in this blog post, and see how we can build a daemon app that accesses Azure resources in other tenants.

## How the multi-tenant app works
To ground ourselves, let's briefly look at how a multi-tenant app works for the common scenario where there is user interaction.

First, you register the application in the developer's tenant and configure what permissions are needed by the application. In most cases, this is data in Microsoft Graph, and some example permissions are User.Read.All or Group.Read.All.

Then, the developer codes the application to access the data. The developer is not required to know which tenant to access ahead of time. This can be figured out dynamically based on the user is interacting with the application. For this "multi-tenant" model, the developer uses the common endpoint in Azure AD ((https://login.microsoftonline.com/common) for authentication. When a user signs in to the application at this endpoint, their sign-in information is used by Azure AD to build the tenant context for that request.

When an administration in a tenant adds this application to their tenant, they can consent to allow the application to access data in that tenant. When a user in that tenant accesses the application, the authentication context contains the user and tenant information. With this context, the application can call Microsoft Graph to access data on behalf of the user.

## Challenges we need to overcome for "daemon" applications

In order to extend this model to "deamon" applications, and access Azure applications, we have to consider the following:

1. In the lack of user interaction, how do we figure out the tenant context? We have to somehow tell the application all the tenants to process. Ideally, Azure AD would tell us all the tenants where this application exists. Unfortunately, Azure AD does not have that capability. So we can either statically configure the tenant list in our application, or find a way to dynamically update it periodically by storing it in a database.

2. The user's authentication when accessing the application served the purpose of authenticating both the user and the application. In the absence of the user, we need to add credentials to the application in the developer's tenant for it to authenticate.

3. The admin's authentication in the customer tenant served the purpose of authenticating the admin and providing an experience to consent to the application being added to that tenant. In the absence of user interaction, we need to find a way for the tenant admin to manually add this application to their tenant

## Building a multi-tenant "deaemon" app, accessing Graph or Azure resources

### Step 1: Create an application, with credentials.

In the developer's tenant, create an Azure AD application registration. Since this will be running without any user interaction, we need to allow it to authenticate by itself. This requires us to configure it with credentials it can use to authenticate. 

1. Create your multi-tenant app in the developer tenant. Go to the Azure AD portal, click "App registrations", and click on "+ New registration".  Give a name for your application, and then pick the "Accounts in any organizational directory (Any Azure AD directory - Multitenant)" option. Since this is an app without any user interaction, we don't need to configure the redirect URI.

2. Clicking register creates the application registration and puts you on a page where you can configure the application. Make a note of the Application (client) ID on this page, you will need it later. Go to Certificates & secrets, and add a new client secret. Save this securely in a key vault in the developer's tenant.


### Step 2: Configure Microsoft Graph permissions needed by the application
This step is only needed if the application needs to access Microsoft Graph in another tenant. If there is no such need, skip this step

1. On the same page where you landed after registering the application, click on "API permissions". Pick "Application permissions" in "What type of permissions does your application require". And then Pick the permissions you want to add (eg: user.read.all).

![Adding permissions to an application](/images/apppermission.png)


### Step 3: Add the created application in other tenants where it is needed. 

This has to be done by an administrator of each of the tenants. The Azure CLI provides a simple way to do this.

1. az login --tenant {tenantid} --allow-no-subscriptions

2. az ad sp create --id {the application id provided by the developer from Step 1}

### Step 4. Consent to the application, to grant it permissions to Microsoft Graph

This is only if you have configured the application with permissions needed in Step 2, otherwise, skip this step.

1. Go to the Azure AD portal in the target tenant, and go to Enterprise Applications.

2. Pick Application Type "All Applications". Enter the application id of the application in the search bar. Click Apply to find the application that was added in Step 3.

3. Go to the application, and click on "Permissions". Click on "Grant admin consent for {tenant}". Make sure you sign in as an admin with permission to grant consent and click Accept in the consent screen

![Consent screen](/images/consent.png)


### Step 5: Accessing Azure resources. 
This step needs to be done in the customer's tenant, by the owner of the Azure resource.

1. Go to the Azure portal in the target tenant. Go to the Azure resource which should be made available to the application. In the IAM policy for that resource, grant the service principal representing the application the appropriate role to access this resource.
![Adding an access policy to keyvault](/images/keyvaultpolicy.png)


### Step 6: Code and deploy your application
At this point, you have completed all the necessary configurations needed for the application to access the resources across tenants. 

Now when the application in the developer's tenant will be able to get a token for the Azure resource in the customer tenant and access it. The application needs to first authenticate itself. You can do this at the customer's tenant endpoint (https://login.microsoftonline.com/{tenantId}) using the OAuth Client Credentials grant flow. Both MSAL and Azure/identity SDK have support for this flow. 

In the developer's tenant, you need to keep a list of all the tenant-ids the application needs to process, along with all the Azure resources specific to each tenant. This information will help us process a tenant at regular intervals, signing into that tenant as the application, and accessing resources in that tenant.

To see a very rudimentary example that illustrates the code in Node, see [this github repo](https://github.com/udayxhegde/multitenant-daemonapp-node). It shows how you can achieve this for both Microsoft Graph and Azure KeyVault.


Comments/feedback/questions? [DM me on twitter](https://twitter.com/messages/compose?recipient_id=1446741344)