- Deployment Guide
    - [Prerequisites](#prerequisites) 
    - [Steps](#Deployment-Steps)
        - [Register AD Application](#1-register-azure-ad-application)
        - [Deploy to Azure subscription](#2-deploy-to-your-azure-subscription)
        - [Set-up Authentication](#3-set-up-authentication)
        - [Add Permissions to your app](#4-add-permissions-to-your-app)
        - [Create the Teams app packages](#5-create-the-teams-app-packages)
        - [Install the apps in Microsoft Teams](#6-install-the-apps-in-microsoft-teams)
    - [Troubleshooting](#troubleshooting)
- - -

# Prerequisites

To begin deployment for Microsoft Teams Emergency Operations Center (TEOC) application, you will need: 
* An Azure subscription where you can create the following kinds of resources:
    * App Service
    * App Service Plan
    * Application Insights
    * App Registration

* You need minimum of contributor access to the Azure subscription to perform the deployment.

- - -

# Deployment Steps

## 1. Register Azure AD application

You need to first create a new Azure AD Application to secure API permissions. Registering your application establishes a trust relationship between your app and the Microsoft identity platform.

1. Log in to the Azure Portal using SharePoint Tenant account, and go to the [App registrations](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps) blade.

1. Click **New registration** to create an Azure AD application.
    - **Name**: Name of your Teams App.
    - **Supported account types**: Select "Accounts in single organizational directory"
    - Leave the "Redirect URI" field blank for now.

    ![Azure AD app registration page](images/App_Registration_1.png)

1. Click **Register** to complete the registration.
1. When the app is registered, you'll be taken to the app's "Overview" page. Copy the **Application (client) Id and Tenant Id**; we will need it later. 

    ![Azure AD app overview page](images/App_Registration_Overview.png)

1. On the side rail in the Manage section, navigate to the "Certificates & secrets" section. In the Client secrets section, click on "+ New client secret". Add a description for the secret, and choose when the secret needs to expire (recommended is 12 months) and click "Add".

1. Once the client secret is created, copy its **Value**; we will need it later. Once you leave this page, you may not be able to copy it again.

    ![Azure AD app secret](images/App_Secret.png)

1. You're done with app registration and client secrets for now. This provides the groundwork for our next steps. Please ensure you have the values ready for Application Id, Tenant Id, Client Secret.

## 2. Deploy to your Azure subscription
1. Click on the **Deploy to Azure** button below.

    [![Deploy to Azure](images/DeployButton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fv-royavinash%2FMIFT-EOC%2Fmain%2FDeployment%2Fazuredeploy.json)

1. When prompted, log in to your Azure subscription.

1. Azure will create a "Custom deployment" based on the TEOC ARM template with pre-filled vlaues.

    > **Note:** Please ensure that you use numbers and lower case in any of the field values otherwise the deployment may fail.

1. Select a subscription and a resource group.
   * We recommend creating a new resource group.
   * The resource group location MUST be in a datacenter that supports the following resources, click [here](https://azure.microsoft.com/en-us/global-infrastructure/services/?products=logic-apps,cognitive-services,search,monitor) to know more about the datacenter and the resources:

     * App Service
     * Application Insights

1. Enter a **Base Resource Name**, which the template uses to generate names for the other resources.
   * The `[Base Resource Name]` must be unique and available. For example, if you select `contosoteoc` as the base name, the name `contosoteoc` must be available (not taken); otherwise, the deployment will fail with a Conflict error.
   * Remember the base resource name that you selected. We will need it later.

1. Update the following fields in the template using the values copied from step 1 in previous section,
    1. **Client ID**: The application (client) ID of the app registered
    2. **User Client Secret**: The client secret of the app registered
    3. **Tenant Id**: The tenant Id

1. Other fields have pre-populated default values, do not change it unless you want it to be customized depending on the need.

    ![Azure App Deploy](images/App_Deploy.png)

1. Click on **Review+Create**. Post successful validation clicks on Create. The deployment will take around 30 to 40 Minutes. Please wait for the deployment to be completed and then proceed with the below steps. 

## 3. Set-up Authentication

1. Go to **App Registrations** page [here](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps) and select the application (TEOC specific) which you created in step 1. Follow the below steps to set up the authentication for the application.

    > NOTE: For following steps you need to use **appDomain** convention for the app service URL.
    - appDomain is the App service URL withouth https:// or <<**resourceBaseName**>>.azurewebsites.net

1. Under **Manage**, click on **Authentication** to bring up authentication settings.

    1. Click on **Add a Platform**. Select Web and add Redirect URIs in below format:
        - **Redirect URI**: Enter `https://%appDomain%/auth-end.html` for the URL (Ex: `https://<<resourceBaseName>>.azurewebsites.net/auth-end.html`)

    1. Click on **Add a Platform**. Select Single-page application and add Redirect URIs in below format:
        - **Redirect URI**: Enter `https://%appDomain%/auth-end.html?clientId= %value%` for the URL (Ex: `https://<<resourceBaseName>>.azurewebsites.net/auth-end.html?clientId=<<Client Id Value>>`)
        - Click on Add URI and add another entry as, `https://%appDomain%/blank-auth-end.html` (Ex: `https://<<resourceBaseName>>.azurewebsites.net/blank-auth-end.html`)     

    1. Click **Save** to commit your changes.

        ![Azure Authentication](images/Setup_Authentication.png)

    1. Back under **Manage**, click on **Expose an API**.

    1. Click on the **Set** link next to **Application ID URI**, and change the value to `api://%appDomain%/%clientId%` (Ex: `api://<<resourceBaseName>>.azurewebsites.net/<<Client Id Value>>`.

    1. Click **Save** to commit your changes.

    1. Click on **Add a scope**, under **Scopes defined by this API**. In the flyout that appears, enter the following values:
        * **Scope name:** access_as_user (_Enter as it is_)
        * **Who can consent?:** Admins and users
        * **Admin consent display name:** Teams can access app’s web APIs
        * **Admin consent description:**  Allows Teams to call the app’s web APIs as the current user.

    1. Click **Add scope** to commit your changes.

        ![API Add Scope](images/Add_Scope.png)

    1. Click **Add a client application**, under **Authorized client applications**. In the flyout that appears, enter the following values:
        * **Client ID**: `5e3ce6c0-2b1f-4285-8d4b-75ee78787346` (_Teams WebApp Client Id_)
        > NOTE: This Id is different than the Client Id from step 1 
        * **Authorized scopes**: Select the scope that ends with `access_as_user`. (There should only be 1 scope in this list.)

    1. Click **Add application** to commit your changes.

    1. **Repeat the previous two steps**, but with below Ids.
        * 1fec8e78-bce4-4aaf-ab1b-5451cc387264 (_Teams Mobile or Desktop App Client Id_)
        * 4345a7b9-9a63-4910-a426-35363201d503 (_Office WebApp Client Id1_)
        * 4765445b-32c6-49b0-83e6-1d93765276ca (_Office WebApp Client Id2_)
        * d3590ed6-52b3-4102-aeff-aad2292ab01c (_Outlook Desktop App Client Id_)
        * 00000002-0000-0ff1-ce00-000000000000 (_Outlook Web App Client Id_)

    1. After this step you should have **6** client applications listed under **Authorized client applications**.

        ![Authorized Apps](images/Authorized_Client_Apps.png)

1. Back under **Manage**, click on **Manifest**.
    1. In the editor that appears, find _accessTokenAcceptedVersion_ and update the value from null to 2
    2. Find the `optionalClaims` property in the JSON Azure AD application manifest, and replace it with the following block:
    ```
        "optionalClaims": {
            "idToken": [],
            "accessToken": [
                {
                    "name": "idtyp",
                    "source": null,
                    "essential": false,
                    "additionalProperties": []
                }
            ],
            "saml2Token": []
        },
    ```    
    3. Click **Save** to commit your changes.

        ![Optional Claims](images/Manifest_Optional_Claims.png)
