---
ms.date: 09/26/2022
author: KarlErickson
ms.author: v-muyaofeng
---

## PostgreSQL support

[Azure Database for PostgreSQL](https://azure.microsoft.com/services/postgresql/) is a relational database service based on the open-source Postgres database engine. It's a fully managed database-as-a-service that can handle mission-critical workloads with predictable performance, security, high availability, and dynamic scalability.

From version `4.5.0-beta.1`, Spring Cloud Azure supports various types of credentials for authentication to Azure Database for PostgreSQL single server.

### Supported PostgreSQL version

The current version of the starter should use Azure Database for PostgreSQL Single Server version `10` or `11`.

### Core Features

#### Passwordless connection

Passwordless connection uses Azure Active Directory (Azure AD) authentication for connecting to Azure services without storing any credentials in the application, its configuration files, or in environment variables. Azure AD authentication is a mechanism for connecting to Azure Database for PostgreSQL using identities defined in Azure AD. With Azure AD authentication, you can manage database user identities and other Microsoft services in a central location, which simplifies permission management.

### How it works

Spring Cloud Azure will first build one of the following types of credentials depending on the application authentication configuration:

- `ClientSecretCredential`
- `ClientCertificateCredential`
- `UsernamePasswordCredential`
- `ManagedIdentityCredential`
- `DefaultAzureCredential`

If none of these types of credentials are found, the `DefaultAzureCredential` credentials will be obtained from application properties, environment variables, managed identities, or the IDE. For detailed information, see the [Spring Cloud Azure authentication](#spring-cloud-azure-authentication) section.

The following high-level diagram summarizes how authentication works using OAuth credential authentication with Azure Database for PostgreSQL. The arrows indicate communication pathways.

:::image type="content" source="../../media/spring-cloud-azure/authentication-postgresql-azure-active-directory.png" alt-text="Diagram showing Azure Active Directory authentication for PostgreSQL ." border="false":::

### Configuration

Spring Cloud Azure for PostgreSQL supports the following two levels of configuration options:

1. The global authentication configuration options of `credential` and `profile` with prefixes of `spring.cloud.azure`.

1. Spring Cloud Azure for PostgreSQL common configuration options.

The following table shows the Spring Cloud Azure for PostgreSQL common configuration options:

> [!div class="mx-tdBreakAll"]
> | Name                                                                  | Description                                                                                                                                                                                            |
> |-----------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
> | spring.datasource.azure.passwordless-enabled                          | Whether to enable passwordless connections to Azure databases by using OAuth2 Azure Active Directory token credentials.                                                                                |
> | spring.datasource.azure.credential.client-certificate-password        | Password of the certificate file.                                                                                                                                                                      |
> | spring.datasource.azure.credential.client-certificate-path            | Path of a PEM certificate file to use when performing service principal authentication with Azure.                                                                                                     |
> | spring.datasource.azure.credential.client-id                          | Client ID to use when performing service principal authentication with Azure. This is a legacy property.                                                                                               |
> | spring.datasource.azure.credential.client-secret                      | Client secret to use when performing service principal authentication with Azure. This is a legacy property.                                                                                           |
> | spring.datasource.azure.credential.managed-identity-enabled           | Whether to enable managed identity to authenticate with Azure. If *true* and the `client-id` is set, will use the client ID as user assigned managed identity client ID. The default value is *false*. |
> | spring.datasource.azure.credential.password                           | Password to use when performing username/password authentication with Azure.                                                                                                                           |
> | spring.datasource.azure.credential.username                           | Username to use when performing username/password authentication with Azure.                                                                                                                           |
> | spring.datasource.azure.profile.cloud-type                            | Name of the Azure cloud to connect to.                                                                                                                                                                 |
> | spring.datasource.azure.profile.environment.active-directory-endpoint | The Azure Active Directory endpoint to connect to.                                                                                                                                                     |
> | spring.datasource.azure.profile.tenant-id                             | Tenant ID for Azure resources.                                                                                                                                                                         |

### Dependency setup

Add the following dependency to your project. This will automatically include the `spring-boot-starter` dependency in your project transitively.

```xml
<dependency>
    <groupId>com.azure.spring</groupId>
    <artifactId>spring-cloud-azure-starter-jdbc-postgresql</artifactId>
</dependency>
```

> [!NOTE]
> If you want to use passwordless connections, you must add dependency version `4.5.0-beta.1`.
>
> Remember to add the BOM `spring-cloud-azure-dependencies` along with the above dependency. For more information, see the [Getting started](#getting-started) section.

### Basic usage

The following sections show the classic Spring Boot application usage scenarios.

> [!IMPORTANT]
> Passwordless connection uses Azure AD authentication. To use Azure AD authentication, you should set the Azure AD admin user first. Only an Azure AD Admin user can create and enable users for Azure AD-based authentication. For more information, see the [Create a PostgreSQL server and set up admin user](../../configure-spring-data-jdbc-with-azure-postgresql.md?branch=release-cred-free-java&tabs=passwordless#create-a-postgresql-server-and-set-up-admin-user) section.

#### Connect to Azure PostgreSQL locally without password

1. To create users and grant permission, see the [Create a PostgreSQL non-admin user and grant permission](../../configure-spring-data-jdbc-with-azure-postgresql.md?branch=release-cred-free-java&tabs=passwordless#create-a-postgresql-non-admin-user-and-grant-permission) section.

1. Configure the following properties in your *application.yml* file:

   ```yaml
   spring:
     datasource:
       url: jdbc:postgresql://${AZ_DATABASE_SERVER_NAME}.postgres.database.azure.com:5432/${AZ_DATABASE_NAME}?sslmode=require
       username: ${AZ_POSTGRESQL_AD_NON_ADMIN_USERNAME}@${AZ_DATABASE_SERVER_NAME}
       azure:
         passwordless-enabled: true
   ```

#### Connect to Azure PostgreSQL using a service principal

1. Assign role to service principal:

   1. Create a SQL script called *create_ad_user_sp.sql* for creating a non-admin user. Add the following contents and save it locally:

      > [!IMPORTANT]
      > Make sure `<service-principal-name>` already exists in your Azure AD tenant, or you won't be able to create the non-admin user.

      ```bash
      export AZ_POSTGRESQL_AD_SP_USERNAME=<service-principal-name>

      cat << EOF > create_ad_user_sp.sql
      SET aad_validate_oids_in_tenant = off;
      CREATE ROLE "$AZ_POSTGRESQL_AD_SP_USERNAME" WITH LOGIN IN ROLE azure_ad_user;
      GRANT ALL PRIVILEGES ON DATABASE $AZ_DATABASE_NAME TO "$AZ_POSTGRESQL_AD_SP_USERNAME";
      EOF
      ```

   1. Use the following command to run the SQL script to create the Azure AD non-admin user:

      ```bash
      psql "host=$AZ_DATABASE_SERVER_NAME.postgres.database.azure.com user=$CURRENT_USERNAME@$AZ_DATABASE_SERVER_NAME dbname=$AZ_DATABASE_NAME port=5432 password=`az account get-access-token --resource-type oss-rdbms --output tsv --query accessToken` sslmode=require" < create_ad_user_sp.sql
      ```

   1. Now use the following command to remove the temporary SQL script file:

      ```bash
      rm create_ad_user_sp.sql
      ```

1. Configure the following properties in your *application.yml* file:

   ```yaml
   spring:
     cloud:
       azure:
         credential:
           client-id: ${AZURE_CLIENT_ID}
           client-secret: ${AZURE_CLIENT_SECRET}
         profile:
           tenant-id: ${AZURE_TENANT_ID}
     datasource:
       url: jdbc:postgresql://${AZ_DATABASE_SERVER_NAME}.postgres.database.azure.com:5432/${AZ_DATABASE_NAME}?sslmode=require
       username: ${AZ_POSTGRESQL_AD_SP_USERNAME}@${AZ_DATABASE_SERVER_NAME}
       azure:
         passwordless-enabled: true
   ```

#### Connect to Azure PostgreSQL with Managed Identity in Azure Spring Apps

1. To enable managed identity, see the [Assign the managed identity using the Azure portal](../../migrate-postgresql-to-passwordless-connection.md?branch=release-cred-free-java&tabs=sign-in-azure-cli%2cjava%2cazure-portal%2cspring-apps%2cspring-apps-identity#assign-the-managed-identity-using-the-azure-portal) section.

1. To grant permissions, see the [Assign role to managed identity](../../migrate-postgresql-to-passwordless-connection.md?branch=release-cred-free-java&tabs=sign-in-azure-cli%2cjava%2cazure-portal%2cspring-apps%2cspring-apps-identity#assign-roles-to-the-managed-identity) section.

1. Configure the following properties in your *application.yml* file:

   ```yaml
   spring:
     cloud:
       azure:
         credential:
           managed-identity-enabled: true
           client-id: ${AZURE_CLIENT_ID}
     datasource:
       url: jdbc:postgresql://${AZ_DATABASE_SERVER_NAME}.postgres.database.azure.com:5432/${AZ_DATABASE_NAME}?sslmode=require
       username: ${AZ_POSTGRESQL_AD_MI_USERNAME}@${AZ_DATABASE_SERVER_NAME}
       azure:
         passwordless-enabled: true
   ```

> [!NOTE]
> For more information, see [Tutorial: Deploy a Spring application to Azure Spring Apps with a passwordless connection to an Azure database](../../deploy-passwordless-spring-database-app.md)