---
title: Certificate rotation for Azure Database for MariaDB
description: Learn about the upcoming changes of root certificate changes that will affect Azure Database for MariaDB
author: mksuni
ms.author: sumuth
ms.service: mariadb
ms.topic: conceptual
ms.date: 01/15/2021
---

# Understanding the changes in the Root CA change for Azure Database for MariaDB

Azure Database for MariaDB will be changing the root certificate for the client application/driver enabled with SSL, use to [connect to the database server](concepts-connectivity-architecture.md). The root certificate currently available is set to expire February 15, 2021 (02/15/2021) as part of standard maintenance and security best practices. This article gives you more details about the upcoming changes, the resources that will be affected, and the steps needed to ensure that your application maintains connectivity to your database server.

>[!NOTE]
> Based on the feedback from customers we have extended the root certificate deprecation for our existing Baltimore Root CA from October 26th, 2020 till February 15, 2021. We hope this extension provide sufficient lead time for our users to implement the client changes if they are impacted.

> [!NOTE]
> Bias-free communication
>
> Microsoft supports a diverse and inclusionary environment. This article contains references to the words _master_ and _slave_. The Microsoft [style guide for bias-free communication](https://github.com/MicrosoftDocs/microsoft-style-guide/blob/master/styleguide/bias-free-communication.md) recognizes these as exclusionary words. The words are used in this article for consistency because they are currently the words that appears in the software. When the software is updated to remove the words, this article will be updated to be in alignment.
>

## What update is going to happen?

In some cases, applications use a local certificate file generated from a trusted Certificate Authority (CA) certificate file to connect securely. Currently customers can only use the predefined certificate to connect to an Azure Database for MariaDB server, which is located [here](https://www.digicert.com/CACerts/BaltimoreCyberTrustRoot.crt.pem). However, [Certificate Authority (CA) Browser forum](https://cabforum.org/) recently published reports of multiple certificates issued by CA vendors to be non-compliant.

As per the industry's compliance requirements, CA vendors began revoking CA certificates for non-compliant CAs, requiring servers to use certificates issued by compliant CAs, and signed by CA certificates from those compliant CAs. Since Azure Database for MariaDB currently uses one of these non-compliant certificates, which client applications use to validate their SSL connections, we need to ensure that appropriate actions are taken (described below) to minimize the potential impact to your MariaDB servers.

The new certificate will be used starting February 15, 2021 (02/15/2021). If you use either CA validation or full validation of the server certificate when connecting from a MySQL client (sslmode=verify-ca or sslmode=verify-full), you need to update your application configuration before February 15, 2021 (02/15/2021).

## How do I know if my database is going to be affected?

All applications that use SSL/TLS and verify the root certificate needs to update the root certificate. You can identify whether your connections verify the root certificate by reviewing your connection string.

- If your connection string includes `sslmode=verify-ca` or `sslmode=verify-identity`, you need to update the certificate.
- If your connection string includes `sslmode=disable`, `sslmode=allow`, `sslmode=prefer`, or `sslmode=require`, you don't need to update certificates.
- If your connection string doesn't specify sslmode, you don't need to update certificates.

If you're using a client that abstracts the connection string away, review the client's documentation to understand whether it verifies certificates.
To understand Azure Database for MariaDB sslmode, review the [SSL mode descriptions](concepts-ssl-connection-security.md#default-settings).

To avoid your application's availability being interrupted as a result of certificates being unexpectedly revoked, or to update a certificate, which has been revoked, refer to the [**"What do I need to do to maintain connectivity"**](concepts-certificate-rotation.md#what-do-i-need-to-do-to-maintain-connectivity) section.

## What do I need to do to maintain connectivity

To avoid your application's availability being interrupted due to certificates being unexpectedly revoked, or to update a certificate, which has been revoked, follow the steps below. The idea is to create a new *.pem* file, which combines the current cert and the new one and during the SSL cert validation after of the allowed values will be used. Refer to the steps below:

- Download **BaltimoreCyberTrustRoot** & **DigiCertGlobalRootG2** CA from links below:

  - [https://www.digicert.com/CACerts/BaltimoreCyberTrustRoot.crt.pem](https://www.digicert.com/CACerts/BaltimoreCyberTrustRoot.crt.pem)
  - [https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem](https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem)

- Generate a combined CA certificate store with both **BaltimoreCyberTrustRoot** and **DigiCertGlobalRootG2** certificates are included.

  - For Java (MariaDB Connector/J) users, execute:

    ```console
    keytool -importcert -alias MariaDBServerCACert  -file D:\BaltimoreCyberTrustRoot.crt.pem  -keystore truststore -storepass password -noprompt
    ```

    ```console
    keytool -importcert -alias MariaDBServerCACert2  -file D:\DigiCertGlobalRootG2.crt.pem -keystore truststore -storepass password  -noprompt
    ```

    Then replace the original keystore file with the new generated one:

    - System.setProperty("javax.net.ssl.trustStore","path_to_truststore_file");
    - System.setProperty("javax.net.ssl.trustStorePassword","password");

  - For .NET (MariaDB Connector/NET, MariaDBConnector) users, make sure **BaltimoreCyberTrustRoot** and **DigiCertGlobalRootG2** both exist in Windows Certificate Store, Trusted Root Certification Authorities. If any certificates don't exist, import the missing certificate.

    ![Azure Database for MariaDB .net cert](media/overview/netconnecter-cert.png)

  - For .NET users on Linux using SSL_CERT_DIR, make sure **BaltimoreCyberTrustRoot** and **DigiCertGlobalRootG2** both exist in the directory indicated by SSL_CERT_DIR. If any certificates don't exist, create the missing certificate file.

  - For other (MariaDB Client/MariaDB Workbench/C/C++/Go/Python/Ruby/PHP/NodeJS/Perl/Swift) users, you can merge two CA certificate files like this format below</b>

    </br>-----BEGIN CERTIFICATE-----
    </br>(Root CA1: BaltimoreCyberTrustRoot.crt.pem)
    </br>-----END CERTIFICATE-----
    </br>-----BEGIN CERTIFICATE-----
    </br>(Root CA2: DigiCertGlobalRootG2.crt.pem)
    </br>-----END CERTIFICATE-----

- Replace the original root CA pem file with the combined root CA file and restart your application/client.
- In future, after the new certificate deployed on the server side, you can change your CA pem file to DigiCertGlobalRootG2.crt.pem.

## What can be the impact of not updating the certificate?

If you're using the Azure Database for MariaDB issued certificate as documented here, your application's availability might be interrupted since the database won't be reachable. Depending on your application, you may receive various error messages including but not limited to:

- Invalid certificate/revoked certificate
- Connection timed out

> [!NOTE]
> Please do not drop or alter **Baltimore certificate** until the cert change is made. We will send a communication after the change is done, after which it is safe for them to drop the Baltimore certificate.

## Frequently asked questions

### 1. If I'm not using SSL/TLS, do I still need to update the root CA?

No actions required if you aren't using SSL/TLS.

### 2. If I'm using SSL/TLS, do I need to restart my database server to update the root CA?

No, you don't need to restart the database server to start using the new certificate. Certificate update is a client-side change, and the incoming client connections need to use the new certificate to ensure that they can connect to the database server.

### 3. What will happen if I don't update the root certificate before February 15, 2021 (02/15/2021)?

If you don't update the root certificate before February 15, 2021 (02/15/2021), your applications that connect via SSL/TLS and does verification for the root certificate will be unable to communicate to the MariaDB database server and application will experience connectivity issues to your MariaDB database server.

### 4. What is the impact if using App Service with Azure Database for MariaDB?

For Azure app services, connecting to Azure Database for MariaDB, there are two possible scenarios depending on how on you're using SSL with your application.

- This new certificate has been added to App Service at platform level. If you're using the SSL certificates included on App Service platform in your application, then no action is needed.
- If you're explicitly including the path to SSL cert file in your code, then you would need to download the new cert and update the code to use the new cert. A good example of this scenario is when you use custom containers in App Service as shared in the [App Service documentation](../app-service/tutorial-multi-container-app.md#configure-database-variables-in-wordpress)

### 5. What is the impact if using Azure Kubernetes Services (AKS) with Azure Database for MariaDB?

If you're trying to connect to the Azure Database for MariaDB using Azure Kubernetes Services (AKS), it's similar to access from a dedicated customers host environment. Refer to the steps [here](../aks/ingress-own-tls.md).

### 6. What is the impact if using Azure Data Factory to connect to Azure Database for MariaDB?

For connector using Azure Integration Runtime, the connector uses certificates in the Windows Certificate Store in the Azure-hosted environment. These certificates are already compatible to the newly applied certificates and so no action is needed.

For connector using Self-hosted Integration Runtime where you explicitly include the path to SSL cert file in your connection string, you'll need to download the [new certificate](https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem) and update the connection string to use it.

### 7. Do I need to plan a database server maintenance downtime for this change?

No. Since the change here's only on the client side to connect to the database server, there's no maintenance downtime needed for the database server for this change.

### 8.  What if I can't get a scheduled downtime for this change before February 15, 2021 (02/15/2021)?

Since the clients used for connecting to the server needs to be updating the certificate information as described in the fix section [here](./concepts-certificate-rotation.md#what-do-i-need-to-do-to-maintain-connectivity), we don't need to a downtime for the server in this case.

### 9. If I create a new server after February 15, 2021 (02/15/2021), will I be impacted?

For servers created after February 15, 2021 (02/15/2021), you can use the newly issued certificate for your applications to connect using SSL.

### 10. How often does Microsoft update their certificates or what is the expiry policy?

These certificates used by Azure Database for MariaDB are provided by trusted Certificate Authorities (CA). So the support of these certificates on Azure Database for MariaDB is tied to the support of these certificates by CA. However, as in this case, there can be unforeseen bugs in these predefined certificates, which need to be fixed at the earliest.

### 11. If I'm using read replicas, do I need to perform this update only on source server or the read replicas?

Since this update is a client-side change, if the client used to read data from the replica server, you'll need to apply the changes for those clients as well.

### 12. If I'm using Data-in replication, do I need to perform any action?

If you're using [Data-in replication](concepts-data-in-replication.md) to connect to Azure Database for MySQL, there are two things to consider:

- If the data-replication is from a virtual machine (on-prem or Azure virtual machine) to Azure Database for MySQL, you need to check if SSL is being used to create the replica. Run **SHOW SLAVE STATUS** and check the following setting. 

  ```azurecli-interactive
  Master_SSL_Allowed            : Yes
  Master_SSL_CA_File            : ~\azure_mysqlservice.pem
  Master_SSL_CA_Path            :
  Master_SSL_Cert               : ~\azure_mysqlclient_cert.pem
  Master_SSL_Cipher             :
  Master_SSL_Key                : ~\azure_mysqlclient_key.pem
  ```

    If you do see the certificate is provided for the CA_file, SSL_Cert and SSL_Key, you will need to update the file by adding the [new certificate](https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem).

- If the data-replication is between two Azure Database for MySQL, then you'll need to reset the replica by executing
**CALL mysql.az_replication_change_master** and provide the new dual root certificate as last parameter [master_ssl_ca](howto-data-in-replication.md#link-the-source-and-replica-servers-to-start-data-in-replication).

### 13. Do we have server-side query to verify if SSL is being used?

To verify if you're using SSL connection to connect to the server refer [SSL verification](howto-configure-ssl.md#verify-the-ssl-connection).

### 14. Is there an action needed if I already have the DigiCertGlobalRootG2 in my certificate file?

No. There's no action needed if your certificate file already has the **DigiCertGlobalRootG2**.

### 15. What if I have further questions?

If you have questions, get answers from community experts in [Microsoft Q&A](mailto:AzureDatabaseformariadb@service.microsoft.com). If you have a support plan and you need technical help, [contact us](mailto:AzureDatabaseformariadb@service.microsoft.com).
