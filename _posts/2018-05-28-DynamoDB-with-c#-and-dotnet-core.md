---
layout: page
title: DynamoDB with c# and .Net Core.
permalink: /dynamodb-post-1/
---

### Objective
When started working with dynamodb database, I found the online documentation and online code samples were quite limited and bit confusing. So I decided to start writing few blogs around the things I learned along the way so it provides some helps for future developers.

Goal of this first blog, is to give a simple example to quickly setup dynamodb client and be able to do some basic operations with the dynamoDb.
Future blogs will be giving more deep guidelines about amazon dynamodb api and dynamodb core features.


### Dependencies

- #### AWSSDK.DynamoDBv2 package:
  .Net API to facilitate the interaction with aws dynamodb in order to execute different operations against the database
  such as (createTable, saveItem, retrieveItem,etc..)

- #### [localstack](https://github.com/localstack/localstack): 
  framework that helps mocking different aws cloud applications, in our example we are going to rely on it to mock amazon dynamodb database.
  localstack really helpful to use when you want to develop a cloud application offline and reduce dependencies on the cloud infrastructure. 
  
### Code Sample
 1. #### DynamoDb client setup
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
In order to create the client we need to pass to `AmazonDynamoDBClient` constructor the accessId and accessKey
and sessionToken in case you have MFA enabled for your account. For the sake of simplicity we are pointing to localstack mock dynamo mock service.


2. #### Create table

```
[DynamoDBTable("person")]
public class Person: IEquatable<Person>
{
    [DynamoDBHashKey]
    public int Id { get; set; }

    public string FirstName { get; set; }
    
    public string LastName { get; set; }

    public bool Equals(Person other)
    {
        if (ReferenceEquals(null, other)) return false;
        if (ReferenceEquals(this, other)) return true;
        return Id == other.Id && string.Equals(FirstName, other.FirstName) && string.Equals(LastName, other.LastName);
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

3. #### tests

```
public class DynamoClientTests
{
    private readonly DynamoClient _dynamoDbClient;

    public DynamoClientTests()
    {
        _dynamoDbClient = new DynamoClient();
        try
        {
            _dynamoDbClient.SetupAsync().Wait();
        }
        catch (AggregateException e)
        {
            //ignore table already created
            Console.WriteLine(e);
        }
    }
    
    [Fact]
    public void SaveAPersonAndRetrieveItBack()
    {
        var person = new Person
        {
            Id = 1,
            FirstName = "sam",
            LastName = "griffen"
        };

        _dynamoDbClient.SavePerson(person).Wait();

        var returnedPerson = _dynamoDbClient.GetPerson(person.Id).Result;

        Assert.Equal(returnedPerson, person);
    }
}
```
