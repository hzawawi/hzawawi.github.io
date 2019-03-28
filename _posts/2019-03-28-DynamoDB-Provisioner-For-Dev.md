---
layout: page
title: Make DynamoDB easy for dev
permalink: /dynamodb-post-2/
---

### DynamoDb in Development:

Working with the DynamoDb API involves a lot of boilerplate code and code duplication for defining tables. 
In early phases of the development cycle, you would expect a lot of destructive changes to the tables design. So relying on the API only, can start to be painful.
This could slow down the development cycle, and quickly grow technical debt around code maintainability.

Introducing _DynamoDbProvisioner_, a tool intended to make all these complexities hidden at development time. This will help developers focus more on the tables design and not worry about any irrelevant details that could suck a lot of development time.

_DynamoDbProvisioner_ will be responsible for initializing  all the defined tables on the start of the application, with the ability to delete/recreate tables when there is a need to.

### How Does DynamoDbProvisioner work?

The provisioner is registered against a set of assemblies, on runtime the provisioner will go through all types in the assemblies and look for any class decorated by _DynamoDBTableAttribute_. Knowing that, it will be enough to register the provisioner against the main assembly and you don’t have to worry about adding any configuration for any new table added.

The provisioner relies on having a class representing the DynamoDb table decorated by some attributes that will feed the dynamodb APIthe information needed to create this table.

Some of these attributes are the following :
- _DynamoDBTableAttribute_ this will map to the table name
- _DynamoDBHashKeyAttribute_ this will specify which of the following model properties will represent the hashkey of the table
- _DynamoDBRangeKeyAttribute_ this will specify which of the following model properties will represent the rangeKey of the table
- _DynamoDBGlobalSecondaryIndexHashKeyAttribute_ this will specify which of the following model properties will represent the hashKey of the global secondary index
- _DynamoDBGlobalSecondaryIndexRangeKeyAttribute_ this will specify which of the following model properties will represent the rangeKey of the global secondary index
- _DynamoDBLocalSecondaryIndexRangeKeyAttribute_
this will specify which of the following model properties will represent the rangeKey of the local secondary index
- _DynamoDBTimeToLiveAttribute_ this will specify which of the following model properties will store  
to store the expiration time for items

### Example of a DynamoDb table
 ```
[DynamoDBTable(nameof(TableWithStringHashKeyAndNumberRangeKey))]
public class TableWithStringHashKeyAndNumberRangeKey
{
  [DynamoDBHashKey]
  public string Id { get; set; }

  [DynamoDBRangeKey]
  public int Timestamp { get; set; }
}
}
```

```
[DynamoDBTable(nameof(TableWithGlobalSecondaryIndexHashKeyAndRangeKey))]
public class TableWithGlobalSecondaryIndexHashKeyAndRangeKey
{
  [DynamoIndex(ProjectionType = ProjectionType.All)] public const string ByLastNameAndFirstNameIndex = "ByLastNameAndFirstName-Index";

  [DynamoDBHashKey]
  public string Id { get; set; }

  [DynamoDBGlobalSecondaryIndexHashKey(ByLastNameAndFirstNameIndex)]
  public string LastName { get; set; }

  [DynamoDBGlobalSecondaryIndexRangeKey(ByLastNameAndFirstNameIndex)]
  public string FirstName { get; set; }
}
```

### DynamoDbProvisioner in integration tests

A good use of DynamoDBProvisioner is in tests, where you need to exercise some dynamoDb operations and you don't want to worry much about creating/cleaning tables while running the tests.
The provisioner gives you the ability to pass in a table prefix during initialization. In tests, you will need to have some random prefix, like a GUID. This will enable you to run many tests in parallel against multiple copies of the same table. 

### DynamoDb locally and not in cloud

In addition to the provisioner, I recommend the use of one of the applications listed below. These will give you a full environment to interact with dynamodb, without the need to provision any table in the cloud. This will cut some costs, make the development/test much faster and easier, and not depend on any external network.

- [Localstack](https://github.com/localstack/localstack)
- [DynamoDBLocal](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html)
  
### Nuget Package
You can download and reference the nuget package from [Pushpay.DynamoDbProvisioner](https://www.nuget.org/packages?q=Pushpay.DynamoDbProvisioner)

### Source code 
You can find the above code snippets on [github](https://github.com/hzawawi/DynamoProvisioner)

