---
layout: post
current: post
navigation: True
title: Azure AD workload identity federation with Google Cloud
date: 2021-11-11 10:00:00
class: post-template
subclass: 'post'
author: uday

---
When applications or services run in environments outside Azure, they need Azure AD application secrets to authenticate to Azure AD and access resources such as Azure and Microsoft Graph. These secrets pose a security risk if they are not stored securely and rotated regularly. Azure AD workload identity federation removes the need for these secrets in selected scenarios. Developers can configure their Azure AD applications to trust tokens issued by another identity provider. This blog post explores how you can access Azure resources without needing secrets when your services are running in the Google Cloud Platform.

![Google to Azure using workload identity federation](/images/gcp-aad-federate/gcp-aad-title.png)


## Using Azure AD workload identity federation with Google Cloud
An [earlier blog post]({% post_url 2021-10-27-azuread-federate-github-actions %}) discussed several aspects of Azure AD workload identity federation. In the GitHub Actions scenario, the AzureLogin action handled all the logic of getting a GitHub token and exchanging it for an Azure AD token. When your service is running in a cloud platform such as Google Cloud, you need to code a similar logic in your service.

![Google AAD federation](/images/gcp-aad-federate/gcp-aad-img.png)

First, let's look at how developers access Azure resources from their services running in Google Cloud today. They first create an application registration in Azure AD and give it the necessary permissions in Azure. Then they configure the application with a secret and use that secret in their service in Google to request an access token for that application from Azure AD.

With Azure AD workload identity federation, you can avoid creating these secrets in Azure AD when your services are running in Google Cloud. Instead, you can configure your Azure AD application to trust a token issued by Google. 

There are three parts to using Azure AD workload identity federation from your service in Google.
1. An identity in Google Cloud to which Google will issue a token.
2. Configure Azure AD application to trust that Google token
3. Get a Google token for your service and exchange it for an Azure AD token

Let's look at each of these three parts in detail.

### Part 1: Use a service account in Google Cloud
We need an identity in the Google Cloud that can be associated with your Azure AD application. Google's concept of [service accounts](https://cloud.google.com/iam/docs/service-accounts) will serve this purpose. You can either use the default service account of your Google project or create a dedicated service account for this purpose.

Each service account has a "Unique ID". When you visit the "IAM & Admin" page in the Google Cloud console, click on Service Accounts. Select the service account you plan to use, and copy its Unique ID. 
![Unique ID of service accounts](/images/gcp-aad-federate/gcp-service-account-img.png)

Tokens issued by Google to the service account will have this unique id as the subject claim. The issuer claim in the tokens will be "https://accounts.google.com". 

We will use these details to configure a trust on Azure AD applications, to allow your application to trust tokens issued by Google to your service account.

### <a name="AzureADTrust"></a> Part 2: Configuring Azure AD application to trust a Google token
Azure AD applications support the capability to add "federatedIdentityCredentials" to set up this trust. You can add up to twenty of these trusts to each Azure AD application.

The most important parts of the federated identity credential are the following:
- subject: this should match the "sub" claim in the token issued by another identity provider, such as Google. This is the Unique ID of the service account you plan to use.
- issuer: this should match the "iss" claim in the token issued by the identity provider. This needs to be an URL that must comply with the OIDC Discovery Spec. Azure AD will use this issuer URL to fetch the keys that are necessary to validate the token. In the case of Google cloud, the issuer is  "https://accounts.google.com"
- audience: this should match the "aud" claim in the token. For security reasons, you should pick a value that is unique for tokens meant for Azure AD. The Microsoft recommended value is "api://AzureADTokenExchange".


#### Permissions needed to create federated credential on an application identity
To create federated credentials on an application, you either need one of these roles:
- [Application Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#application-administrator)
- [Application Developer](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#application-developer)
- [Cloud Application Administrator](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#cloud-application-administrator)
- [Application Owner](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#application-owner)

Or this permission:
- [`microsoft.directory/applications/credentials/update`](https://learn.microsoft.com/en-us/azure/active-directory/roles/custom-available-permissions#microsoftdirectoryapplicationscredentialsupdate)

#### Configuring a federated credential on an application using Azure CLI
Create a JSON file called credential.json.
```
{ "name": "AccessFromGoogle", 
  "issuer": "https://accounts.google.com", 
  "subject": <Unique ID of the service account in Google",
  "audiences": ["api://AzureADTokenExchange"], 
  "description": "This is meant for workloads in project X" 
}
```
Then create a federated identity credential on the Azure AD application

```
az ad app federated-credential create --id <appid> --parameters credential.json
```


### Part 3: Getting a Google token and exchanging it for an Azure AD token
Now that we have configured the Azure AD application to trust the Google service account, we are ready to get a token from Google and exchange it for an Azure AD access token,

#### Getting an ID token for your Google service account
As mentioned earlier, Google cloud resources such as app-engine automatically use the default service account of your Google project. You can also configure the app-engine to use a different service account when you deploy your service. Your service can request an [ID token](https://cloud.google.com/compute/docs/instances/verifying-instance-identity#request_signature) for that service account from the "metadata service endpoint" that handles such requests. With this approach, you dont need any keys for your service account: these are all managed by Google. Here's an example in Node.js:
```javascript
async function googleIDToken() {
    const headers = new Headers();
    const endpoint="http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=api://AzureADTokenExchange";
    return fetch(endpoint, {method: "GET", headers: headers});
}
```
You are requesting Google for a token to identify your service account (an ID token) to Azure AD. Note that the audience here needs to match the audience value you configured on your Azure AD application when setting up the federated identity credential.


#### exchanging the identity token for an Azure AD access token

***This section has been updated in November 2022***

The Azure Identity SDK has added support for a new credential called ClientAssertionCredential. This credential accepts a callback function to get a federated token. I like using this credential since it simplifies the developer experience.

Here’s a sample code snippet to demonstrate this. You can see a more detailed sample in [my GitHub repo](https://github.com/udayxhegde/aad-federate-blobstore-node)

```javascript
import { ClientAssertionCredential } from "@azure/identity";

// pass the googleIDToken function we defined above to ClientAssertionCredential
// It gets called whenever ClientAssertionCredential needs a new federated token from Google
tokenCredential = new ClientAssertionCredential( tenantID,
                                                 clientID,
                                                 googleIDToken ); 

```

Once you have the tokenCredential, you can use this in an Azure SDK such as storage-blob.

```javascript
const { BlobServiceClient } = require("@azure/storage-blob");

const blobClient = new BlobServiceClient(blobUrl, tokenCredential);
```

When requests are made to the blobClient to access storage, the blobClient calls the getToken method on the ClientAssertionCredential. This call results in a request for a fresh token from Google, which then gets exchanged for an Azure AD access token. The ClientAssertionCredential takes care of caching tokens, so tokens are requested from Google or Azure AD only when necessary.

This ClientAssertionCredential is available in the latest shipping Azure Identity SDKs. It was introduced in these versions:

-	.NET: [1.6.0](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/identity/Azure.Identity/CHANGELOG.md#160-2022-04-05)
-	Java: [1.5.0](https://github.com/Azure/azure-sdk-for-java/blob/main/sdk/identity/azure-identity/CHANGELOG.md#150-2022-04-05)
-	JavaScript: [2.1.0](https://github.com/Azure/azure-sdk-for-js/blob/main/sdk/identity/identity/CHANGELOG.md#210-2022-07-08)
-	Python: [1.9.0](https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/identity/azure-identity/CHANGELOG.md#190-2022-04-05)
-	Go: [1.2.0](https://github.com/Azure/azure-sdk-for-go/blob/main/sdk/azidentity/CHANGELOG.md#120-2022-11-08)

Now that we have completed the end-to-end walkthrough, let's look at some additional details.

## Additional details

### Using MSAL instead of AzureIdentity for getting tokens.
If your preferred SDK is Microsoft Authentication Library (MSAL), that SDK is available in the following languages and supports exchanging tokens. 
- [MSAL .Net](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet)
- [MSAL Java](https://github.com/AzureAD/microsoft-authentication-library-for-java)  
- [MSAL Node](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-node)
- [MSAL Python](https://github.com/AzureAD/microsoft-authentication-library-for-python)
- [MSAL Go (Preview)](https://github.com/AzureAD/microsoft-authentication-library-for-go)

[My GitHub repo](https://github.com/udayxhegde/aad-federate-blobstore-node) also has a sample using MSAL to exchange a Google token for an Azure AD token.


### Using an Azure managed identity instead of Azure AD application
***As of October 2022, you can also add federated credentials on a managed identity***

Azure AD has two kinds of workload identities: applications and managed identities. In the walkthrough so far, we considered an application identity. You can also choose to use a managed identity instead of an application identity.
You can configure a federated credential on a managed identity using Azure portal, Azure CLI, or Azure ARM templates.

To create the federated credential on a managed identity, you need to have the role of either owner or contributor of the managed identity. 

#### Configuring a federated credential on a managed identity using Azure CLI
```
az identity federated-credential create --name AccessFromGoogle --identity-name workload-federate-MI1 --resource-group codesamples-rg --issuer https://accounts.google.com --subject <Unique ID for Google service account> --audience api://AzureADTokenExchange
```
O
### Configuring federated credentials programmatically
You can also configure the federated credentials programmatically on both applications and managed identities.

#### Using the Microsoft Graph SDKs for Azure AD application identity
You can use the Microsoft Graph SDKs, available in several languages, to view or modify a federated identity credential. You can choose either the delegated flow or the application-only flow: make sure the service principal you use for this purpose has consent for the Application.ReadWrite.All permissions.

The federatedIdentityCredentials are currently in beta (https://graph.microsoft.com/v1.0/applications/"object-id-of-app"/federatedIdentityCredentials). [The Graph SDK is available in several languages](https://learn.microsoft.com/en-us/graph/sdks/sdks-overview).

#### Using ARM templates for Azure managed identity
You can use [ARM templates, Bicep, or Terraform](https://learn.microsoft.com/en-us/azure/templates/microsoft.managedidentity/userassignedidentities/federatedidentitycredentials?pivots=deployment-language-arm-template) to configure federated identity credentials on a managed identity. 

### In conclusion
Azure AD workload identity federation is a new capability that allows you to get rid of secrets in several scenarios such as GitHub Actions workflow and services running in Google Cloud. Stay tuned for many more scenarios where this capability can be used to get rid of secrets.

If you have any comments, feedback, or suggestions on this topic, I would love to hear from you. [DM me on twitter](https://twitter.com/messages/compose?recipient_id=1446741344)

