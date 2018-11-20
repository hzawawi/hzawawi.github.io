---
layout: page
title: Make DynamoDB easy for dev
permalink: /dynamodb-post-2/
---

### Challenges:

It gets bit challenging when working with dynamodb to make your development story quite smooth and easy to on-board new contributors to use your solution in a fast manner.
- How to not depend on AWS and the must to provision tables there yet?
- How to get rid of all boiler plate code around creating tables?
- How to cleanup tables and do any table alternation needed?
- How should I implement integration tests?


### Dependencies

- #### AWSSDK.DynamoDBv2 package:
  this is a .NET API that facilitates the interaction with AWS DynamoDB in order to execute different operations against the database such as (createTable, saveItem, retrieveItem,etc..)

- #### [Localstack](https://github.com/localstack/localstack): 
  Localstack is a framework that helps mock different AWS cloud applications; In our example below, we are going to rely on it to mock the Amazon DynamoDB database.
  Localstack is really helpful to use when you want to develop a cloud application offline and reduce dependencies on the cloud infrastructure. 


### Solution:

To tackle this multi-dimensional problem in Pushpay, we built it something we call dynamodb provisioner.

The library is build on top of localstack that will act as the AWS cloud environment, so you dont have to spend any money 
during dev cycle and you dont need to worry about provisioning the table infrastructure early on when it is still in very variable state.


The solution is easy to setup for any given project and get value out of it straight away. You can hook hook it up by registering 
a new middleware or on the init of your tests, I will show some examples later one.
The provisioner will take all responsibility to find all the tables attributed by `DynamoDBTable` and map them correctly to the right dynamo table schema leveraging the AWS API.
You can also choose to persist the data between different runs or demolished on each run which is the favourable behaviour when it comes to unit tests.


### Table naming strategy

- Prefix: to support different environments such as dev/qa and production example:
dev-table, qa-table


-Random: a generated guid that prefix the orginal table name, this will enable us to allow parrallel executiong of different tests
without having table collisions


### Custom attributes:

we have added a couple of extra custom attributes to be able to be able to drive all the table provisioning though table decorations
and make the setup of a new table as easy as possible

- DynamoDBTimeToLiveAttribute: to specify a TTL for a certain table

- DynamoIndexAttribute: to specify different index projection types



### Setup snippets


 #### Service Middleware

 ```
public class CreateTablesMiddleware
    {
        readonly DynamoProvisioner _provisioner;
        readonly RequestDelegate _next;

        public CreateTablesMiddleware(DynamoProvisioner provisioner, RequestDelegate next)
        {
            _provisioner = provisioner;
            _next = next;
        }

        public async Task Invoke(HttpContext context)
        {
            await _provisioner.Setup();

            await _next(context).ConfigureAwait(false);
        }
    }
}

  ```


#### IntegrationFixture


```

public class IntegrationFixture : IAsyncLifetime
    {
        const int DynamoPort = 4569;

        public IntegrationFixture()
        {
            var containerBuilder = new ContainerBuilder();

            var config = new AwsInitializationConfiguration
            {
                DynamoModelAssemblies = new[] {typeof(DynamoProvisioner).Assembly},
                DynamoTableNamingStrategy = TableNamingStrategy.Random,
                UseLocalStack = c => true,
            };
            DynamoConfigurationUtils.RegisterDynamoDb(containerBuilder, config.DynamoTablePrefix, config.UseLocalStack,
                c => BuildServiceUrl(c, config.LocalStackHost, DynamoPort), config.DynamoTableNamingStrategy, config.DynamoModelAssemblies);
            
            Container = containerBuilder.Build();

        }

        public IContainer Container { get; }

        public IDynamoDBContext DynamoDbContext => Container.Resolve<IDynamoDBContext>();

        public static string BuildServiceUrl(IComponentContext c, Func<IComponentContext, string> localstackHost, int port)
        {
            string host = localstackHost == null ? "localhost" : localstackHost(c) ?? "localhost";
            return "http://" + host + ":" + port;
        }

        public Task InitializeAsync()
        {
            return Container.Resolve<DynamoProvisioner>().Setup(performFirstTimeTestTableCleanup: true);
        }

        public Task DisposeAsync()
        {
            return Container.Resolve<DynamoProvisioner>().Cleanup();
        }
    }


```

### Reference
You can find the above code snippets on [github](https://github.com/hzawawi/DynamoProvisioner)