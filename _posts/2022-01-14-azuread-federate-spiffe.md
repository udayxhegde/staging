---
layout: post
current: post
navigation: True
title: Azure AD workload identity federation with SPIFFE and SPIRE
date: 2022-01-14 10:00:00
class: post-template
subclass: 'post'
author: uday

---
[SPIFFE](https://spiffe.io/docs/latest/spiffe-about/overview/) is a set of open-source standards for providing identities to your software workloads. Since it is platform agnostic with possibilities such as mTLS, it is an attractive option for services deployed across platforms and cloud vendors. 
The [Kubernetes blog post]({% post_url 2022-01-11-azuread-federate-k8s %}) discussed how services running in a Kubernetes cluster can use Azure AD workload identity federation to access Azure resources without needing secrets. This blog post explores how services relying on SPIFFE can also use this capability to access Azure resources. No secrets are necessary.

![SPIFFE AAD federation](/images/spiffe-federate/spiffe-aad-title.png)


## What are SPIFFE and SPIRE?
SPIFFE is a set of open-source standards and specifications. They specify how software workloads can dynamically get an identity (SPIFFE ID) in heterogeneous environments. These workloads are issued cryptographic identity documents called SVIDs in two formats: X.509 certificate and JWT token. There are several reasons why SPIFFE is an attractive option for developers:
- The short-lived X.509 certs enable mutual authentication and data encryption (mTLS)
- Allows a uniform model for building services deployed in heterogeneous platforms and environments, with support for Kubernetes, Azure, AWS, GCP, bare-metal, etc.
- Workload attestation allows dynamic assignment of identities as services come and go
- User-friendly names for SPIFF ID (eg: spiffe://demo.identitydigest.com/demo/client)
- Can integrate with service mesh offerings such as Istio.
- Open-source.

SPIRE is the reference implementation of the SPIFFE specifications. 

The [SPIFFE concepts](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/) and [SPIRE concepts](https://spiffe.io/docs/latest/spire-about/spire-concepts/) are great resources to learn more about SPIFFE and SPIRE. The [quickstart for Kubernetes](https://spiffe.io/docs/latest/try/getting-started-k8s/) and [OIDC authentication with AWS](https://spiffe.io/docs/latest/keyless/oidc-federation-aws/) are good references to use for deploying SPIFFE and SPIRE. 

## Using Azure AD workload identity federation with SPIFFE and SPIRE

![SPIFFE AAD federation](/images/spiffe-federate/spiffefederate_img.png)

My [GitHub repo](https://github.com/udayxhegde/aad-federate-blobstore-node) uses a modified version of these quickstart materials. It has two key aspects that are different from the SPIRE quickstarts:
- We will use an updated version of the OIDC discovery provider.  It supports adding the "use" key required by Azure AD in the OIDC discovery document. Rather than co-host this provider with the SPIRE server, we will run it as a separate service.
- We will use a client workload that gets a SPIFFE JWT token and accesses Azure Blob Store.

Our walk-through will comprise of four parts:
1. Setup SPIRE components in a Kubernetes cluster. Configure it with JWT and OIDC discovery support.
2. Deploy a sample workload in our cluster and assign it a SPIFFE ID 
3. Configure an Azure AD application to trust the SPIFFE ID.
4. Use the SPIFFE JWT issued to our service to access Azure resources

### Part 1: Setting up the SPIRE components with JWT and OIDC support
We will use example.org as our trust domain, as used in the SPIRE quickstarts. The [yaml files in my repo](https://github.com/udayxhegde/aad-federate-blobstore-node/tree/main/deployment/spiffe) are in two folders:
1. [spire server and agents](https://github.com/udayxhegde/aad-federate-blobstore-node/tree/main/deployment/spiffe/spire)
2. [OIDC discovery](https://github.com/udayxhegde/aad-federate-blobstore-node/tree/main/deployment/spiffe/oidc)

For this part, you need the following:
- a Kubernetes cluster where you deploy the SPIRE components (server, agent, OIDC discovery).
- a domain name to host the OIDC discovery endpoint. You will need permissions to manage the DNS records of that domain name.

You also need to customize these files from my repository to suit your environment. The following changes are needed:
1. Look for the TODO comments in server-configmap-oidc.yaml. Replace the FQDN of the discovery domain with your domain name. Replace the cluster name with the name of your cluster.
2. Look for the TODO comments in the agent-configmap.yaml. Replace the cluster name with the name of your cluster
3. Look for the TODO comments in the oidc-ingress.yaml. Replace the FQDN of the OIDC discovery domain with your domain name
4. Look for the TODO comments in the oidc-dp-configmap.yaml. Replace the FQDN of the OIDC discovery domain with your domain name. Also, change the email reference.

#### After you make these changes, deploy the SPIRE server:
Go to the deployment/spiffe/spire directory and deploy the server:
```
kubectl apply -f spire-namespace.yaml
kubectl apply -f server-account.yaml \
              -f spire-bundle-configmap.yaml \
              -f server-cluster-role.yaml
kubectl apply -f server-configmap-oidc.yaml \
              -f server-statefulset.yaml \
              -f server-service.yaml
```
Next, deploy the agents:
```
kubectl apply -f agent-account.yaml \
              -f agent-cluster-role.yaml
kubectl apply -f agent-configmap.yaml \
              -f agent-daemonset.yaml
```

Confirm that the server and agents are up and running.
```
kubectl get pods -n spire

NAME                                   READY   STATUS    RESTARTS   AGE
spire-agent-2w9dp                      1/1     Running   0          1d
spire-agent-bgpzq                      1/1     Running   0          1d
spire-agent-ctjgj                      1/1     Running   0          1d
spire-server-0                         1/1     Running   0          1d
```

Assign a SPIFFE identity to each of the agents running on the nodes. Agents will be able to authenticate to the SPIFFE server using this identity. Following this, we can also assign identities to our other workloads running in the cluster.

```
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
  -selector k8s_sat:cluster:<YOUR_CLUSTER_NAME> \
  -selector k8s_sat:agent_ns:spire \
  -selector k8s_sat:agent_sa:spire-agent \
  -node
```

#### Now deploy the OIDC server
The SPIRE implementation of the OIDC discovery provider is at https://gcr.io/spiffe-io/oidc-discovery-provider. 

Azure AD requires that the JWKS pointed by the OIDC discovery URL contains a "use" key with the value "sig". By default, the SPIRE OIDC discovery provider does not add this key. Adding this key is [a recent option](https://github.com/spiffe/spire/issues/2623). It's available in v1.1.2 or above as well as the recent nightly builds of the OIDC discovery provider at gcr.io/spiffe-io/oidc-discovery-provider:nightly. 

Go to the deployment/spiffe/oidc directory, and deploy the OIDC discovery service.
```
kubectl apply -f oidc-account.yaml \
              -f oidc-dp-configmap.yaml
kubectl apply -f oidc-ingress.yaml \
              -f oidc-service.yaml
kubectl apply -f oidc-deployment.yaml

kubectl get pods -n spire
NAME                                   READY   STATUS    RESTARTS   AGE
spire-agent-2w9dp                      1/1     Running   0          1d
spire-agent-bgpzq                      1/1     Running   0          1d
spire-agent-ctjgj                      1/1     Running   0          1d
spire-oidc-provider-8666c9c5fb-zzr6k   1/1     Running   0          1d
spire-server-0                         1/1     Running   0          1d
```
This OIDC provider will set up the OIDC discovery endpoint for us. It will publish the keys needed by other providers to validate the tokens issued by the SPIRE server. We will assign it a SPIFFE identity to authenticate to the server.
```
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://example.org/oidc-discovery \
  -parentID spiffe://example.org/ns/spire/sa/spire-agent \
  -selector k8s:ns:spire \
  -selector k8s:sa:spire-oidc
```

The OIDC discovery endpoint needs to be reachable over the internet. This will allow other identity providers to validate the JWT issued to workloads. For this reason, the OIDC provider service will publish an external IP address. 
```
kubectl get service -n spire

NAME           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
spire-oidc     LoadBalancer   10.0.170.196   21.98.214.16   443:31873/TCP    13d
spire-server   NodePort       10.0.89.46     <none>        8081:30494/TCP   13d
```
Your OIDC discovery domain should point to this external IP of the OIDC endpoint. Visit the DNS administration page for your OIDC discovery domain. Add an A record for your host with the external IP address as the value. For example, I am using demo-oidc.identitydigest.com. So my A record says host=oidc, value=IP address of my OIDC service.

At this point, you should see your OIDC discovery endpoint and the related JWKS:
https://YOUR_OIDC_DISCOVERY_DOMAIN/.well-known/openid-configuration

This is roughly what you would expect to see. It's an example from my demo setup:

```
{
  "issuer": "https://demo-oidc.identitydigest.com",
  "jwks_uri": "https://demo-oidc.identitydigest.com/keys",
  "authorization_endpoint": "",
  "response_types_supported": [
    "id_token"
  ],
  "subject_types_supported": [],
  "id_token_signing_alg_values_supported": [
    "RS256",
    "ES256",
    "ES384"
  ]
}
```
https://YOUR_OIDC_DISCOVERY_DOMAIN/keys. This is an example from my demo setup: 
```
{
  "keys": [
    {
      "use": "sig",
      "kty": "RSA",
      "kid": "mM0QXrh1f17ChHyWHQO0OF4zHT5isaYb",
      "n": "zmK4Kfn00s-ZDZbUICSaTewB9ZmTmVrf_uRk6wsL3PdNqNXhznrssH7qmaIprw-it3ReZPYkVyuPrrFY1soWu39a1U7dY48BwuqbIP8DsIUMTa_0FjJtgQ2M4JuYicSpFBY20By85hdQF3Gyzge1WCXf_HwFaCmu0qJ_xshYMABeUM_wm8ioSk67Jfe3pMfDKkY-UETJ3UnssXyHAhKTbHO0aqEmm5Szu6pzs3KQwedOLJkfrTBFDwJNWncczJjPJoOm_enRAH4NVQ-LdDEf5vze0StXxegiSOvmmn0W01Iqr9ukr-kGh7ch4meQ8rihKQMLQEi3hzQC8XAPgOcH3Q",
      "e": "AQAB"
    }
  ]
}
```
Notice the "use" key needed by Azure AD. We are updating Azure AD so this is optional in the future. In the meanwhile, you need to tell the OIDC provider to add this.  I am achieving that with the following in my oidc-dp-configmap.yaml: 
```
set_key_use = true
```

### Part 2: Deploy a sample workload and assign it a SPIFFE ID
The source code in the GitHub repo has a sample workload. Create a container image and add it to your image repository. 

Here is an example, which refers to my Azure Container Repository:
```
npm run build
docker build -f deployment\docker\dockerfile -t storefederate .

docker tag storefederate acruday.azurecr.io/storefederate:v1
az acr login -n acruday.azurecr.io

Docker push acruday.azurecr.io/storefederate:v1
```

Now deploy this workload on the cluster. Go to the deployment/spiffe/client folder in the repo. **Make sure you modify the deployment.yaml file to point to your image**
```
kubectl apply -f demo-namespace.yaml
kubectl apply -f serviceaccount.yaml 
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

We will assign a SPIFFE ID to this workload based on its namespace and Kubernetes service account. 

```
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://example.org/ns/demo-spiffe/sa/demo-sa \
  -parentID spiffe://example.org/ns/spire/sa/spire-agent \
  -selector k8s:ns:demo-spiffe \
  -selector k8s:sa:demo-sa
```

Our workload now has been assigned a SPIFFE ID: spiffe://example.org/ns/demo-spiffe/sa/demo-sa

### Part 3: Configure an Azure AD application to trust this SPIFFE ID.
We will add a federated identity credential to our Azure AD application. The two primary components needed to add this are: 
- the issuer URL. The is the OIDC discovery URL we configured earlier. In my demo environment, it is https://demo-oidc.identitydigest.com
- the subject identifier. The SPIFFE ID issued to our workload. (spiffe://example.org/ns/demo-spiffe/sa/demo-sa)


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
{ "name": "AccessusingSpiffe", 
  "issuer": "https://demo-oidc.identitydigest.com", 
  "subject": spiffe://example.org/ns/demo-spiffe/sa/demo-sa",
  "audiences": ["api://AzureADTokenExchange"], 
  "description": "This is meant for workloads in project X" 
}
```
Then create a federated identity credential on the Azure AD application

```
az ad app federated-credential create --id <appid> --parameters credential.json
```


### Part 4: Get a SPIFFE JWT and access Azure resources
The earlier blog posts of [Kubernetes]({% post_url 2022-01-11-azuread-federate-k8s %}) and [Google Cloud]({% post_url 2021-11-11-azuread-federate-gcp %}) showed how you can use an external trusted token to access Azure resources. We will use the same model here. The primary difference is how you get the JWT for your SPIFFE identity.

SPIFFE defines a workload API to get the SVIDs. This API defines how workloads talk to the SPIFFE agent to get either the X.509 certs or the JWTs. [This](https://github.com/spiffe/go-spiffe/blob/main/proto/spiffe/workload/workload.proto) proto file defines the surface of this workload API.

Here's a sample function in Node.JS to get a SPIFFE JWT.

```javascript
async function SPIFFEToken() {
  return new Promise<any>((resolve, reject) => {
    var meta = new grpc.Metadata();
    meta.add('workload.spiffe.io', true);

    this.grpcClient.FetchJWTSVID({audience: ['api://AzureADTokenExchange']}, meta, function(err:any, message:any) {
      if (err) {
        logger.error("spiffe token error %o", err);
        reject(err);
      }
      else {
        logger.debug("spiffe token is %o", message);
        resolve(message.svids[0].svid);
      }   
    });
  });
}
```
You are requesting SPIFFE for a token to identify your workload to Azure AD. Note that the audience here needs to match the audience value you configured on your Azure AD application when setting up the federated identity credential.


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

When requests are made to the blobClient to access storage, the blobClient calls the getToken method on the ClientAssertionCredential. This call results in a request for a fresh token from SPIFFE, which then gets exchanged for an Azure AD access token. The ClientAssertionCredential takes care of caching tokens, so tokens are requested from SPIFFE or Azure AD only when necessary.

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

### Configuring federated credentials programmatically
You can also configure the federated credentials programmatically on both applications and managed identities.

#### Using the Microsoft Graph SDKs for Azure AD application identity
You can use the Microsoft Graph SDKs, available in several languages, to view or modify a federated identity credential. You can choose either the delegated flow or the application-only flow: make sure the service principal you use for this purpose has consent for the Application.ReadWrite.All permissions.

The federatedIdentityCredentials are currently in beta (https://graph.microsoft.com/v1.0/applications/"object-id-of-app"/federatedIdentityCredentials). [The Graph SDK is available in several languages](https://learn.microsoft.com/en-us/graph/sdks/sdks-overview).

#### Using ARM templates for Azure managed identity
You can use [ARM templates, Bicep, or Terraform](https://learn.microsoft.com/en-us/azure/templates/microsoft.managedidentity/userassignedidentities/federatedidentitycredentials?pivots=deployment-language-arm-template) to configure federated identity credentials on a managed identity. 

### Configuring federated credentials on an application using the Azure AD portal

The Azure AD portal has added preview features to simplify this configuration. Visit the Azure AD portal, and pick your application under "App registrations". Go to "Certificated & secrets" and pick "Federated Credentials". Select "Add credential". 
![Azure AD App credential](/images/spiffe-federate/spiffe_start.jpg)

In the "Federated credential scenario", pick "Other issuer".
![Azure AD spiffe scenario](/images/spiffe-federate/spiffe_set.jpg)

Provide your OIDC discovery domain as the issuer URL. The SPIFFE ID is the subject identifier. 
![Azure AD spiffe credential](/images/spiffe-federate/spiffe_done.jpg)


## In conclusion
Azure AD workload identity federation is a new capability that allows you to get rid of secrets in several scenarios such as SPIFFE, Kubernetes, services running in Google Cloud, and GitHub Actions workflow. Stay tuned for many more scenarios where you can use this capability to get rid of secrets.

If you have any comments, feedback, or suggestions on this topic, I would love to hear from you. [DM me on Twitter](https://twitter.com/messages/compose?recipient_id=1446741344)
