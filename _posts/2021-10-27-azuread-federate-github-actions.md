---
layout: post
current: post
navigation: True
title: Use Azure AD workload identity federation to remove secrets in GitHub Actions
date: 2021-10-27 10:00:00
class: post-template
subclass: 'post'
author: uday

---
GitHub announced support for federating with identity providers to get rid of secrets in the Actions workflows. Developers can now deploy from their GitHub repositories to their Azure resources using the identity of their GitHub repo and their workflow job! This avoids the need to create and store secrets from Azure AD into their GitHub repo. To support this model, Azure AD has a new capability, currently in public preview, called Workload Identity Federation. This capability allows you to configure Azure AD applications to trust the GitHub token (JWT) that represent your GitHub repo and workflow. This blog post looks under the covers on how this works end to end.

![GitHub Actions AAD federation](/images/github-federate/githubactions_img.png)

## What is Azure AD workload identity federation?
When applications, scripts, or services run in Azure, they can use Azure managed identities to avoid dealing with secrets for Azure AD identities. With Azure managed identities, the secrets are stored and managed by the Azure platform.

However, when applications or services run in environments outside Azure, they need Azure AD application secrets to authenticate to Azure AD and access resources such as Azure and Microsoft Graph. These secrets pose a security risk if they are not stored securely and rotated regularly. Azure AD workload identity federation removes the need for these secrets in selected scenarios. Developers can configure their Azure AD applications to trust tokens issued by another identity provider. These trusted tokens can then be used to access resources available to those applications.

### The problems customers face with secrets today
When applications need to be provisioned with secrets, there are several challenges faced by developers, DevOps, and IT admins today:
- Where to store these secrets for them to be available to the applications? The natural place to store this is in your application config or expose them as environment variables. But making them part of your code and checking it into your GitHub repository is a recipe for disaster. Anyone having access to the repo can also use those secrets. To avoid this, developers have to consider secure storage solutions such as a vault.

- How to mitigate potential leakage of secrets? Even as you store the secret securely, you should expect the "bad guys" are trying to break your secret. It's a matter of time before they can figure out the secret, even though that time may be several weeks or months. So to prevent the inevitable, most companies require secrets to expire after a fixed amount of time. This forces developers to rotate them often.

- Dealing with service downtime. When secrets expire, the service loses access to resources which can cause parts of the service to be unavailable. Ideally, this is avoided by building regular hygiene for rotating secrets before they expire. In reality, however, this rotation is hard to automate.

- Secrets leaving the organization. When secrets are created and handled manually, it allows someone to walk away with the secrets when they leave the company. This poses a risk of unauthorized access to resources through these secrets. It is usually not practical to rotate secrets every time someone leaves the organization. Any aggressive secret rotation has negative consequences of service downtime.

### How Azure AD workload identity federation works
There are two main components to Azure AD workload identity federation.
1. Configuring your application to trust a token issued by an external provider (like GitHub)
2. Exchanging the trusted token to an Azure AD access token

#### Configure the Azure AD application to trust an external token
Azure AD applications now support "Federated Identity Credentials".These can be added using [Microsoft Graph APIs](https://docs.microsoft.com/en-us/graph/api/resources/federatedidentitycredentials-overview?view=graph-rest-beta). Adding this credential allows you to indicate which token is trusted by your application. Three important pieces are needed when setting up this trust. Tokens with claims that match these pieces are considered as trusted by Azure AD. These three pieces are issuer, subject, and audience. 
1. The issuer is a URL that must comply with the [OIDC Discovery Spec](https://openid.net/specs/openid-connect-discovery-1_0.html). During the token exchange step, Azure AD will use this URL to get the keys necessary to validate the token.

2. The subject is a string that identifies an entity within the issuer. In the case of GitHub, this would identify your repo and workflow job details.

3. The audience tells Azure AD what value will be in trusted tokens to indicate those tokens are meant for Azure AD. The recommended audience value for Azure AD is api://AzureADTokenExchange. Using this value minimizes any mistakes in the configuration. 

The federated identity credential can be configured either via the Azure AD portal, the Azure CLI, or using the Graph SDK. We will see specific examples when we look at the GitHub scenario in more detail later in this blog.

#### Exchange an external token to an Azure AD access token
Once the Azure AD application is configured with a federated identity credential, you can later exchange an external token matching that trust for an Azure AD access token. *The external token should be a JWT matching the [ID token spec](https://openid.net/specs/openid-connect-core-1_0.html). In addition, the header "typ" must be "JWT".* Azure AD workload identity federation uses the [OAuth 2.0 client credentials flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow#third-case-access-token-request-with-a-federated-credential) to allow this exchange. You specify the client_id for which you are requesting the Azure AD token. And you pass the external token as the client_assertion in this flow. 
```
POST /{tenant}/oauth2/v2.0/token HTTP/1.1               
Host: login.microsoftonline.com
Content-Type: application/x-www-form-urlencoded

scope=https%3A%2F%2Fgraph.microsoft.com%2F.default
&client_id=97e0a5b7-d745-40b6-94fe-5f77d35c6e05
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsIng1dCI6Imd4OHRHeXN5amNScUtqRlBuZDdSRnd2d1pJMCJ9.eyJ{a lot of characters here}M8U3bSUKKJDEg
&grant_type=client_credentials
```

Azure Identity SDKs and MSAL (Microsoft Authentication Library) have been updated to support this token exchange flow. More details on that in a later blog.

There is also a [beta release of Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-beta?tabs=powershell) that allows you to log in the Azure CLI using a service principal id and a trusted token. 
```
az login --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID --federated-token $(cat $AZURE_FEDERATED_TOKEN_FILE) --subscription $SUBSCRIPTION_ID

```


## Using Azure AD workload identity federation with GitHub Actions


### Configuring Azure AD application to trust the GitHub token
As discussed earlier, you first need to configure a federated identity credential on your application, to trust tokens issued by GitHub to your workflow. The issuer, subject and audience components of the federated credential need to be as follows:

1. The GitHub token issuer is https://token.actions.githubusercontent.com
2. The GitHub subject claim is generated by GitHub based on your workflow.
- Workflow job in an environment, repo:<org/repo>:environment:<env>
- Workflow triggered by pull request, repo:<org/repo>:pull_request
- Workflow originated from a branch, repo:<org/repo>:ref:refs/heads/<branch>
- Workflow originated with a tag, repo:<org/repo>:ref:refs/tag/<tag-name>
- [See more examples here](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#examples)


3. The recommended audience value for Azure AD is api://AzureADTokenExchange. This is the value used by the azure/login action when requesting a token from GitHub.

To create a federated credential on your Azure AD application, the easiest option is to use the Azure AD portal. Go to the App registrations blade and find your application. Open "Certificates & secrets". You will see an option to add Federated credentials. You can provide your repo name and the workflow details and it generates the federated credential with the issuer and subject matching the GitHub token.
![Azure AD portal for federated credential](/images/github-federate/githubactions_aad_img.png)

Another option to add a federated credential to your application is to use the Azure CLI. The Azure CLI is being updated to support federated credentials natively. Right now you can use its REST interface (az rest) to call the Microsoft Graph API to configure this. In a bash cmd, do the following:
```
 az rest --method POST --uri 'https://graph.microsoft.com/beta/applications/'<application object id>'/federatedIdentityCredentials' --body '{"name":"RepoXBranchY","issuer":"https://token.actions.githubusercontent.com","subject":"<orgname/repo-name>:ref:refs/heads/<branch-name>","audiences":["api://AzureADTokenExchange"]}'
```
You can also use the Microsoft Graph SDK to programmatically add federated credentials to your application.


### Setup your GitHub Actions workflow to use federation
GitHub has added the ability for the actions in your workflow to get a GitHub token. Since these tokens can be used to access resources in different cloud providers, a developer has to explicitly set permission in their job to enable tokens. 
#### Configure your workflow to enable GitHub tokens
You add the permissions section in your workflow, indicating the need for a token in the workflow
```
on:
  push:
    branches: [ main ]

permissions:
   id-token: write
   contents: read
```

#### Update your azure login action to the version that supports federation
The Azure login action has been updated to use the GitHub token if it is available. It will also perform the token exchange with Azure AD and get an Azure AD access token. Later actions can use this Azure AD token to deploy to Azure resources.
```
- name: 'Az CLI login'
      uses: azure/login@v1
      with:
        client-id: <your Azure AD Application id>
        tenant-id: <your Azure AD tenant id>
        subscription-id: <your Azure subscription id>
```
Notice there are no Azure AD secrets passed to the Azure login action.

Other than these changes, the rest of your workflow remains unchanged.


### Digging into the details of the end-to-end flow.
Now that we saw how to configure all the pieces for this to work, let's look at a few parts to understand them better.

#### The initial GitHub token
When you add permission to the workflow to enable tokens, GitHub first issues a bearer token in an environment variable. Subsequent actions need this token to request a GitHub JWT that represents the repo and job details.

#### Using the bearer token to get a GitHub JWT
Actions, such as azure/login@v1.4.0, will use the bearer token issued to the workflow to request a JWT token with Azure AD as the audience. You can also choose to get the JWT token on your own if you prefer to write your custom actions. Here is an example:

```
 - run: |
     bearerToken=${ACTIONS_ID_TOKEN_REQUEST_TOKEN}
     runtimeUrl=${ACTIONS_ID_TOKEN_REQUEST_URL}
     runtimeUrl="${runtimeUrl}&audience=api://AzureADTokenExchange"       
     echo ::set-output name=JWTTOKEN::$(curl -H "Authorization: bearer $bearerToken" $runtimeUrl | jq -r ".value")
   id: tokenForAAD
```

#### Anatomy of a GitHub JWT token
The GitHub JWT token consists of multiple claims that identify the specifics of the repo and the job. [See the anatomy of the GitHub JWT](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token). Azure AD will only use the issuer, subject, and audience claims in the GitHub JWT.

The default lifetime for the GitHub JWT is 1 hour. Workflows that run for longer durations will need additional work in keeping the tokens refreshed.

#### A note on the audience for the JWT token
By default, GitHub issues JWT tokens with the audience being the org name (https://github.com/orgname). When dealing with multiple identity providers within a job, it is a good security practice to request tokens with a different audience value for each of these providers. This avoids security issues such as a token presented to one provider cannot be misused to access resources protected by another identity provider.

For Azure AD, the recommended value of this audience is api://AzureADTokenExchange. However, you can choose to customize this to another audience value. Just make sure the audience configured on the Azure AD application for this trust relationship matches what is in the GitHub JWT. The azure/login action uses the recommended value and the Azure AD portal defaults to this recommended value. This avoids mistakes such as the audience value in the federated credential not matching the one in the JWT. 

### In conclusion
Azure AD workload identity federation is a new capability that allows you to get rid of secrets in the GitHub Actions workflow. Stay tuned for many more scenarios where this capability can be used to get rid of secrets.

If you have any comments, feedback or suggestions  on this topic, I would love to hear from you. [DM me on twitter](https://twitter.com/messages/compose?recipient_id=1446741344)

