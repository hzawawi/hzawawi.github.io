---
layout: page
title: DynamoDB with C# and .NET Core.
permalink: /dynamodb-post-1/
---

### Objective
When I first started working with DynamoDB database, I found the online documentation and code samples were quite limited and a bit confusing. This is why I decided to start writing few blogs around the things I learned along the way, to share knowledge and assist interested developers.

The goal of this first blog, is to give a simple example of how to quickly setup DynamoDB client, and how to do some basic operations with the DynamoDB.
Future blogs will give deeper guidelines about Amazon DynamoDB API and its core features.

### Dependencies

- #### AWSSDK.DynamoDBv2 package:
  this is a .NET API that facilitates the interaction with AWS DynamoDB in order to execute different operations against the database such as (createTable, saveItem, retrieveItem,etc..)

- #### [Localstack](https://github.com/localstack/localstack): 
  Localstack is a framework that helps mock different AWS cloud applications; In our example below, we are going to rely on it to mock the Amazon DynamoDB database.
  Localstack is really helpful to use when you want to develop a cloud application offline and reduce dependencies on the cloud infrastructure. 
  
### Code Sample
#### 1. DynamoDb client setup
 ```
 public class DynamoClient
 {
 	private readonly AmazonDynamoDBClient _amazonDynamoDBClient;
 	private readonly DynamoDBContext _context;

 	public DynamoClient()
 	{
 	    //localstack ignores secrets
 	    _amazonDynamoDBClient = new AmazonDynamoDBClient("awsAccessKeyId", "awsSecretAccessKey",
 	        new AmazonDynamoDBConfig
 	        {
 	            ServiceURL = "http://localhost:4569", //default localstack url
 	            UseHttp = true,
 	        });

 	    _context = new DynamoDBContext(_amazonDynamoDBClient, new DynamoDBContextConfig
 	    {
 	        TableNamePrefix = "test_"
 	    });
 	}
}
```

In order to create the client, we need to pass the accessId and accessKey to the **AmazonDynamoDBClient** constructor.
If you have MFA enabled on your AWS account, you will need to pass in the seessionToken.
For the sake of simplicity, we are pointing to localstack's mock DynamoDB service.

#### 2. Setup table and equivalent model class

```
[DynamoDBTable("student")]
public class Student : IEquatable<Student>
{
     [DynamoDBHashKey] 
     public int Id { get; set; }
     
     public string FirstName { get; set; }

     public string LastName { get; set; }

     public bool Equals(Student other)
     {
         if (ReferenceEquals(null, other)) return false;
         if (ReferenceEquals(this, other)) return true;
         return Id == other.Id && string.Equals(FirstName, other.FirstName) && string.Equals(LastName, other.LastName);
     }

     public override bool Equals(object obj)
     {
         if (ReferenceEquals(null, obj)) return false;
         if (ReferenceEquals(this, obj)) return true;
         if (obj.GetType() != this.GetType()) return false;
         return Equals((Student) obj);
     }

     public override int GetHashCode()
     {
         unchecked
         {
             var hashCode = Id;
             hashCode = (hashCode * 397) ^ (FirstName != null ? FirstName.GetHashCode() : 0);
             hashCode = (hashCode * 397) ^ (LastName != null ? LastName.GetHashCode() : 0);
             return hashCode;
         }
     }
}
```
In this example model, we are using two attributes:
- ***DynamoDBTable*** to map the DynamoDB equivalent table
- ***DynamoDBHashKey*** to map the table hashkey

```
public async Task<CreateTableResponse> SetupAsync()
{
    var createTableRequest = new CreateTableRequest
    {
        TableName = "test_student",
        AttributeDefinitions = new List<AttributeDefinition>(),
        KeySchema = new List<KeySchemaElement>(),
        GlobalSecondaryIndexes = new List<GlobalSecondaryIndex>(),
        LocalSecondaryIndexes = new List<LocalSecondaryIndex>(),
        ProvisionedThroughput = new ProvisionedThroughput
        {
            ReadCapacityUnits = 1,
            WriteCapacityUnits = 1
        }
    };
    createTableRequest.KeySchema = new[]
    {
        new KeySchemaElement
        {
            AttributeName = "Id",
            KeyType = KeyType.HASH,
        },

    }.ToList();

    createTableRequest.AttributeDefinitions = new[]
    {
        new AttributeDefinition
        {
            AttributeName = "Id",
            AttributeType = ScalarAttributeType.N,
        }
    }.ToList();

    return await _amazonDynamoDBClient.CreateTableAsync(createTableRequest);
}
```

In this snippet, we are building the create table request to construct the table. To do this; we have to specify all defined keys on the table. This seems like a bit of ceremony to do if you have quite few tables, in future I will demonstrate some solutions to do this in an easier and faster maner.
For the sake of simplicity we are just going to stick with the raw API calls in this blog.

#### 3. Save/Query from the table

```
public async Task SaveOrUpdateStudent(Student student)
{
    await _context.SaveAsync(student);
}

public async Task<Student> GetStudentUsingHashKey(int id)
{
    return await _context.LoadAsync<Student>(id);
}

public async Task<Student> ScanForStudentUsingFirstName(string firstName)
{
    var search = _context.ScanAsync<Student>
    (
        new[]
        {
            new ScanCondition
            (
                nameof(Student.FirstName),
                ScanOperator.Equal,
                firstName
            )
        }
    );
    var result = await search.GetRemainingAsync();
    return result.FirstOrDefault();
}
```
- SaveOrUpdateStudent: saving a new entity is quite straight forward, all you need to do is to call `SaveAsync`, Bear in mind, `SaveAsync` will create/update the record; if the record already exists, the method invocation will just override the data on the matched record.
- GetStudentUsingHashKey: will retrieve the record back using our hashKey
- ScanForStudentUsingFirstName: will scan the whole table looking for a matching firstName

#### 4. Prevent overwriting existing records
```
public async Task SaveOnlyStudent(Student student)
{
    var identityEventTable = Table.LoadTable(_amazonDynamoDBClient, "test_student");

    var expression = new Expression
    {
        ExpressionAttributeNames = new Dictionary<string, string>
        {
            {"#key", nameof(student.Id)},
        },
        ExpressionAttributeValues =
        {
            {":key", student.Id},
        },
        ExpressionStatement = "attribute_not_exists(#key) OR #key <> :key",
    };

    var document = _context.ToDocument(student);

    await identityEventTable.PutItemAsync(document, new PutItemOperationConfig
    {
        ConditionalExpression = expression,
        ReturnValues = ReturnValues.None
    });
}
```
In order to prevent overriding existing records, we need to leverage conditional expressions in DynamoDB.
So if we try to insert a record with a hashKey already present in the table, we will get a *ConditionalCheckFailedException* being thrown.

### Reference
You can find the above code snippets on [github](https://github.com/hzawawi/DynamoDbSamples)
 
