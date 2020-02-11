---
title: Resource specific consent in Teams
description: Describes resource specific consent in Teams and how to make advantage of it.
keywords: teams authorization OAuth SSO AAD rsc
---
# Resource specific consent in Teams
Resource-specific consent (RSC) in Microsoft Teams allows team owners to grant consent to an app to access their team's data and replaces the need for a global admin to approve the app with [tenant-wide admin permissions](/azure/active-directory/manage-apps/grant-admin-consent).
These permissions include the ability to create, rename and delete channels, read channel messages, create tabs, and add or remove members from a team. This article provides a step-by-step guide to configure RSC in your Teams app.
> [!Note]
> To provide consent to your app, the team owner must firstly install the Teams app. Webpages running outside Teams can make RSC Graph calls only after the corresponding Teams app has been installed.

> [!Note]
> When you make an RSC API call, Graph doesn't know on behalf of which user (if any) you are doing the work. Hence, you should be aware of the use cases such as creating a channel or removing a tab, as you are calling Application permissions rather than delegated permissions.


![Consent screen](~/assets/images/rsc/rsc-consentscreen.md)
<!--![Consent screen.](../../assets/images/rsc/rsc-consentscreen.md)-->

## Check admin settings

<!--As an admin, you can control whether team owners in your organization can grant team-specific consent (enabled by default) through [settings](https://review.docs.microsoft.com/en-us/MicrosoftTeams/resource-specific-consent?branch=v-lanac-rsc) that you configure in the Azure Active Directory (Azure AD) PowerShell module or the Azure portal and the Microsoft Teams admin center.-->
Before you configure RSC for your AAD app, ensure these [settings](https://review.docs.microsoft.com/en-us/MicrosoftTeams/resource-specific-consent?branch=v-lanac-rsc) are enabled by your tenant administrator.

> **Important**: Changing any of these settings doesn't affect data access for apps that were already granted consent. 

## Configure Teams app to support RSC (Developer)

The basic steps required to configure a service and get a token from the Microsoft identity platform endpoint that your service can use to call Microsoft Graph under its own identity are:

1. Register your application in Azure AD.
2. Check permissions configured in the 'API Permissions' section.
3. Update your Teams app manifest to link to your AAD app ID.
4. Get an access token.
5. Make a Graph call.


#### 1. Register a new application in Azure AD

[Register](/graph/auth-register-app-v2) a new application in the [Azure Active Directory- App Registration portal](https://ms.portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade) to get an AAD app ID. If your AAD app supports [single sign-on (SSO) for tabs](/microsoftteams/platform/tabs/how-to/authentication/auth-aad-sso), then use the same AAD app ID for configuring RSC.
>[!Note]
>It is not a good idea to have multiple Teams apps link to the same AAD app ID. Team AAD app ID must be unique – if anyone tries to install and link multiple Teams app to the same AAD app ID, the installation will fail.

#### 2. Check permissions configured in the 'API Permissions' section
RSC permissions are only configured in the app manifest file. However, we need to check permissions in the 'API permissions' section in the app registration portal. Navigate to the 'API Permissions' section to check for any RSC and non-RSC permissions. If your app is newly registered, it may not have any additional permissions configured. Hence you can proceed further to step 3 to configure RSC permissions in the app manifest file. On the other side, if your app is an existing app, it may have additional permissions configured. In this case, ensure there are no RSC permissions configured here as we will configure them in the app manifest file as part of step 3. You can retain the non-RSC permissions like part of any normal app registration process.

E.g. Graph permissions are not always about Teams data.
For example, Your AAD app may need Mail.Read permission, but this is not an RSC scenario because it is not about team data, and team owners cannot give consent to other's mailboxes. APIs for Files, SharePoint, OneNote, Planner, and Calendar within a team do not support RSC yet.

#### 3. Update your Teams app manifest to link to your AAD app ID
Add a [webApplicationInfo](/microsoftteams/platform/resources/schema/manifest-schema#webapplicationinfo) section to the manifest that includes two properties - `id` is the AAD app ID, and `applicationPermissions` are the RSC permissions (scopes) your app requires.

> **Note**: If your app needs non-RSC permissions, you can continue to use them as always, but they do not go in the manifest. 

```
{ 
  "$schema": ".../MicrosoftTeams.schema.json", 
   "webApplicationInfo": { 
      "id": "cb38cf54-ac89-4a7a-9ea3-095d3d080037", 
      "applicationPermissions": [  
      "File.Read.Group",
      "TeamSettings.Read.Group",
      "ChannelMessage.Read.Group",
      "Member.Read.Group"
      ] 
  }
  ... 
} 
```
RSC permissions are the ones that end with .Group suffix. For e.g. TeamSettings.Read.Group, ChannelSettings.Read.Group. 
Refer to the list of available Graph permissions for RSC in <a href="https://docs.microsoft.com/en-us/graph/permissions-reference?context=graph%2Fapi%2Fbeta&view=graph-rest-beta">Teams permissions</a> section.

#### 4. Get an access token 

Before you make a REST call to the Graph, you need to [get an access token](/graph/api/resources/teams-api-overview?view=graph-rest-beta) for the application permissions similar to getting an application permission token for non-RSC use.
You specify the pre-configured permissions by passing `https://graph.microsoft.com/.default` as the value for the `scope` parameter in the token request. See the `scope` parameter description in the token request below for details.

### Token request

```
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Host: login.microsoftonline.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={appId}
&client_secret={appSecret}
&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default

```

#### Token response

A successful response looks like this:

```JSON
{
  "token_type": "{token_type}",
  "expires_in": {seconds},
  "access_token": "{access_token}"
}
```

  
#### 5. Make a Graph call 

You can now make a REST call like a non-RSC call or use the SDK. See [Microsoft Graph](https://developer.microsoft.com/en-us/graph) and [Use the Microsoft Graph API to work with Microsoft Teams](/graph/api/resources/teams-api-overview?view=graph-rest-beta) for more details. 
