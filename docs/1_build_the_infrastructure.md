## 🛠️ Build the Azure Databricks Infrastructure

To build the infrastructure for this project, we will need to set up the following components in Azure:

* A **Databricks Workspace** to run our data processing and analytics workloads.
* An **Access Connector** to securely connect our Databricks workspace to our storage account.
* A **Storage Account** to store our raw and processed data.
* A **Container** in the storage account to organize our data.

In Databricks' side, we will set up the following components:
* A **Storage Credential** to access the storage account.
* An **External Location** to reference the container in our storage account.

Step 1️⃣ : Create the Azure Databricks Workspace

Search for "*Azure Databricks*" in the search bar, click on *Create*, and fill in the required details such as subscription, resource group, workspace name, region, and pricing tier.

Here we created the `de_project_workspace` in the East US region with the standard pricing tier.

![1771318723268](image/1_build_the_infrastructure/1771318723268.png)

Click the *Launch Workspace* button to open the Databricks UI.

![1771318828105](image/1_build_the_infrastructure/1771318828105.png)

Step 2️⃣ : Create an Access Connector

Search for "*Access Connector for Azure Databricks*" in the search bar, click on *Create*, and fill in the required details and the Databricks workspace you just created.

Here we created the `de_project_ext_access_connector` in the East US region.

![1771318910600](image/1_build_the_infrastructure/1771318910600.png)

Take note of the **Resource ID** of the Access Connector as we will need it in Step 5.

![1771318922921](image/1_build_the_infrastructure/1771318922921.png)

Step 3️⃣ : Create the Storage Account

Search for "*Storage Account*" in the search bar, click on *Create*, and fill in the required details.

Here we created the `deprojectextdatalake` storage account in the East US region with the standard performance.

![1771318935623](image/1_build_the_infrastructure/1771318935623.png)

Step 4️⃣ : Enable Access to the Storage Account

Let's enable access to the storage account by adding the Access Connector we created in Step 2 to the Storage Account's IAM settings.

In the *Access Control (IAM)* settings of the Storage Account `deprojectextdatalake`, add a new role assignment with the following details:
* **Role**: Storage Blob Data Contributor
* **Principal**: Select the Access Connector `de_project_ext_access_connector`.

![1771319010038](image/1_build_the_infrastructure/1771319010038.png)

Search for the *Storage Blob Data Contributor* role and select it.

![1771319026583](image/1_build_the_infrastructure/1771319026583.png)

Here we can see the role *Storage Blob Data Contributor* has been assigned to `de_project_ext_access_connector`.

![1771319046052](image/1_build_the_infrastructure/1771319046052.png)

Step 5️⃣ : Create A Storage Credential in Databricks

Let's now head to the Databricks UI to create a Storage Credential that will allow us to access the Storage Account from Databricks.

In the Catalog Explorer, navigate to *Data* > *Create* > *Storage Credential* and fill in the required details.

Copy the **Resource ID** of the Access Connector you created in Step 2 and paste it in the *Access connection ID* field.

![1771319063481](image/1_build_the_infrastructure/1771319063481.png)

Here we created the `de_project_ext_storage_credential` storage credential.

![1771319105431](image/1_build_the_infrastructure/1771319105431.png)

Step 6️⃣ : Create A Container

Lets go back to the Azure Portal and create a container in the Storage Account to contain our raw data. 

In the *Containers* section of the `deprojectextdatalake` Storage Account, click on *Add container* and provide the container name.

Here we created a container named `deprojectcontainer`.

![1771319121053](image/1_build_the_infrastructure/1771319121053.png)

Step 7️⃣ : Create An External Location

Let's now create an External Location in Databricks that references the container we just created in Step 6. This will allow us to access the data in the container from Databricks. 

We can create an External Location programmatically using the DDL statement below. We'll reference the `deprojectcontainer` Container in the `deprojectextdatalake` Storage Account using the `abfss` protocol, which is used for accessing Azure Data Lake Storage Gen2. Specify the `de_project_ext_storage_credential` Storage Credential we created in Step 5 to authenticate our access to the Storage Account.

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS de_project_ext_location
URL 'abfss://deprojectcontainer@deprojectextdatalake.dfs.core.windows.net/'
WITH (STORAGE_CREDENTIAL de_project_ext_storage_credential)
COMMENT 'External location for the de_project';
```

Here we created the `de_project_ext_location` External Location.

![1771319139540](image/1_build_the_infrastructure/1771319139540.png)

Step 8️⃣ : Upload the Datasets

Finally, let's upload the raw CSV files to the container in our Storage Account.

![1771319153158](image/1_build_the_infrastructure/1771319153158.png)
