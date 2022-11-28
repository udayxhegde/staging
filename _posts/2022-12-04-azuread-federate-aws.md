---
layout: post
current: post
navigation: True
title: Azure AD workload identity federation with AWS
date: 2022-11-10 10:00:00
class: post-template
subclass: 'post'
author: uday

---
Digital transformation is resulting in the deployment of more software workloads. Businesses are building or rewriting their software workloads using cloud-native architectures. Multi-cloud is becoming the new normal, as developers cherry-pick the cloud resources that best suit their needs. When applications or services run in environments outside Azure and need access to Azure resources, they need secrets to authenticate to Azure AD. These secrets pose a security risk. Securing these secrets is a nightmare for developers since the secrets need to be stored securely and rotated regularly. Azure AD workload identity federation removes the need for these secrets in selected scenarios. Developers can configure Azure AD workload identities to trust tokens issued by another identity provider. This blog post explores how you can access Azure resources from software workloads running in Amazon Web Services (AWS). 

![AWS AAD federation](/images/aws-aad-federate/aws-aad-title.png)

## Using Azure AD workload identity federation with AWS
Azure AD workload identity federation is a new capability on workload identities such as Azure AD applications and managed identities. Earlier blog posts on this site provide details for using this capability with [Kubernetes]({% post_url 2022-01-11-azuread-federate-k8s %}), [SPIFFE]({% post_url 2022-01-14-azuread-federate-spiffe %}), [GitHub]({% post_url 2021-10-27-azuread-federate-github-actions %}), and [Google Cloud Platform]({%post_url 2021-11-11-azuread-federate-gcp %}). 

Using this pattern with AWS is not straightforward. The identity model in AWS is different from other platforms such as GCP, Kubernetes, and GitHub. While AWS IAM provides an elegant model for accessing resources within the AWS cloud, it's not easy to use it to access resources in other cloud providers. The step-by-step guide in the first part of this blog post shows how you can pick pieces of AWS Cognito to federate with Azure AD. For a detailed understanding of how this works end-to-end, see the advanced topics later in this blog post.

There are three parts to using Azure AD workload identity federation from your service in AWS.
1. An identity in AWS Cloud to which AWS Cognito will issue a token.
2. Configure Azure AD application or managed identity to trust that AWS identity
3. Get a token for the AWS Cognito identity and exchange it for a token for an Azure AD workload identity.

Let's look at each of these three parts in detail.

### Part 1: Create an identity in AWS Cognito

The AWS Cognito Identity pool is the most suited for our needs. It is necessary, for security purposes, to create a dedicated identity pool to federate with Azure AD since Cognito only issues tokens with the identity pool as the audience. See the advanced topics in the latter part of this blog post for more details on why this matters.
#### Create an identity pool to federate with Azure AD
In this step, we are going to create a Cognito identity pool. And set it up so that we can use identities from that pool for our software workloads. 
Head to the AWS console and select the Cognito service. Go to "Manage Identity Pools" and pick "Create new identity pool".

In the "Create Identity Pool" wizard, provide a name for your identity pool. Then pick "Authentication providers". Select "Custom" and provide a name of your choice in the "Developer provider name". This custom provider option tells Cognito that authorized workloads using this developer provider name can get tokens for identities in this pool.

![Cognito identity pool creation](/images/aws-aad-federate/cognito-pool-wizard.png "Creating an identity pool") 

Once you create the pool, Cognito prompts you to create IAM roles. This is only necessary when you use Cognito to access AWS resources. Since this pool is dedicated to Azure, select “Cancel” to avoid creating these roles.

You will now have the id of the identity pool you just created. You can also find the Identity pool id by selecting the pool in Cognito. This selection takes you to the Cognito pool dashboard where you can see the number of Cognito identities created and their activity.

![Cognito identity pool dashboard](/images/aws-aad-federate/cognito-pool-dashboard.png)

 Notice the warning "You have not specified roles for this identity. Click here to fix it". Since these roles are only required when you use Cognito to access AWS resources, you can ignore this warning. Selecting "Edit identity pool" takes you to the details of your identity pool, including the pool id.

In my case, the identity pool id is:
```
us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef
```

#### Create a Cognito identity for our workload
At this point, we have created a Cognito identity pool. You can now create identities in this pool to be used by your software workload. To create an identity, you can use AWS CLI with your credentials and do the following:

```
aws cognito-identity get-open-id-token-for-developer-identity \
    --identity-pool-id <the pool id you just created>  \
    --logins <developer provider name>=<a_unique_string_identifying_your_workload> 
    --region <aws region>
```

for example, in my case:
```
aws cognito-identity get-open-id-token-for-developer-identity \
    --identity-pool-id us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef  \
    --logins azure-access=wlid_1 --region us-east-1
```
Note that wlid_1 is just a random string I made up for one of my workloads. Any workload authorized by me to get tokens from this pool can request tokens by providing this identity. Here's the response from the AWS CLI.
```
{
    "Token": <a long token string>,
    "IdentityId": "us-east-1:d2237c93-e079-4eb3-964c-9c9a87c465d7"
}
```
Since this is the first time the identity pool sees a request for wlid_1, it created an identity in the pool and linked it with my wlid_1 identity. Any future token request for wlid_1 will always return a token for this IdentityId. The subject claim of the token will contain the Cognito IdentityId. In my example above, the subject claim will be "us-east-1:d2237c93-e079-4eb3-964c-9c9a87c465d7"

You may wonder how you will keep track of an additional set of these developer identities for your workloads. A simpler alternative is to use the client_id of the Azure AD workload identity you plan to use for the federation instead of inventing your strings.
Let's use that approach in this walkthrough and create a Cognito identity using the client_id of the managed identity we will use.

```
aws cognito-identity get-open-id-token-for-developer-identity \
    --identity-pool-id us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef  
    --logins azure-access=cc312bc3-3859-42f1-9598-2371055dbfa4 --region us-east-1
```
Here's the response from AWS Cognito
```
{
    "Token": <a long token string>,
    "IdentityId": "us-east-1:fa442090-a36f-44d4-81ac-ab21418d0e90"
}
```
At this point, we have a Cognito identity linked to the client_id of our managed identity. You can visit the identity browser in Cognito to see your identities. In the below screenshot, notice that the Cognito identity is linked to my developer-provided identity.

![Cognito identity browse](/images/aws-aad-federate/cognito-identity-browse-combined.png)


### Part 2: Configuring an Azure AD identity to trust the AWS Cognito token

Azure AD has two kinds of workload identities: applications and managed identities. Both identities support federation with a token from an OIDC token issuer. 

In this step-by-step, we will use a managed identity that has already been granted access to my Azure storage. We will tell Azure AD to link the managed identity to the Cognito identity by creating a Federated Identity Credential on the managed identity. You can add up to twenty of these trusts to each Azure AD workload identity.

The most important parts of the federated identity credential are the following:
- subject: this should match the "sub" claim in the token issued by another identity provider, such as AWS Cognito. This is the IdentityId we got from Cognito in the earlier section. (In this example: "us-east-1:fa442090-a36f-44d4-81ac-ab21418d0e90").
- issuer: this should match the "iss" claim in the token issued by the identity provider. The issuer is an URL that must comply with the OIDC Discovery Spec. Azure AD will use this issuer URL to fetch the keys necessary to validate the token. In the case of AWS Cognito, the issuer is  "https://cognito-identity.amazonaws.com"
- audience: this should match the "aud" claim in the token. For security reasons, you should pick a value that is unique for tokens meant for Azure AD. The Microsoft recommended value is "api://AzureADTokenExchange". However, there is no option to configure this in AWS Cognito. It only issues tokens with the identity pool id as the audience. So we will configure the audience in the federated identity credential with the Identity pool id we got in Part 1. (In this example: us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef)

Let's use these values to configure the federated identity credential on our managed identity. You can configure the federated credential on a managed identity using the Azure portal, the Azure CLI, or Azure ARM templates. To create the federated credential, you need to have the role of either owner or contributor of the managed identity. 

#### Using Azure CLI
```
az identity federated-credential create --name AccessFromAWS --identity-name workload-federate-MI1 \
   --resource-group codesamples-rg --issuer https://cognito-identity.amazonaws.com \
   --subject us-east-1:fa442090-a36f-44d4-81ac-ab21418d0e90 --audience us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef
```
On success, you should expect to see something like this:

```
{
  "audiences": [
    "us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef"
  ],
  "id": "/subscriptions/22a08bf1-fb31-4757-a836-cd035976a2c0/resourcegroups/codesamples-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/workload-federate-MI1/federatedIdentityCredentials/AccessFromAWS",
  "issuer": "https://cognito-identity.amazonaws.com",
  "name": "AccessFromAWS",
  "resourceGroup": "codesamples-rg",
  "subject": "us-east-1:fa442090-a36f-44d4-81ac-ab21418d0e90",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
}
```


### Part 3: Getting a Cognito token and exchanging it for an Azure AD token
Now that we have configured an Azure AD workload identity to trust the Cognito identity, we are ready for our software workload to get a token from Cognito and exchange it for an Azure AD access token.

We will use the AWS Cognito SDK to get Cognito tokens. Depending on where the software workload is running, EC2 or Lambda, you first need to grant the compute environment permissions to request tokens from your Cognito pool.

#### Configure your EC2 instance or Lambda with permissions to your Cognito identity pool
Head to the AWS IAM console. Create a permission policy for the Cognito pool. Pick the Cognito Identity service, and add the actions of "GetOpenIdTokenForDeveloperIdentity", "LookupDeveloperIdentity", "MergeDeveloperIdentities", and "UnlinkDeveloperIdentities". Pick the specific resource that identifies the Cognito identity pool just created.

![Cognito permission](/images/aws-aad-federate/cognito-pool-permission.png) 

Give an appropriate name for this policy and create it.

![Cognito permission](/images/aws-aad-federate/cognito-pool-permission2.png) 

Create an EC2 role, and assign this permission policy to that role.

![EC2 role](/images/aws-aad-federate/cognito-pool-permission3.png) 

Assign this role to our EC2 instance. Any software workload running on this EC2 instance now has permission to request tokens from our Cognito pool.

![Assign to EC2 instance](/images/aws-aad-federate/cognito-pool-permission4.png) 

#### Get the AWS token from your code in EC2

Use the client-cognito-identity AWS SDK to get Cognito tokens.
```javascript
import { CognitoIdentityClient, GetOpenIdTokenForDeveloperIdentityCommand } from "@aws-sdk/client-cognito-identity"; // ES Modules import
```

Create a CognitoIdentityClient.

```javascript
    cognitoClient = new CognitoIdentityClient({ region }); 
```

Use this client to get Cognito tokens.

```javascript
async function getCognitoToken() {
    var Logins:any = {};
    
    // hardcoding value here for demonstration purposes only. 
    // "cc312bc3-3859-42f1-9598-2371055dbfa4" is our developer id assigned to this
    // workload, it is linked to a unique Cognito identity created within the pool 
    // when we used the aws cli earlier in this blog post. azure-access is 
    // the developer provider name that we configured when we created the Cognito pool
    Logins["azure-access"] = "cc312bc3-3859-42f1-9598-2371055dbfa4"; 
        
    const command = new GetOpenIdTokenForDeveloperIdentityCommand(
                            {IdentityPoolId: poolId, Logins });

    return cognitoClient.send(command)
    .then(function(data:any) {
        return data.Token;
    })
    .catch(function(error:any) {
        throw(error);
    });
}
```

#### Exchanging the identity token for an Azure AD access token
The Azure Identity SDK has added support for a new credential called ClientAssertionCredential. This credential accepts a callback function to get a federated token. I like using this credential since it simplifies the developer experience.

Here’s a sample code snippet to demonstrate this. You can see a more detailed sample in [my GitHub repo](https://github.com/udayxhegde/aad-federate-blobstore-node)

```javascript
import { ClientAssertionCredential } from "@azure/identity";

// pass the getCognitoToken function we defined above to ClientAssertionCredential
// It gets called whenever ClientAssertionCredential needs a new federated token from Cognito
tokenCredential = new ClientAssertionCredential( tenantID,
                                                 clientID,
                                                 getCognitoToken ); 

```

Once you have the tokenCredential, you can use this in an Azure SDK such as storage-blob.

```javascript
const { BlobServiceClient } = require("@azure/storage-blob");

const blobClient = new BlobServiceClient(blobUrl, tokenCredential);
```

When requests are made to the blobClient to access storage, the blobClient calls the getToken method on the ClientAssertionCredential. This call results in a request for a new token from Cognito, which then gets exchanged for an Azure AD access token. The ClientAssertionCredential takes care of caching tokens, so tokens are requested from Cognito or Azure AD only when necessary.

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

[My GitHub repo](https://github.com/udayxhegde/aad-federate-blobstore-node) also has a sample using MSAL to exchange a Cognito token for an Azure AD token.


### Using an Azure AD application instead of managed identity
Azure AD has two kinds of workload identities: applications and managed identities. In the walkthrough so far, we considered a user-assigned managed identity. You can also choose to use an application identity instead of managed identity.
You can configure a federated credential on an application using Azure portal, Azure CLI, or Microsoft Graph SDK.

#### Permissions needed to create federated credentials on an application identity
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
{ "name": "AccessFromAWS", 
  "issuer": "https://cognito-identity.amazonaws.com", 
  "subject": "us-east-1:fa442090-a36f-44d4-81ac-ab21418d0e90",
  "audiences": ["us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef"], 
  "description": "This is meant for workloads in project X" 
}
```
Then create a federated identity credential on the Azure AD application

```
az ad app federated-credential create --id <appid> --parameters credential.json
```
In my example, it succeeds with this message:
```JSON
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#applications('1a4314ae-2735-4fc8-866d-c7d320ab8fd0')/federatedIdentityCredentials/$entity",
  "audiences": [
    "us-east-1:59d4a12a-deaa-4a98-85d1-b5d9fc2d41ef"
  ],
  "description": "This is meant for workloads in project X",
  "id": "430e2e5d-d06e-4bd2-a6aa-b10f3255e5e7",
  "issuer": "https://cognito-identity.amazonaws.com",
  "name": "AccessFromAWS",
  "subject": "us-east-1:fa442090-a36f-44d4-81ac-ab21418d0e90"
}
```

### Configuring federated credentials programmatically
You can also configure the federated credentials programmatically on both applications and managed identities.

#### Using the Microsoft Graph SDKs for Azure AD application identity
You can use the Microsoft Graph SDKs, available in several languages, to view or modify a federated identity credential. You can choose either the delegated flow or the application-only flow: make sure the service principal you use for this purpose has consent for the Application.ReadWrite.All permissions.

The federatedIdentityCredentials API URL is (https://graph.microsoft.com/v1.0/applications/"object-id-of-app"/federatedIdentityCredentials). [The Graph SDK is available in several languages](https://learn.microsoft.com/en-us/graph/sdks/sdks-overview).

#### Using ARM templates for Azure managed identity
You can use [ARM templates, Bicep, or Terraform](https://learn.microsoft.com/en-us/azure/templates/microsoft.managedidentity/userassignedidentities/federatedidentitycredentials?pivots=deployment-language-arm-template) to configure federated identity credentials on a managed identity. 


### Understanding the token details

AWS is different from other cloud providers when dealing with identities for software workloads. There is no native concept of a workload identity or service account. And there is no direct mechanism for an EC2 instance to request a JWT token, for example via IMDS, to authenticate to another identity provider.

I am new to Cognito, so take everything I say here with a grain of salt. My colleague, Alexander Riman, suggested that the Cognito Identity pool may be a viable option to federate with Azure AD.

AWS Cognito issues JWT tokens which can be validated via the OpenID Connect protocol. Cognito has two pools, the User pool, and the Identity pool. The User pool enables developers to authenticate users to software workloads. The Identity pool has a custom authentication provider for developers to get tokens identifying their software workloads. The identity pool's primary purpose is to authenticate workloads to access AWS resources. The audience claim in the token is always the identity pool id and is not customizable. Despite this limitation, the Identity pool serves the purpose of federating with Azure AD.

Cognito [documents](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html) two auth flows in the Identity pool, the enhanced auth flow, and the basic auth flow. 

#### Understanding the Cognito enhanced auth flow
This flow is also called the simplified flow. In this flow, the workload only communicates with the Cognito service and gets temporary credentials for one of the two roles in Cognito. It can access the AWS resources available to that role.
![Cognitor enhanced end flow](/images/aws-aad-federate/amazon-cognito-dev-auth-enhanced-flow.png)
1. The software workload uses a developer provided id and requests a token from Cognito (GetOpenIdTokenForDeveloperIdentity)
2. Cognito finds the Cognito id matching the developer id (or creates a new matching Cognito id) and returns a token for that identity.
3. The software workload presents the token to Cognito. Cognito matches this to one of the two roles in Cognito. Cognito requests temporary credentials from AWS STS for the AWS resources permitted via that role.
4. Cognito returns the temporary creds for access to those AWS resources.

In this auth flow, you manage only two roles in Cognito. However, all your workloads get access to the same set of AWS resources available through the two roles configured for that pool.

#### Understanding the Cognito basic auth flow
In the basic auth flow, the software workload uses the JWT token issued by Cognito to assume an AWS IAM role. 

![Cognito basic flow](/images/aws-aad-federate/amazon-cognito-dev-auth-basic-flow.png)
1. The software workload uses a developer provided id and requests a token from Cognito (GetOpenIdTokenForDeveloperIdentity)
2. Cognito finds the Cognito id matching the developer id (or creates a new matching Cognito id) and returns a token for that identity.
3. The software workload presents the token to AWS STS with an AWS IAM role to AssumeRoleWithWebIdentity.
4. AWS IAM returns the temporary creds for that role. The software workload can access AWS resources assigned to that role. 

This approach allows you to provide finer-grained access to your workloads by defining different roles in AWS IAM.

#### Understanding the Cognito to Azure AD auth flow
Since we are not configuring access to AWS resources, both these flows are irrelevant to our use. We use a pattern similar to the basic auth flow: get a token from Cognito and then present it to Azure AD to get a token for an identity with access to Azure resources.

![end to end flow](/images/aws-aad-federate/cognitoaadflow.png)
1. The software workload uses a developer provided id and requests a token from Cognito (GetOpenIdTokenForDeveloperIdentity)
2. Cognito finds the Cognito id matching the developer id (or creates a new matching Cognito id) and returns a token for that identity.
3. The software workload presents the token to Azure AD and requests a token for a managed identity with a federated credential accepting the Cognito identity.
4. Azure AD validates the token with Cognito and returns an Azure AD token.

Since we are not accessing AWS resources using our pool, we don’t need to add any roles to Cognito.

### The need to keep the Cognito pool dedicated to Azure AD federation
The audience claim in JWT tokens is critical for security reasons. It contains the intended recipient of the token. Any service receiving a token must check it is the intended recipient of the token. To see why this is important, let's consider a hypothetical example. Service-A needs to authenticate to Service-B and Service-C. If it uses the same token to authenticate to both services, Service-B can also use that token to act like Service-A when accessing Service-C.  However, if the token had the audience indicating the token was for Service-B, then Service-C would reject that token since the intended audience is Service-B.

In the  WS Cognito identity pool, the audience is fixed and always the identity pool id.  Tokens issued by the identity pool are intended for a single recipient. When you configure Azure AD to accept tokens with the identity pool id as the audience, you make Azure AD the intended recipient of these tokens. It's a security best practice to avoid using tokens with the same audience when accessing AWS resources. AWS resources. Use a different identity pool for that purpose.

## In conclusion
Azure AD workload identity federation is a new capability that allows you to get rid of secrets in several scenarios such as services running in Kubernetes clusters, GitHub Actions workflow, and services running in Google and AWS Cloud. Stay tuned for many more scenarios  where this capability can help remove secrets.

If you have any comments, feedback, or suggestions on this topic, I would love to hear from you. [DM me on twitter](https://twitter.com/messages/compose?recipient_id=1446741344)

