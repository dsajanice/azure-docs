---
title: Indexing Azure Table Storage with Azure Search
description: Learn how to index data stored in Azure Tables with Azure Search
services: search
documentationcenter: ''
author: chaosrealm
manager: pablocas
editor: ''

ms.assetid: 1cc27411-d0cc-40ed-8aed-c7cb9ab402b9
ms.service: search
ms.devlang: rest-api
ms.workload: search
ms.topic: article
ms.tgt_pltfrm: na
ms.date: 01/18/2017
ms.author: eugenesh
---

# Indexing Azure Table Storage with Azure Search
This article shows how to use Azure Search to index data stored in Azure Table Storage. The new Azure Search table indexer makes this process quick and seamless.

## Setting up Azure table indexing
To set up and configure an Azure table indexer, you can use the Azure Search REST API to create and manage **indexers** and **data sources** as described in [Indexer operations](https://msdn.microsoft.com/library/azure/dn946891.aspx). You can also use [version 2.0-preview](https://msdn.microsoft.com/library/mt761536%28v=azure.103%29.aspx) of the .NET SDK. In the future, support for table indexing will be added to the Azure Portal.

A data source specifies which data to index, credentials needed to access the data, and policies that enable Azure Search to efficiently identify changes in the data (new, modified or deleted rows).

An indexer reads data from a data source and loads it into a target search index.

To set up table indexing:

1. Create a data source
   * Set the `type` parameter to `azuretable`
   * Pass in your storage account connection string as the `credentials.connectionString` parameter. See [How to specify credentials](#Credentials) below for details.
   * Specify the table name using the `container.name` parameter
   * Optionally, specify a query using the `container.query` parameter. Whenever possible, use a filter on PartitionKey for best performance; any other query will result in a full table scan, which can result in poor performance for large tables.
2. Create a search index with the schema that corresponds to the columns in the table that you want to index.
3. Create the indexer by connecting your data source to the search index.

### Create data source
    POST https://[service name].search.windows.net/datasources?api-version=2016-09-01
    Content-Type: application/json
    api-key: [admin key]

    {
        "name" : "table-datasource",
        "type" : "azuretable",
        "credentials" : { "connectionString" : "DefaultEndpointsProtocol=https;AccountName=<account name>;AccountKey=<account key>;" },
        "container" : { "name" : "my-table", "query" : "PartitionKey eq '123'" }
    }   

For more on the Create Datasource API, see [Create Datasource](https://msdn.microsoft.com/library/azure/dn946876.aspx).

<a name="Credentials"></a>
#### How to specify credentials ####

You can provide the credentials for the table in one of these ways: 

- **Full access storage account connection string**: `DefaultEndpointsProtocol=https;AccountName=<your storage account>;AccountKey=<your account key>`. You can get the connection string from the Azure portal by navigating to the storage account blade > Settings > Keys (for Classic storage accounts) or Settings > Access keys (for Azure Resource Manager storage accounts).
- **Storage account shared access signature** (SAS) connection string: `TableEndpoint=https://<your account>.table.core.windows.net/;SharedAccessSignature=?sv=2016-05-31&sig=<the signature>&spr=https&se=<the validity end time>&srt=co&ss=t&sp=rl`. The SAS should have the list and read permissions on containers (tables in this case) and objects (table rows).
-  **Table shared access signature**: `ContainerSharedAccessUri=https://<your storage account>.table.core.windows.net/<table name>?tn=<table name>&sv=2016-05-31&sig=<the signature>&se=<the validity end time>&sp=r`. The SAS should have query (read) permissions on the table.

For more info on storage shared access signatures, see [Using Shared Access Signatures](../storage/storage-dotnet-shared-access-signature-part-1.md).

> [!NOTE]
> If you use SAS credentials, you will need to update the data source credentials periodically with renewed signatures to prevent their expiration. If SAS credentials expire, the indexer will fail with an error message similar to `Credentials provided in the connection string are invalid or have expired.`.  

### Create index
    POST https://[service name].search.windows.net/indexes?api-version=2016-09-01
    Content-Type: application/json
    api-key: [admin key]

    {
          "name" : "my-target-index",
          "fields": [
            { "name": "key", "type": "Edm.String", "key": true, "searchable": false },
            { "name": "SomeColumnInMyTable", "type": "Edm.String", "searchable": true }
          ]
    }

For more on the Create Index API, see [Create Index](https://msdn.microsoft.com/library/dn798941.aspx)

### Create indexer
Finally, create the indexer that references the data source and the target index. For example:

    POST https://[service name].search.windows.net/indexers?api-version=2016-09-01
    Content-Type: application/json
    api-key: [admin key]

    {
      "name" : "table-indexer",
      "dataSourceName" : "table-datasource",
      "targetIndexName" : "my-target-index",
      "schedule" : { "interval" : "PT2H" }
    }

For more details on the Create Indexer API, check out [Create Indexer](https://msdn.microsoft.com/library/azure/dn946899.aspx).

That's all there is to it - happy indexing!

## Dealing with different field names
Often, the field names in your existing index will be different from the property names in your table. You can use **field mappings** to map the property names from the table to the field names in your search index. To learn more about field mappings, see [Azure Search indexer field mappings bridge the differences between data sources and search indexes](search-indexer-field-mappings.md).

## Handling document keys
In Azure Search, the document key uniquely identifies a document. Every search index must have exactly one key field of type `Edm.String`. The key field is required for each document that is being added to the index (in fact, it is the only required field).

Since table rows have a compound key, Azure Search generates a synthetic field called `Key` that is a concatenation of partition key and row key values. For example, if a row’s PartitionKey is `PK1` and RowKey is `RK1`, then `Key` field's value will be `PK1RK1`.

> [!NOTE]
> The `Key` value may contain characters that are invalid in document keys, such as dashes. You can deal with invalid characters by using the `base64Encode` [field mapping function](search-indexer-field-mappings.md#base64EncodeFunction). If you do this, remember to also use URL-safe Base64 encoding when passing document keys in API calls such as Lookup.
>
>

## Incremental indexing and deletion detection
When you set up a table indexer to run on a schedule, it reindexes only new or updated rows, as determined by a row’s `Timestamp` value. You don’t have to specify a change detection policy – incremental indexing is enabled for you automatically.

To indicate that certain documents must be removed from the index, you can use a soft delete strategy – instead of deleting a row, add a property to indicate that it is deleted, and set up a soft deletion detection policy on the datasource. For example, the policy shown below will consider that a row is deleted if it has a property `IsDeleted` with the value `"true"`:

    PUT https://[service name].search.windows.net/datasources?api-version=2016-09-01
    Content-Type: application/json
    api-key: [admin key]

    {
        "name" : "my-table-datasource",
        "type" : "azuretable",
        "credentials" : { "connectionString" : "<your storage connection string>" },
        "container" : { "name" : "table name", "query" : "query" },
        "dataDeletionDetectionPolicy" : { "@odata.type" : "#Microsoft.Azure.Search.SoftDeleteColumnDeletionDetectionPolicy", "softDeleteColumnName" : "IsDeleted", "softDeleteMarkerValue" : "true" }
    }   


## Help us make Azure Search better
If you have feature requests or ideas for improvements, please reach out to us on our [UserVoice site](https://feedback.azure.com/forums/263029-azure-search/).
