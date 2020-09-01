# Amazon DynamoDB code examples in C#

This folder contains code examples for moving from SQL to NoSQL, 
specifically Amazon DynamoDB,
as described in the Amazon DynamoDB Developer Guide at
[From SQL to NoSQL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SQLtoNoSQL.html).

All of these code examples are written in C#, 
using the beta version of the AWS SDK for .NET.
Getting the 3.5 bits is straightforward using the command line 
from the same folder as your ```.csproj``` file.
For example, to load the beta version of the Amazon DynamoDB bits:

```
dotnet add package AWSSDK.DynamoDBv2 --version 3.5.0-beta
```

## Using asynch/await

Read the 
[Migrating to Version 3.5 of the AWS SDK for .NET](https://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/net-dg-v35.html) 
topic for details.

## Before you write any code

Read the
[Best Practices for Modeling Relational Data in DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-relational-modeling.html)
topic in the Amazon DynamoDB Developer Guide for information about moving 
from a relational database to Amazon DynamoDB.

**IMPORTANT**

NoSQL design requires a different mindset than RDBMS design. 
For an RDBMS, you can create a normalized data model without thinking about access patterns. 
You can then extend it later when new questions and query requirements arise. 
By contrast, in Amazon DynamoDB, 
you shouldn't start designing your schema until you know the questions that it needs to answer. 
Understanding the business problems and the application use cases up front is absolutely essential.

### A simple example

Let's take a simple order entry system,
with just three tables: Customers, Orders, and Products.
- Customer data includes a unique ID, their name, address, email address.
- Orders data includes a unique ID, customer ID, product ID, order date, and status.
- Products data includes a unique ID, description, quantity, and cost.

You might end up with access patterns including the following:

- Get all orders for all customers within a given date range
- Get all orders of a given product for all customers
- Get all products below a given quantity

In a relational database, these might be satisfied by the following SQL queries:

```
select * from Orders where Order_Date between '2020-05-04 05:00:00' and '2020-08-13 09:00:00'
select * from Orders where Order_Product = '3'
select * from Products where Product_Quantity < '100'
```

Given the data in *customers.csv*, *orders.csv*, and *products.csv*,
these queries return (as CSV):

```
Order_ID,Order_Customer,Order_Product,Order_Date,Order_Status
1,1,6,"2020-07-04 12:00:00",pending
11,5,4,"2020-05-11 12:00:00",delivered
12,6,6,"2020-07-04 12:00:00",delivered

Order_ID,Order_Customer,Order_Product,Order_Date,Order_Status
4,4,3,"2020-04-01 12:00:00",backordered
8,2,3,"2019-01-01 12:00:00",backordered

Product_ID,Product_Description,Product_Quantity,Product_Cost
4,"2'x50' plastic sheeting",45,450
```

## Modeling data in Amazon DynamoDB

Amazon DynamoDB supports the following data types,
so you might have to create a new data model:

- Scalar Types

  A scalar type can represent exactly one value.
  The scalar types are number, string, binary, Boolean, and null.

- Document Types
 
  A document type can represent a complex structure with nested attributes,
  such as you would find in a JSON document.
  The document types are list and map.

- Set Types

  A set type can represent multiple scalar values.
  The set types are string set, number set, and binary set.
  
Figure out how you want to access your data.
Many, if not most, stored procedures can be implemented using
[AWS Lambda](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.Lambda.BestPracticesWithDynamoDB.html).

Determine the type of primary key you want:

- Partition key, which is a unique identifier for the item in the table.
  If you use a partition key, every key must be unique.
  The table we create in these code examples will contain 
  a partition key that uniquely identifies a record,
  which can be a customer, an order, or a product.

  Therefore, we'll create some global seconday indices
  to query the table.
  
- Partition key and sort key.
  In this case, you need not have a unique partition key,
  however, the combination of partition key and sort key must be unique.

We'll show you how to create all of these when you create a table,
and how to use them when you access a table.

## Modeling Customers, Orders, and Products in Amazon DynamoDB

Your Amazon DynamoDB schema to model these tables might look like:

| Key | Data Type | Description |
| --- | --- | ---
| ID | String | The unique ID of the item
| Type | String | Customer, Order, or Product
| Customer_ID | Number | The unique ID of a customer
| Customer_Name | String | The name of the customer
| Customer_Address | String | The address of the customer
| Customer_Email | String | The email address of the customer
| Order_ID | Number | The unique ID of an order
| Order_Customer | Number | The Customer_ID of a customer
| Order_Product | Number | The Product_ID of a product
| Order_Date | Number | When the order was made
| Order_Status | String | The status (open, in delivery, etc.) of the order
| Product_ID | Number | The unique ID of a product
| Product_Description | String | The description of the product
| Product_Quantity | Number | How many are in the warehouse
| Product_Cost | Number | The cost, in cents, of one product

## Creating the example databases

We'll use three CSV (comma-separated value) files to define a set of customers,
orders, and products.
Then we'll load that data into a relational database and Amazon DynamoDB.
Finally, we'll run some SQL commands against the relational database,
and show you the corresponding queries or scan against an Amazon DynamoDB table.

The three sets of data are in:

- *customers.csv*, which defines six customers
- *orders.csv*, which defines 12 orders
- *products.csv*, which defines six products

## Default configuration

Every project has an *app.config* file that typically contains the following
configuration values:

```
key="Region" value="us-west-2"
key="Table" value="CustomersOrdersProducts"
```

Therefore, all of the projects that require a table name use the default table
**CustomersOrdersProducts** in the default region **us-west-2**.
Similar values exist for most variable values in all projects.
This means there are few command-line arguments for any executable.

## General code pattern

It's important that you understand the new asynch/await programming model in the
[AWS SDK for .NET](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide).

These code examples use the following NuGet packages:

- AWSSDK.Core, v3.5.0
- AWSSDK,DynamoDBv2, v3.5.0

## Unit tests

We use [moq4](https://github.com/moq/moq4) to create unit tests with mocked objects.
You can install this and the unit testing Nuget packages with:

```
dotnet add package moq
dotnet add package Microsoft.UnitTestFramework.Extensions
```

A typical unit test looks something like the following,
which tests a call to **PutTableAsync**:

```
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.Model;

using Microsoft.VisualStudio.TestTools.UnitTesting;
using Microsoft.VisualStudio.TestTools.UnitTesting.Logging;

using Moq;

using System.Threading.Tasks;
using System.Threading;

namespace DotNetCoreConsoleTemplate
{

    [TestClass]
    public class CreateTableTest
    {
        string tableName = "testtable";

        private IAmazonDynamodDB CreateMockDynamoDBClient()
        {
            var mockDynamoDBClient = new Mock<IAmazonDynamoDB>();
             
            mockDynamoDBClient.Setup(client => client.PutTableAsync(
                It.IsAny<PutTableRequest>(), 
                It.IsAny<CancellationToken>()))
                .Callback<PutTableRequest, CancellationToken>((request, token) =>
                {
                    if(!string.IsNullOrEmpty(tableName))
                    {
                        Assert.AreEqual(tableName, request.TableName);
                    }
                })
                .Returns((PutTableRequest r, CancellationToken token) =>
                {
                    return Task.FromResult(new PutTableResponse());
                });

            return mockDynamoDBClient.Object;
        }

        [TestMethod]
        public async Task CheckCreateTable()
        {
            IAmazonDynamodDB client = CreateMockDynamoDBClient();

            var result = await CreateTable.MakeTable(client, tableName);
            Logger.LogMessage("Created table {0}, tableName);
        }
    }
}
```

## Listing all of the tables in a region

Use the **ListTables** project to list all of the tables in a region.

The default region is defined as **Region** in *app.config*.

## Creating a table

Use the **CreateTable** project to create a table.

The default table name is defined as **Table**
and the default region is defined as **Region**
in *app.config*.

## Listing the items in a table

Use the **ListItems** project to list the items in a table.

The default table name is defined as **Table**
and the default region is defined as **Region**
in *app.config*.

## Adding an item to the table

Use the **AddItem** project to add an item to a table.

The default table name is defined as **Table**
and the default region is defined as **Region**
in *app.config*.

It requires the following options:

- ```-k``` *KEYS*, where *KEYS* is a a comma-separated list of keys
- ```-v``` *VALUES*, where *VALUES* is a a comma-separated list of values

There must be the same number of keys as values.

- If the item is a customer, the schema should match that in *customers.csv*,
  with one additional key, ID, which defines the partition ID for the item.
- If the item is an order, the schema should match that in *orders.csv,
  with one additional key, ID, which defines the partition ID for the item*.
- If the item is a product, the schema should match that in *products.csv,
  with one additional key, ID, which defines the partition ID for the item*.

It's up to you to determine the appropriate partition key value (ID).
If you provide the same value as an existing table item,
the values of that item are overwritten.

## Uploading items to a table

The **AddItems** project incorporates data from three comma-separated value 
(CSV) files to populate a table.

The default table name is defined as **Table**,
the default region is defined as **Region**,
and the default table names are defined as
**Customers**, **Orders**, and **Products**
in *app.config*.

## Listing all of the items in a table

The **ListItems** project lists all of the items in a table.

The default table name is defined as **Table**,
and the default region is defined as **Region**
in *app.config*.

If you previously ran **AddItems**,
there should be 24 items in the table.

## Reading data from a table

You can read data from an Amazon DynamoDB table using a number of techniques.

- By the item's primary key
- By searchng for a particular item or items based on the value of one or more keys

### Reading an item using its primary key

Use the **GetItem** project 
to retrieve information about the customer, order, 
or product with the given primary key.

The default table name is defined as **Table**,
and the default region is defined as **Region**
in *app.config*.

It requires the following command-line option:

- ```-i``` *ID*, which is the partition ID of the item in the table.

If you previously ran **AddItems**,
you can retrieve information about items with an *ID* in the range from 0 to 23.

### Getting orders within a given date range

The **GetOrdersInDateRange** project includes the following method to retrieve
all orders for all customers within the date range from *start* to *end*.

The default table name is defined as **Table**,
the default region is defined as **Region**,
*start* is defined by **StartTime**,
and *end* is defined by **EndTime**
in *app.config*.

Note the required format of the dates: **yyyy-MM-dd HH:mm:ss**.

If you previously ran **AddItems**,
this should return three items:

```
Order_Status: delivered
Order_Customer: 5
Order_Product: 4
Order_ID: 11
Order_Date: Monday, May 11, 2020

Order_Status: pending
Order_Customer: 1
Order_Product: 6
Order_ID: 1
Order_Date: Saturday, July 4, 2020

Order_Status: delivered
Order_Customer: 6
Order_Product: 6
Order_ID: 12
Order_Date: Saturday, July 4, 2020
```

### Getting orders for a given product

The **GetOrdersForProduct** project includes the following method to retrieve
all orders of the product with product ID *productId* for all customers.

The default table name is defined as **Table**,
the default region is defined as **Region**,
and *productId* is defined by **ProductID**
in *app.config*.

If you previously ran **AddItems**,
this should return two items:

```
Order_Status: backordered
Order_Customer: 2
Order_Product: 3
Order_ID: 8
Order_Date: Tuesday, January 1, 2019

Order_Status: backordered
Order_Customer: 4
Order_Product: 3
Order_ID: 4
Order_Date: Wednesday, April 1, 2020
```

### Getting products with fewer than a given number in stock

The **GetLowProductStock** project includes the following method to retrieve
all products with fewer than *minimum* items in stock.

The default table name is defined as **Table**,
the default region is defined as **Region**,
and *minimum* is defined by **Minimum**
in *app.config*.

If you previously ran **AddItems**,
this should return one item:

```
Product_Quantity: 45
Product_Description: 2'x50' plastic sheeting
Product_Cost: 450
Product_ID: 4
```

## Managing indexes

Global secondary indices give you the ability to treat a set of 
Amazon DynamoDB table keys as if they were a separate table.

### Creating an index

Use the **CreateIndex** project to create an index.

The default table name is defined as **Table**,
and the default region is defined as **Region**
in *app.config*.

It requires the following command-line options:

- ```-i``` *INDEX-NAME*, where *INDEX-NAME* is the name of the index
- ```-m``` *MAIN-KEY*, where *MAIN-KEY* is the partition key of the index
- ```-k``` *MAIN-KEY-TYPE*, where *MAIN-KEY-TYPE* is one of string or number
- ```-s``` *SECONDARY-KEY*, where *SECONDARY-KEY* is the sort key of the index
- ```-t``` *SECONDARY-KEY-TYPE*, where *SECONDARY-KEY-TYPE* is one of string or number

To create a global secondary index (GSI) for customers, orders, and products,
execute the following commands, 
where the **-m** flag defines the main (partition key) value 
and the **-s** flag defines the secondary (sort key) value:

```
CreateIndex.exe -i Customers -m Customer_ID -k number -s Customer_Email   -t string
CreateIndex.exe -i Orders    -m Order_ID    -k number -s Order_Date       -t number
CreateIndex.exe -i Products  -m Product_ID  -k number -s Product_Quantity -t number
```

Note that you cannot execute these commands one right after another.
You must wait until one GSI is created before you can attempt to create another GSI.
We recommend you use the Amazon DynamoDB console to monitor the progress
of creating a GSI to avoid errors.

## Modifying a table item

Use the **UpdateItem** project to modify the status of an order in the table.

The default table name is defined as **Table**,
and the default region is defined as **Region**
in *app.config*.

It takes the following options:

- ```-i``` *ID*, where *ID* is the value of the order's ORDER_ID attribute
- ```-s``` *STATUS*, where *STATUS* is the new status value (backordered, delivered, delivering, or pending)

This code example demonstrates two different levels of accessing an item in a table.
- If you set the status (```-s``` *STATUS*) to **pending**,
  the code example uses the **DynamoDBContext** class to load the table.
  If the item is not an order, the update silently fails.
- If you set the status to any other valid value,
  the code example uses the lower-level **UpdateItemAsync** method,
  which throws an exception if the item is not an order.

## Deleting an item from a table

Use the **DeleteItem** project to delete an item from the table.

The default table name is defined as **Table**,
and the default region is defined as **Region**
in *app.config*.

It takes the following option:

- ```-p``` *PARTITION*, where *PARTITION* is the value of the partition key
- ```-s``` *AREA*, where *AREA* is **Customer**, **Order**, or **Product**.

If you provide an *AREA* value that does not match that of the item,
the example silently fails to delete the item from the table.

## Deleting a table

Use the **DeleteTable** project to delete a table.

The default table name is defined as **Table**,
and the default region is defined as **Region**
in *app.config*.

