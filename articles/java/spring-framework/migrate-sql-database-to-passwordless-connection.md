---
title: Migrate an application to use passwordless connections with Azure SQL Database
description: Learn how to migrate existing applications using Azure SQL Database away from authentication patterns such as passwords to more secure approaches like Managed Identity.
ms.service: sql-database
ms.topic: how-to
author: KarlErickson
ms.author: xiada
ms.date: 09/26/2022
---

# Migrate an application to use passwordless connections with Azure SQL Database

This article explains how to migrate from traditional authentication methods to more secure, passwordless connections with Azure SQL Database.

Application requests to Azure SQL Database must be authenticated. Azure SQL Database provides several different ways for apps to connect securely. One of the ways is to use passwords. However, you should prioritize passwordless connections in your applications when possible.

## Compare authentication options

When authenticating with Azure SQL Database, the application provides a username and password pair to connect to the database. Depending on where the identities are stored, there are two types of authentication: Azure Active Directory (Azure AD) authentication and Azure SQL Database authentication.

### Azure AD authentication

Microsoft Azure AD authentication is a mechanism for connecting to Azure SQL Database using identities defined in Azure AD. With Azure AD authentication, you can manage database user identities and other Microsoft services in a central location, which simplifies permission management.

Using Azure AD for authentication provides the following benefits:

- Authentication of users across Azure Services in a uniform way.
- Management of password policies and password rotation in a single place.
- Multiple forms of authentication supported by Azure AD, which can eliminate the need to store passwords.
- Customers can manage database permissions using external (Azure AD) groups.
- Azure AD authentication uses Azure SQL database users to authenticate identities at the database level.
- Support of token-based authentication for applications connecting to Azure SQL Database.

### Azure SQL Database authentication

You can create accounts in Azure SQL Database. If you choose to use passwords as credentials for the accounts, these credentials will be stored in the `sys.database_principals` table. Because these passwords are stored in Azure SQL Database, you'll need to manage the rotation of the passwords by yourself.

Although it's possible to connect to Azure SQL Database with passwords, you should use them with caution. You must be diligent to never expose the passwords in an unsecure location. Anyone who gains access to the passwords is able to authenticate. For example, if a password is accidentally checked into source control, sent through an unsecure email, pasted into the wrong chat, or viewed by someone who shouldn't have permission, there's risk of a malicious user accessing the application. Instead, consider updating your application to use passwordless connections.

## Introducing passwordless connections

With a passwordless connection, you can connect to Azure services without storing any credentials in the application, its configuration files, or in environment variables.

To ensure that connections are passwordless, you must take into consideration both local development and the production environment. If a password is required in either place, then the application isn't passwordless.

In your local development environment, you can authenticate with Azure CLI, Azure PowerShell, Visual Studio, or Azure plugins for Visual Studio Code or IntelliJ. In this case, you can use that credential in your application instead of configuring properties.

When you deploy applications to an Azure hosting environment, such as a virtual machine, you can enable managed identity in that environment. Then, you won't need to provide credentials to connect to Azure services.

> [!NOTE]
> Since the JDBC driver for Azure SQL Database doesn't support passwordless connections from local environments yet, this article will focus only on applications deployed to Azure hosting environments and how to migrate them to use passwordless connections.
>
> A managed identity provides a security identity to represent an app or service. The identity is managed by the Azure platform and does not require you to provision or rotate any secrets. For more information, see [What are managed identities for Azure resources?](/azure/active-directory/managed-identities-azure-resources/overview)

## Migrate an existing application to use passwordless connections

The following steps explain how to migrate an existing application to use passwordless connections instead of a password-based solution.

### 0) Prepare the working environment

First, use the following command to set up some environment variables.

```bash
export AZ_RESOURCE_GROUP=<YOUR_RESOURCE_GROUP>
export AZ_DATABASE_SERVER_NAME=<YOUR_DATABASE_SERVER_NAME>
export AZ_DATABASE_NAME=demo
export CURRENT_USERNAME=$(az ad signed-in-user show --query userPrincipalName --output tsv)
export CURRENT_USER_OBJECTID=$(az ad signed-in-user show --query id --output tsv)
```

Replace the placeholders with the following values, which are used throughout this article:

- `<YOUR_RESOURCE_GROUP>`: The name of the resource group your resources are in.
- `<YOUR_DATABASE_SERVER_NAME>`: The name of your Azure SQL Database server. It should be unique across Azure.

### 1) Configure Azure SQL Database

#### 1.1) Enable Azure AD-based authentication

To use Azure Active Directory access with Azure SQL Database, you should set the Azure AD admin user first. Only an Azure AD Admin user can create/enable users for Azure AD-based authentication.

If you're using Azure CLI, run the following command to make sure it has sufficient permission:

```bash
az login --scope https://graph.microsoft.com/.default
```

Then, run following command to set the Azure AD admin:

```azurecli
az sql server ad-admin create \
    --resource-group $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --display-name $CURRENT_USERNAME \
    --object-id $CURRENT_USER_OBJECTID
```

This command will set the Azure AD admin to the current signed-in user.

> [!NOTE]
> You can only create one Azure AD admin per Azure SQL Database server. Selection of another one will overwrite the existing Azure AD admin configured for the server.

### 2) Migrate the app code to use passwordless connections

Next, use the following steps to update your code to use passwordless connections. Although conceptually similar, each language uses different implementation details.

#### [Java](#tab/java)

1. Inside your project, add the following reference to the `azure-identity` package. This library contains all of the entities necessary to implement passwordless connections.

   ```xml
   <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>
        <version>1.5.4</version>
   </dependency>
   ```

1. Enable the Azure Active Directory Managed Identity authentication in the JDBC URL.v Identify the locations in your code that currently create a `java.sql.Connection` to connect to Azure SQL Database. Update your code to match the following example:

   ```java
   String url = "jdbc:sqlserver://$AZ_DATABASE_SERVER_NAME.database.windows.net:1433;databaseName=$AZ_DATABASE_NAME;authentication=ActiveDirectoryMSI;"   
   Connection con = DriverManager.getConnection(url);
   ```

1. Replace the two `$AZ_DATABASE_SERVER_NAME` variables and one `$AZ_DATABASE_NAME` variable with the values that you configured at the beginning of this article.

1. Remove the `user` and `password` from the JDBC URL.

#### [Spring](#tab/spring)

1. Inside your project, add a reference to the `spring-cloud-azure-starter` package. This library contains all of the entities necessary to implement passwordless connections.

   ```xml
   <dependency>
       <groupId>com.azure.spring</groupId>
       <artifactId>spring-cloud-azure-starter</artifactId>
       <version>4.5.0-beta.1</version>
   </dependency>
   ```

1. Update the *application.yaml* or *application.properties* file as shown in the following example. Remove `spring.datasource.username` and `spring.datasource.password` properties.

   ```yaml
   spring:
     datasource:
       url: jdbc:sqlserver://${AZ_DATABASE_SERVER_NAME}.database.windows.net:1433;databaseName=${AZ_DATABASE_NAME};authentication=ActiveDirectoryMSI;
   ```

---

### 3) Configure the Azure hosting environment

After your application is configured to use passwordless connections, the same code can authenticate to Azure services after it's deployed to Azure. For example, an application deployed to an Azure App Service instance that has a managed identity enabled can connect to Azure Storage.

In this section, you'll execute two steps to enable your application to run in an Azure hosting environment in a passwordless way:

- Create the managed identity for your Azure hosting environment.
- Assign roles to the managed identity.

> [!NOTE]
> Azure also provides [Service Connector](/azure/service-connector/overview), which can help you connect your hosting service with SQL server. With Service Connector to configure your hosting environment, you can omit the step of assigning roles to your managed identity because Service Connector will do it for you. The following section describes how to configure your Azure hosting environment in two ways: one via Service Connector and the other by configuring each hosting environment directly.

> [!IMPORTANT]
> Service Connector's commands require [Azure CLI](/cli/azure/install-azure-cli) 2.41.0 or above.

#### Create the managed identity using the Azure portal

The following steps show you how to create a system-assigned managed identity for various web hosting services. The managed identity can securely connect to other Azure services using the app configurations you set up previously.

##### [Service Connector](#tab/service-connector)

When using Service Connector, it can help to create the system-assigned managed identity for your Azure hosting environment. However, Azure portal doesn’t support configuring Azure Database this way, so you'll need to use Azure CLI to create the identity.

##### [Azure App Service](#tab/app-service)

1. On the main overview page of your Azure App Service instance, select **Identity** from the navigation pane.

1. On the **System assigned** tab, make sure to set the **Status** field to **on**. A system assigned identity is managed by Azure internally and handles administrative tasks for you. The details and IDs of the identity are never exposed in your code.

   :::image type="content" source="media/passwordless-connections/migration-create-identity.png" alt-text="Screenshot of Azure portal Identity page of App Service resource with System assigned tab showing and Status field highlighted." lightbox="media/passwordless-connections/migration-create-identity.png":::

##### [Azure Container Apps](#tab/container-apps)

1. On the main overview page of your Azure Container App instance, select **Identity** from the navigation pane.

1. On the **System assigned** tab, make sure to set the **Status** field to **on**. A system assigned identity is managed by Azure internally and handles administrative tasks for you. The details and IDs of the identity are never exposed in your code.

   :::image type="content" source="media/passwordless-connections/container-apps-identity.png" alt-text="Screenshot of Azure portal Identity page of Container App resource showing System assigned tab with Status field highlighted." lightbox="media/passwordless-connections/container-apps-identity.png":::

##### [Azure Spring Apps](#tab/spring-apps)

1. On the main overview page of your Azure Spring Apps instance, select **Identity** from the navigation pane.

1. On the **System assigned** tab, make sure to set the **Status** field to **on**. A system assigned identity is managed by Azure internally and handles administrative tasks for you. The details and IDs of the identity are never exposed in your code.

   :::image type="content" source="media/passwordless-connections/spring-apps-identity.png" alt-text="Screenshot of Azure portal Identity page of App resource with System assigned tab showing and Status field highlighted." lightbox="media/passwordless-connections/spring-apps-identity.png":::

##### [Azure virtual machines](#tab/virtual-machines)

1. On the main overview page of your virtual machine, select **Identity** from the navigation pane.

1. On the **System assigned** tab, make sure to set the **Status** field to **on**. A system assigned identity is managed by Azure internally and handles administrative tasks for you. The details and IDs of the identity are never exposed in your code.

   :::image type="content" source="media/passwordless-connections/virtual-machine-identity.png" alt-text="Screenshot of Azure portal Identity page of Virtual machine resource with System assigned tab showing and Status field highlighted." lightbox="media/passwordless-connections/virtual-machine-identity.png":::

---

You can also enable managed identity on an Azure hosting environment using the Azure CLI.

##### [Service Connector](#tab/service-connector-identity)

You can use Service Connector to create a connection between an Azure compute hosting environment and a target service by using the Azure CLI. Service Connector currently supports the following compute services:

- Azure App Service
- Azure Spring Apps
- Azure Container Apps

If you're using Azure App Service, use the `az webapp connection` command, as shown in the following example:

```azurecli
az webapp connection create sql \
    --resource-group $AZ_RESOURCE_GROUP \
    --name <app-service-name>
    --target-resource-group $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --database $AZ_DATABASE_NAME \
    --system-identity
```

If you're using Azure Spring Apps, use `the az spring connection` command, as shown in the following example:

```azurecli
az spring connection create sql \
    --resource-group $AZ_RESOURCE_GROUP \
    --service <service-name> \
    --app <service-instance-name> \
    --target-resource-group $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --database $AZ_DATABASE_NAME \
    --system-identity
```

If you're using Azure Container Apps, use the `az containerapp connection` command, as shown in the following example:

```azurecli
az containerapp connection create sql \
    --resource-group $AZ_RESOURCE_GROUP \
    --name <app-service-name>
    --target-resource-group $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --database $AZ_DATABASE_NAME \
    --system-identity
```

##### [Azure App Service](#tab/app-service-identity)

You can assign a managed identity to an Azure App Service instance with the [az webapp identity assign](/cli/azure/webapp/identity) command, as shown in the following example:

```azurecli
AZ_MI_OBJECT_ID=$(az webapp identity assign \
    --resource-group $AZ_RESOURCE_GROUP \
    --name <service-instance-name> \
    --query principalId \
    --output tsv)
```

##### [Azure Container Apps](#tab/container-apps-identity)

You can assign a managed identity to an Azure Container Apps instance with the [az containerapp identity assign](/cli/azure/containerapp/identity) command, as shown in the following example:

```azurecli
AZ_MI_OBJECT_ID=$(az containerapp identity assign \
    --resource-group $AZ_RESOURCE_GROUP \
    --name <service-instance-name> \
    --query principalId \
    --output tsv)
```

##### [Azure Spring Apps](#tab/spring-apps-identity)

You can assign a managed identity to an Azure Spring Apps instance with the [az spring app identity assign](/cli/azure/spring/app/identity) command, as shown in the following example:

```azurecli
AZ_MI_OBJECT_ID=$(az spring app identity assign \
    --resource-group $AZ_RESOURCE_GROUP \
    --name <service-instance-name> \
    --service <service-name> \
    --query identity.principalId \
    --output tsv)
```

##### [Azure virtual machines](#tab/virtual-machines-identity)

You can assign a managed identity to a virtual machine with the [az vm identity assign](/cli/azure/vm/identity) command, as shown in the following example:

```azurecli
AZ_MI_OBJECT_ID=$(az vm identity assign \
    --resource-group $AZ_RESOURCE_GROUP \
    --name <service-instance-name> \
    --query principalId \
    --output tsv)
```

##### [Azure Kubernetes Service](#tab/aks-identity)

You can assign a managed identity to an Azure Kubernetes Service (AKS) instance with the [az aks update](/cli/azure/aks) command, as shown in the following example:

```azurecli
AZ_MI_OBJECT_ID=$(az aks update \
    --resource-group $AZ_RESOURCE_GROUP \
    --name <AKS-cluster-name> \
    --enable-managed-identity \
    --query identityProfile.kubeletidentity.objectId \
    --output tsv)
```

---

#### Assign roles to the managed identity

Next, grant permissions to the managed identity you created to access your SQL database.

##### [Service Connector](#tab/assign-role-service-connector)

If you connected your services using Service Connector, the previous step's commands already assigned the role, so you can skip this step.

##### [Azure CLI](#tab/assign-role-azure-cli)

This step will create a database user for the managed identity and grant read and write permissions to it.

The following command will retrieve the display name of the managed identity and construct the commands to create a user for the managed identity and grant permissions:

```bash
AZ_DATABASE_AD_MI_USERNAME=$(az ad sp show --id $AZ_MI_OBJECT_ID --query displayName --output tsv)
cat << EOF
CREATE USER "$AZ_DATABASE_AD_MI_USERNAME" FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER "$AZ_DATABASE_AD_MI_USERNAME";
ALTER ROLE db_datawriter ADD MEMBER "$AZ_DATABASE_AD_MI_USERNAME";
GO
EOF
```

Copy the output of this command, then go to the Azure portal and find the SQL database. Sign in to the query editor as the Azure AD admin user, as shown in the following screenshot.

:::image type="content" source="media/migrate-sql-database-to-passwordless-connection/sql-database-query-editor.jpg" alt-text="Screenshot of Azure portal showing the SQL Database query editor.":::

Past the output of last command into the query editor, and run the SQL commands, as shown in the following screenshot.

:::image type="content" source="media/migrate-sql-database-to-passwordless-connection/sql-database-create-user.jpg" alt-text="Screenshot of Azure portal showing SQL Database query editor with query to create user and add roles.":::

---

#### Test the app

After making these code changes, you can build and redeploy the application. Then, browse to your hosted application in the browser. Your app should be able to connect to the Azure SQL database successfully. Keep in mind that it may take several minutes for the role assignments to propagate through your Azure environment. Your application is now configured to run both locally and in a production environment without the developers having to manage secrets in the application itself.

## Next steps

In this tutorial, you learned how to migrate an application to passwordless connections.

You can read the following resources to explore the concepts discussed in this article in more depth:

- [Authorize access to blob data with managed identities for Azure resources](/azure/storage/blobs/authorize-managed-identity).
- [Authorize access to blobs using Azure Active Directory](/azure/storage/blobs/authorize-access-azure-active-directory)