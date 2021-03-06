---
services: app-service, servicebus
platforms: dotnet
author: nonik0
---

# Use Service Bus from App Service with Managed Service Identity

## Background
For authenticating from applications to Service Bus, the approach so far involved creating an application and an associated shared access key in Service Bus, and then using that shared access key to authenticate directly with Service Bus. While this approach works well, there are two shortcomings:
1. The application's shared access key is typically hard coded in source code or configuration files. Developers tend to push the code to source repositories as-is, which leads to credentials in source.
2. The application's shared access key never expires.

With [Managed Service Identity (MSI)](https://docs.microsoft.com/en-us/azure/app-service/app-service-managed-service-identity), both these problems are solved. This sample shows how a Web App can authenticate to Azure Service Bus without the need to explicitly create and manage a Service Bus shared access key.

## Prerequisites
To run and deploy this sample, you need the following:
1. An Azure subscription to create an App Service and a Service Bus Namespace. 
2. [Visual Studio 2017 v15.6](https://blogs.msdn.microsoft.com/visualstudio/2017/12/07/visual-studio-2017-version-15-6-preview/) or later.

## Step 1: Create an App Service with a Managed Service Identity (MSI)
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fnonik0%2Fapp-service-msi-servicebus-dotnet%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

Use the "Deploy to Azure" button to deploy an ARM template to create the following resources:
1. App Service with MSI.
2. Service Bus Namespace with a Service Bus Queue

Be sure to create a Service Bus Namespace in one of the Azure regions that have preview support for role-based access control (RBAC): **East US**, **East US 2**, or **West Europe**. 

After using the ARM template to deploy the needed resources, review the resources created using the Azure portal. You should see an App Service and a Service Bus Namespace that contains a Service Bus Queue. You can verify your App Service has Managed Service Identity (MSI) enabled by navigating to the App Service, clicking on "Managed service identity (Preview)" in the left-hand menu, and verifying "Register with Azure Active Directory" is set to "on".

## Step 2: Grant yourself access to the Service Bus Namespace
Using the Azure Portal, go to the Service Bus Namespace's access control tab, and grant yourself **Contributor** access to the Service Bus Namespace. This will allow you to run the application on your local development machine (**Note**: There is currently an issue where inherited permissions will not grant access; you must explicity add permissions here).

To grant access:
1.	Search for your Service Bus Namespace in “Search Resources dialog box” in Azure Portal.
2.	In the left-hand menu, select "Access control (IAM)"
3.	Click on "Add", then select "Contributor" from the dropdown for "Role"
4.	Click on the "Select" field, then search for and select your user account/email
5.	Click on "Save" to complete adding your user account as a new "Contributor" for your Service Bus Namespace

## Step 3: Clone the repo 
Clone the repo to your development machine. 

The project has two relevant Nuget packages:
1. Microsoft.Azure.Services.AppAuthentication - makes it easy to fetch access tokens for Service-to-Azure-Service authentication scenarios. 
2. Microsoft.Azure.ServiceBus - contains methods for interacting with Service Bus. 

The relevant code is in WebAppServiceBus/WebAppServiceBus/Controllers/HomeController.cs file. The **Send** method will create an instance of a Service Bus QueueClient, configured to use a ManagedServiceIdentityTokenProvider. This TokenProvider's implementation uses the AzureServiceTokenProvider found in the Microsoft.Azure.Services.AppAuthentication library. AzureServiceTokenProvider will follow a set number of different methods, depending on the environment, to get an access token. Please refer to the AppAuthentication library's documentation for [local development authentication](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#local-development-authentication) to see more detail.

```csharp    
var tokenProvider = TokenProvider.CreateManagedServiceIdentityTokenProvider();
QueueClient sendClient = new QueueClient($"sb://{Config.Namespace}.servicebus.windows.net/", Config.Queue, tokenProvider);
await sendClient.SendAsync(new Message(Encoding.UTF8.GetBytes(messageInfo.MessageToSend)));
await sendClient.CloseAsync();
```

## Step 4: Update the Service Bus Namespace and Service Bus Queue names
In the Web.config file, change the Service Bus Namespace and Queue to the ones you just created. Replace **ServiceBusNamespace** with the name of your Service Bus Namespace and **ServiceBusQueue** with the name of your Service Bus Queue.

## Step 5: Run the application on your local development machine
When running your sample, the previously-configured QueueClient will use the ManagedServiceIdentityTokenProvider, which uses the developer's security context to get a token to authenticate to Service Bus. This removes the need to create a Service Bus shared access key and share it with the development team. It also prevents credentials from being checked in to source code. For local development, the ManagedServiceIdentityTokenProvider will use **Visual Studio**, **Azure CLI**, or **Active Directory Integrated Authentication** to authenticate to Azure AD to get a token. That token will be used to both send and receive data from your Service Bus Queue.

Visual Studio authentication will work if the following conditions are met:
 1. You have installed [Visual Studio 2017 v15.6](https://blogs.msdn.microsoft.com/visualstudio/2017/12/07/visual-studio-2017-version-15-6-preview/) or later.
 2. You are signed in to Visual Studio and have selected an account to use for local development. Use Tools > Options > Azure Service Authentication to choose a local development account. 

Azure CLI will work if the following conditions are met:
 1. You have [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed. Version 2.0.12 supports the get-access-token option used by AzureServiceTokenProvider. If you have an earlier version, please upgrade. 
 2. You are logged into Azure CLI. You can login using **az login** command.
 
Azure Active Directory authentication will only work if the following conditions are met:
 1. Your on-premise active directory is synced with Azure AD. 
 2. You are running this code on a domain joined machine.   

Since your developer account has access to the Service Bus Namespace, you should be able to send and receive data on your Service Bus Queue using the web app's interface.

You can also use a service principal to run the application on your local development machine. See the section "Running the application using a service principal" in the [documentation for the AppAuthentication library](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#running-the-application-using-a-service-principal) for how to do this.
>Note: It is recommended to use your developer context for local development, since you do not need to create or share a service principal for that. If that does not work for you, you can use a service principal, but do not check in the certificate or secret in source repos, and share them securely.

## Step 6: Grant App Service MSI access to the Service Bus Namespace
Follow the steps in Step 2 used to grant yourself "Contributor" access to the Service Bus Namespace. However, instead of granting "Contributor" access to yourself, instead grant "Contributor" access to the App Service's MSI. To find the App Service's MSI in the "Select" field when adding a new role to the Service Bus Namespace, type in the name of the app service you used when deploying in Step 1.

## Step 7: Deploy the Web App to Azure
Use any of the methods outlined on [Deploy your app to Azure App Service](https://docs.microsoft.com/en-us/azure/app-service-web/web-sites-deploy) to publish the Web App to Azure. 
After you deploy it, browse to the web app. You should be able to send and receive data on your Service Bus Namespace on the web page as you did when you locally deployed the web app in Step 5. However, different from Step 5, the QueueClient using the ManagedServiceIdentityTokenProvider will use the web app's own Managed Service Identity (MSI) to authenticate to Service Bus instead of your local developer context. This did not require any code changes between local development and being published to Azure.

## Summary
The web app was successfully able to send and receive data on your Service Bus Namespace using your developer account during development, and using MSI when deployed to Azure, without any code change between local development environment and Azure. 
As a result, you did not have to explicitly create and handle a Service Bus shared access key to authenticate to and call Service Bus. You do not have to worry about renewing the MSI's credentials either, since MSI takes care of that.  

## Troubleshooting

### Common issues during local development:

1. Azure CLI is not installed, or you are not logged in, or you do not have the latest version. 
Run **az account get-access-token** to see if Azure CLI shows a token for you. If it says no such program found, please install Azure CLI 2.0. If you have installed it, you may be prompted to login. 

2. AzureServiceTokenProvider cannot find the path for Azure CLI.
AzureServiceTokenProvider finds Azure CLI at its default install locations. If it cannot find Azure CLI, please set environment variable **AzureCLIPath** to the Azure CLI installation folder. AzureServiceTokenProvider will add the environment variable to the Path environment variable.

3. You are logged into Azure CLI using multiple accounts, or the same account has access to subscriptions in multiple tenants. You get an Access Denied error when trying to make calls to Service Bus during local development. 
Using Azure CLI, set the default subscription to one which has the account you want use, and is in the same tenant as your Service Bus Namespace: **az account set --subscription [subscription-id]**. If no output is seen, then it succeeded. Verify the right account is now the default using **az account list**.

### Common issues when deployed to Azure App Service:

1. MSI is not setup on the App Service. 

Check the environment variables MSI_ENDPOINT and MSI_SECRET exist using [Kudu debug console](https://azure.microsoft.com/en-us/resources/videos/super-secret-kudu-debug-console-for-azure-web-sites/). If these environment variables do not exist, MSI is not enabled on the App Service. 

### Common issues across environments:

1. System.UnauthorizedAccessException: Failed to authorize request after RBAC check. 

The principal used does not have access to the Service Bus Namespace. Grant either your user account or the App Service's MSI "Contributor" access to the Service Bus Namespace, depending on whether you are running the sample on your local development machine or deployed in Azure to your App Service.
