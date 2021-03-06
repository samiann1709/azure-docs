---
title: Move data from OData sources | Microsoft Docs
description: Learn about how to move data from OData sources using Azure Data Factory.
services: data-factory
documentationcenter: ''
author: linda33wj
manager: jhubbard
editor: monicar

ms.assetid: de28fa56-3204-4546-a4df-21a21de43ed7
ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 12/07/2016
ms.author: jingwang

---
# Move data From a OData source using Azure Data Factory
This article outlines how you can use the Copy Activity in an Azure data factory to move data from an OData source to another data store. This article builds on the [data movement activities](data-factory-data-movement-activities.md) article, which presents a general overview of data movement with copy activity and supported data store combinations.

## Supported versions and authentication types
This OData connector support OData version 3.0 and 4.0, and you can copy data from both cloud OData and on-premises OData sources. For the latter, you need to install the Data Management Gateway. See [Move data between on-premises and cloud](data-factory-move-data-between-onprem-and-cloud.md) article for details about Data Management Gateway.

Below authentication types are supported:

* To access **cloud** OData feed, you can use anonymous, basic (user name and password), or Azure Active Directory based OAuth authentication.
* To access **on-premises** OData feed, you can use anonymous, basic (user name and password), or Windows authentication.

## Copy data wizard
The easiest way to create a pipeline that copies data from an OData source is to use the Copy data wizard. See [Tutorial: Create a pipeline using Copy Wizard](data-factory-copy-data-wizard-tutorial.md) for a quick walkthrough on creating a pipeline using the Copy data wizard.

The following examples provide sample JSON definitions that you can use to create a pipeline by using [Azure portal](data-factory-copy-activity-tutorial-using-azure-portal.md) or [Visual Studio](data-factory-copy-activity-tutorial-using-visual-studio.md) or [Azure PowerShell](data-factory-copy-activity-tutorial-using-powershell.md). They show how to copy data from an OData source to an Azure Blob Storage. However, data can be copied to any of the sinks stated [here](data-factory-data-movement-activities.md#supported-data-stores-and-formats) using the Copy Activity in Azure Data Factory.

## Sample: Copy data from OData source to Azure Blob
This sample shows how to copy data from an OData source to Azure Blob Storage. However, data can be copied **directly** to any of the sinks stated [here](data-factory-data-movement-activities.md#supported-data-stores-and-formats) using the Copy Activity in Azure Data Factory.  

The sample has the following data factory entities:

1. A linked service of type [OData](#odata-linked-service-properties).
2. A linked service of type [AzureStorage](data-factory-azure-blob-connector.md#azure-storage-linked-service).
3. An input [dataset](data-factory-create-datasets.md) of type [ODataResource](#odata-dataset-type-properties).
4. An output [dataset](data-factory-create-datasets.md) of type [AzureBlob](data-factory-azure-blob-connector.md#azure-blob-dataset-type-properties).
5. A [pipeline](data-factory-create-pipelines.md) with Copy Activity that uses [RelationalSource](#odata-copy-activity-type-properties) and [BlobSink](data-factory-azure-blob-connector.md#azure-blob-copy-activity-type-properties).

The sample copies data from querying against an OData source to an Azure blob every hour. The JSON properties used in these samples are described in sections following the samples.

**OData linked service**
This example uses the Anonymous authentication. See [OData linked service](#odata-linked-service-properties) section for different types of authentication you can use.

    {
        "name": "ODataLinkedService",
           "properties":
        {
            "type": "OData",
               "typeProperties":
            {
                "url": "http://services.odata.org/OData/OData.svc",
               "authenticationType": "Anonymous"
               }
           }
    }


**Azure Storage linked service**

    {
          "name": "AzureStorageLinkedService",
        "properties": {
            "type": "AzureStorage",
            "typeProperties": {
              "connectionString": "DefaultEndpointsProtocol=https;AccountName=<accountname>;AccountKey=<accountkey>"
            }
          }
    }

**OData input dataset**

Setting “external”: ”true” informs the Data Factory service that the dataset is external to the data factory and is not produced by an activity in the data factory.

    {
        "name": "ODataDataset",
        "properties":
        {
            "type": "ODataResource",
            "typeProperties":
            {
                 "path": "Products"
            },
            "linkedServiceName": "ODataLinkedService",
            "structure": [],
            "availability": {
                "frequency": "Hour",
                "interval": 1
            },
            "external": true,
            "policy": {
                "retryInterval": "00:01:00",
                "retryTimeout": "00:10:00",
                "maximumRetry": 3                
            }
        }
    }

Specifying **path** in the dataset definition is optional.

**Azure Blob output dataset**

Data is written to a new blob every hour (frequency: hour, interval: 1). The folder path for the blob is dynamically evaluated based on the start time of the slice that is being processed. The folder path uses year, month, day, and hours parts of the start time.

    {
        "name": "AzureBlobODataDataSet",
        "properties": {
            "type": "AzureBlob",
            "linkedServiceName": "AzureStorageLinkedService",
            "typeProperties": {
                "folderPath": "mycontainer/odata/yearno={Year}/monthno={Month}/dayno={Day}/hourno={Hour}",
                "format": {
                    "type": "TextFormat",
                    "rowDelimiter": "\n",
                    "columnDelimiter": "\t"
                },
                "partitionedBy": [
                    {
                        "name": "Year",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "yyyy"
                        }
                    },
                    {
                        "name": "Month",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "MM"
                        }
                    },
                    {
                        "name": "Day",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "dd"
                        }
                    },
                    {
                        "name": "Hour",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "HH"
                        }
                    }
                ]
            },
            "availability": {
                "frequency": "Hour",
                "interval": 1
            }
        }
    }



**Pipeline with Copy activity**

The pipeline contains a Copy Activity that is configured to use the input and output datasets and is scheduled to run every hour. In the pipeline JSON definition, the **source** type is set to **RelationalSource** and **sink** type is set to **BlobSink**. The SQL query specified for the **query** property selects the latest (newest) data from the OData source.

    {
        "name": "CopyODataToBlob",
        "properties": {
            "description": "pipeline for copy activity",
            "activities": [
                {
                    "type": "Copy",
                    "typeProperties": {
                        "source": {
                            "type": "RelationalSource",
                            "query": "?$select=Name, Description&$top=5",
                        },
                        "sink": {
                            "type": "BlobSink",
                            "writeBatchSize": 0,
                            "writeBatchTimeout": "00:00:00"
                        }
                    },
                    "inputs": [
                        {
                            "name": "ODataDataSet"
                        }
                    ],
                    "outputs": [
                        {
                            "name": "AzureBlobODataDataSet"
                        }
                    ],
                    "policy": {
                        "timeout": "01:00:00",
                        "concurrency": 1
                    },
                    "scheduler": {
                        "frequency": "Hour",
                        "interval": 1
                    },
                    "name": "ODataToBlob"
                }
            ],
            "start": "2016-02-01T18:00:00Z",
            "end": "2016-02-03T19:00:00Z"
        }
    }


Specifying **query** in the pipeline definition is optional. The **URL** that the Data Factory service uses to retrieve data is: URL specified in the linked service (required) + path specified in the dataset (optional) + query in the pipeline (optional).

## OData linked Service properties
The following table provides description for JSON elements specific to OData linked service.

| Property | Description | Required |
| --- | --- | --- |
| type |The type property must be set to: **OData** |Yes |
| url |Url of the OData service. |Yes |
| authenticationType |Type of authentication used to connect to the OData source. <br/><br/> For cloud OData, possible values are Anonymous, Basic, and OAuth (note Azure Data Factory currently only support Azure Active Directory based OAuth). <br/><br/> For on-premises OData, possible values are Anonymous, Basic, and Windows. |Yes |
| username |Specify user name if you are using Basic authentication. |Yes (only if you are using Basic authentication) |
| password |Specify password for the user account you specified for the username. |Yes (only if you are using Basic authentication) |
| authorizedCredential |If you are using OAuth, click **Authorize** button in the Data Factory Copy Wizard or Editor and enter your credential, then the value of this property will be auto-generated. |Yes (only if you are using OAuth authentication) |
| gatewayName |Name of the gateway that the Data Factory service should use to connect to the on-premises OData service. Specify only if you are copying data from on-prem OData source. |No |

### Using Basic authentication
    {
        "name": "inputLinkedService",
        "properties":
        {
            "type": "OData",
               "typeProperties":
            {
               "url": "http://services.odata.org/OData/OData.svc",
               "authenticationType": "Basic",
               "username": "username",
               "password": "password"
           }
       }
    }

### Using Anonymous authentication
    {
        "name": "ODataLinkedService",
           "properties":
        {
            "type": "OData",
            "typeProperties":
            {
               "url": "http://services.odata.org/OData/OData.svc",
               "authenticationType": "Anonymous"
           }
       }
    }

### Using Windows authentication accessing on-premises OData source
    {
        "name": "inputLinkedService",
        "properties":
        {
            "type": "OData",
               "typeProperties":
            {
               "url": "<endpoint of on-premises OData source e.g. Dynamics CRM>",
               "authenticationType": "Windows",
               "username": "domain\\user",
               "password": "password",
               "gatewayName": "mygateway"
           }
       }
    }

### Using OAuth authentication accessing cloud OData source
    {
        "name": "inputLinkedService",
        "properties":
        {
            "type": "OData",
               "typeProperties":
            {
               "url": "<endpoint of cloud OData source e.g. https://<tenant>.crm.dynamics.com/XRMServices/2011/OrganizationData.svc">",
               "authenticationType": "OAuth",
               "authorizedCredential": "<auto generated by clicking the Authorize button on UI>"
           }
       }
    }

## OData dataset type properties
For a full list of sections & properties available for defining datasets, see the [Creating datasets](data-factory-create-datasets.md) article. Sections such as structure, availability, and policy of a dataset JSON are similar for all dataset types (Azure SQL, Azure blob, Azure table, etc.).

The **typeProperties** section is different for each type of dataset and provides information about the location of the data in the data store. The typeProperties section for dataset of type **ODataResource** (which includes OData dataset) has the following properties

| Property | Description | Required |
| --- | --- | --- |
| path |Path to the OData resource |No |

## OData Copy Activity type properties
For a full list of sections & properties available for defining activities, see the [Creating Pipelines](data-factory-create-pipelines.md) article. Properties such as name, description, input and output tables, and policy are available for all types of activities.

Properties available in the typeProperties section of the activity on the other hand vary with each activity type. For Copy activity, they vary depending on the types of sources and sinks.

When source is of type **RelationalSource** (which includes OData) the following properties are available in typeProperties section:

| Property | Description | Example | Required |
| --- | --- | --- | --- |
| query |Use the custom query to read data. |"?$select=Name, Description&$top=5" |No |

[!INCLUDE [data-factory-structure-for-rectangualr-datasets](../../includes/data-factory-structure-for-rectangualr-datasets.md)]

### Type mapping for OData
As mentioned in the [data movement activities](data-factory-data-movement-activities.md) article, Copy activity performs automatic type conversions from source types to sink types with the following 2-step approach:

1. Convert from native source types to .NET type
2. Convert from .NET type to native sink type

When moving data from OData data stores, OData data types are mapped to .NET types.

[!INCLUDE [data-factory-column-mapping](../../includes/data-factory-column-mapping.md)]

[!INCLUDE [data-factory-type-repeatability-for-relational-sources](../../includes/data-factory-type-repeatability-for-relational-sources.md)]

## Performance and Tuning
See [Copy Activity Performance & Tuning Guide](data-factory-copy-activity-performance.md) to learn about key factors that impact performance of data movement (Copy Activity) in Azure Data Factory and various ways to optimize it.
