## 🛠️ Build the Azure Databricks Infrastructure

To build the infrastructure for this project, we will need to set up the following components in Azure:

* A **Databricks Workspace** to run our data processing and analytics workloads.
* An **Access Connector** to securely connect our Databricks workspace to our storage account.
* A **Storage Account** to store our raw and processed data.
* A **Container** in the storage account to organize our data.

In Databricks' side, we will set up the following components:
* A **Storage Credential** to access the storage account.
* An **External Location** to reference the container in our storage account.

<details>
<summary>Step 1️⃣ : Create the Azure Databricks Workspace</summary>

Search for "*Azure Databricks*" in the search bar, click on *Create*, and fill in the required details such as subscription, resource group, workspace name, region, and pricing tier.

Here we created the `de_project_workspace` in the East US region with the standard pricing tier.

![1770637481388](image/README/1770637481388.png)

Click the *Launch Workspace* button to open the Databricks UI.

![1771129013988](image/README/1771129013988.png)

</details>

<details>
<summary>Step 2️⃣ : Create an Access Connector</summary>

Search for "*Access Connector for Azure Databricks*" in the search bar, click on *Create*, and fill in the required details and the Databricks workspace you just created.

Here we created the `de_project_ext_access_connector` in the East US region.

![1770637584703](image/README/1770637584703.png)

Take note of the **Resource ID** of the Access Connector as we will need it in Step 5.

![1770637630239](image/README/1770637630239.png)

</details>

<details>
<summary>Step 3️⃣ : Create the Storage Account</summary>

Search for "*Storage Account*" in the search bar, click on *Create*, and fill in the required details.

Here we created the `deprojectextdatalake` storage account in the East US region with the standard performance.

![1770637685309](image/README/1770637685309.png)

</details>

<details>
<summary>Step 4️⃣ : Enable Access to the Storage Account</summary>

Let's enable access to the storage account by adding the Access Connector we created in Step 2 to the Storage Account's IAM settings.

In the *Access Control (IAM)* settings of the Storage Account `deprojectextdatalake`, add a new role assignment with the following details:
* **Role**: Storage Blob Data Contributor
* **Principal**: Select the Access Connector `de_project_ext_access_connector`.

![1771129212869](image/README/1771129212869.png)

Search for the *Storage Blob Data Contributor* role and select it.

![1771129232625](image/README/1771129232625.png)

Here we can see the role *Storage Blob Data Contributor* has been assigned to `de_project_ext_access_connector`.

![1770637797073](image/README/1770637797073.png)

</details>

<details>
<summary>Step 5️⃣ : Create A Storage Credential in Databricks</summary>

Let's now head to the Databricks UI to create a Storage Credential that will allow us to access the Storage Account from Databricks.

In the Catalog Explorer, navigate to *Data* > *Create* > *Storage Credential* and fill in the required details.

Copy the **Resource ID** of the Access Connector you created in Step 2 and paste it in the *Access connection ID* field.

![1771129292579](image/README/1771129292579.png)

Here we created the `de_project_ext_storage_credential` storage credential.

![1770637904308](image/README/1770637904308.png)

</details>

<details>
<summary>Step 6️⃣ : Create A Container</summary>

Lets go back to the Azure Portal and create a container in the Storage Account to contain our raw data. 

In the *Containers* section of the `deprojectextdatalake` Storage Account, click on *Add container* and provide the container name.

Here we created a container named `deprojectcontainer`.

![1770638361588](image/README/1770638361588.png)

</details>

<details>
<summary>Step 7️⃣ : Create An External Location</summary>

Let's now create an External Location in Databricks that references the container we just created in Step 6. This will allow us to access the data in the container from Databricks. 

We can create an External Location programmatically using the DDL statement below. We'll reference the `deprojectcontainer` Container in the `deprojectextdatalake` Storage Account using the `abfss` protocol, which is used for accessing Azure Data Lake Storage Gen2. Specify the `de_project_ext_storage_credential` Storage Credential we created in Step 5 to authenticate our access to the Storage Account.

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS de_project_ext_location
URL 'abfss://deprojectcontainer@deprojectextdatalake.dfs.core.windows.net/'
WITH (STORAGE_CREDENTIAL de_project_ext_storage_credential)
COMMENT 'External location for the de_project';
```

Here we created the `de_project_ext_location` External Location.

![1770637951731](image/README/1770637951731.png)

</details>

<details>
<summary>Step 8️⃣ : Upload the Datasets</summary>

Finally, let's upload the raw CSV files to the container in our Storage Account.

![1770638062683](image/README/1770638062683.png)

</details>