# Medallion Architecture on Databricks: A Cyclistic Case Study

## Introduction

Welcome to the **Medallion Architecture on Databricks: A Cyclistic Case Study**! In this project, we will explore how to implement the Medallion Architecture on Azure Databricks, with a focus on the Cyclistic bike share dataset.

This project will cover the following key areas:
- Understand the business requirements
- Explore the Cyclistic dataset
- Build the Azure Databricks infrastructure
- Build the Catalog & Schemas
- Build the Landing, Bronze, Silver, and Gold layers
- Perform data analytics and derive insights


## 📝 Understand the Business Requirements

The director of marketing team believes Cyclistic's future success depends on maximizing the number of annual memberships. Therefore, your team wants to understand how casual riders and annual members use Cyclistic bikes differently. From these insights, the Marketing team will design a new marketing strategy to convert casual riders into members.

Determine how MEMBERS and CASUAL users use Cyclistic bikes differently. Specifically:
1. Analyze the data and identify trends and patterns,
2. Create visualizations, and
3. Provide key findings and recommendations.

Let's ask the **SMART** questions.

* **Specific**. Which bike type is the most preferred by MEMBERS and CASUAL users?
* **Measurable**. What percentage of the total rides is made by the MEMBERS?
* **Action-oriented**. How can we convince the CASUAL users to join the annual membership?
* **Relevant**. What programs can we launch to attract more users to ride bikes?
* **Time-bound**. What trends can we observe from different short-term time periods?


## 🔍 Explore the Cyclistic Dataset

Cyclistic’s datasets can be downloaded [here](https://divvy-tripdata.s3.amazonaws.com/index.html). This case study covers the 2025 data available in the repository. The files are first-party datasets owned, prepared and shared by Cyclistic with file naming format ‘YYYYMM-divvy-tripdata.csv’.

Let's be reminded that Cyclistic is a fictional company that represents a real-world organization. Its datasets are prepared to maintain anonymity. The data has been made available by Motivate International Inc. under this [license](https://divvybikes.com/data-license-agreement).

<details>
<summary>👉 Here's a quick look of the dataset</summary>


**Sample File** : `202501-divvy-tripdata.csv`

![1771125885457](image/README/1771125885457.png)

**Shape & Structure**

* **Rows × Columns** : 138,689 × 13
* **Columns** : `ride_id`, `rideable_type`, `started_at`, `ended_at`, `start_station_name`, `start_station_id`, `end_station_name`, `end_station_id`, `start_lat`, `start_lng`, `end_lat`, `end_lng`, `member_casual`

**Data Types**

* **STRING** : `ride_id`, `rideable_type`, `start_station_name`, `start_station_id`, `end_station_name`, `end_station_id`, `member_casual`
* **TIMESTAMP** : `started_at`, `ended_at`
* **DOUBLE** : `start_lat`, `start_lng`, `end_lat`, `end_lng`

</details>


## 🛠️ Build the Azure Databricks Infrastructure

To build the infrastructure for this project, we will need to set up the following components in Azure:

* A **Databricks Workspace** to run our data processing and analytics workloads.
* An **Access Connector** to securely connect our Databricks workspace to our storage account.
* A **Storage Account** to store our raw and processed data.
* A **Container** in the storage account to organize our data.

In Databricks' side, we will set up the following components:
* A **Storage Credential** to access the storage account.
* An **External Location** to reference the container in our storage account.

Check out the documentation here 👉 [Build the Azure Databricks Infrastructure](docs/1_build_the_infrastructure.md)


## 🗂️ Build the Catalog & Schemas

We begin the project by creating a Catalog and Schemas in Databricks. The Catalog is a logical container for databases, and the Schemas are logical containers for tables. This structure allows us to organize our data according to the Medallion Architecture, which consists of the Landing, Bronze, Silver, and Gold layers.

Check out the documentation here 👉 [Build the Catalog & Schemas](docs/2_build_the_catalog_schemas.md)

## 🏗️ Build the Landing, Bronze, Silver & Gold Layers

The Medallion Architecture is a data design pattern that organizes data into different layers based on its level of refinement and quality. The layers are typically named Landing, Bronze, Silver, and Gold. Each layer serves a specific purpose in the data processing pipeline, allowing for better organization, governance, and scalability of data.

In this project, we will build each layer of the Medallion Architecture on Azure Databricks, starting with the Landing layer where we will store the raw data files, followed by the Bronze layer for raw ingested data, the Silver layer for cleansed and conformed data, and finally the Gold layer for analytics and reporting.

Check out the documentation here 👉 [Build the Landing, Bronze, Silver & Gold Layers](docs/3_build_the_layers.md)



## 🚀 Wrap Up



## 👨‍💻 About Me

Hello! I'm **Paul Joseph Mendoza**, a junior data engineer with a love for building new stuff and uncovering the stories hidden within data. As a career shifter, I transitioned into data engineering after discovering my passion for working with data and solving complex problems. I have a strong proficiency in SQL, Python, Excel, Power BI, and cloud platforms like Azure and AWS. I'm always eager to learn new technologies and build data engineering projects. 

When I'm not working on data projects, I spend my time reading (I'm into Stephen King books right now), walking around Cebu City with my wife, and napping (Yeah, this is the best). Let's stay in touch! Feel free to connect with me on LinkedIn or check out my GitHub profile for more projects coming soooon.

[![GitHub]](https://github.com/nice1-pjcmendoza)

[![LinkedIn]](https://www.linkedin.com/in/paul-joseph-mendoza/)
