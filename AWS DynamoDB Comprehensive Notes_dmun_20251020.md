# **AWS DynamoDB Comprehensive Notes**

Compiled by: Douglas Mun, aided by Google Gemini AI.

## **1\. What is Amazon DynamoDB?**

Amazon DynamoDB is a fully managed, serverless **NoSQL** database service provided by AWS. It is designed for high performance and massive scalability, supporting both **document** and **key-value** store data models.

* **Key Strength:** Offers single-digit millisecond latency at any scale, making it ideal for high-traffic web/mobile apps, gaming, ad-tech, and IoT.  
* **Serverless:** No infrastructure to manage. AWS handles hardware provisioning, setup, configuration, scaling, and replication.

## **2\. Setting up the AWS CLI (Prerequisite)**

Before interacting with DynamoDB via the command line, you need to configure your local machine with your access keys.

### **2.1 Install and Verify**

First, ensure the AWS CLI is installed and check its version:  
aws \--version  
\# Example output: aws-cli/2.15.50 Python/3.11.8 Linux/6.6.13-23.77.amzn2023.x86\_64 exe/x86\_64.amzn.2023

### **2.2 Configure Credentials (aws configure)**

This command guides you through setting up your security credentials and default region.  
aws configure

| Prompt | Example Input | Notes |
| :---- | :---- | :---- |
| AWS Access Key ID \[None\]: | AKIAIOSFODNN7EXAMPLE | Get this from your IAM console. |
| AWS Secret Access Key \[None\]: | wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY | This is highly sensitive. |
| Default region name \[None\]: | us-east-1 | Use the region where your DynamoDB table resides. |
| Default output format \[None\]: | json | Recommended format for machine-readable output. |

## **3\. Basic Concepts of DynamoDB**

### **Primary Key (Fundamental)**

Every table **must** have a primary key, which uniquely identifies each item.

| Key Type | Structure | Description |
| :---- | :---- | :---- |
| **Partition Key** (Simple Key) | Single attribute (Hash Key) | Uniquely identifies the item. Data is distributed across partitions based on the hash of this key. **Best for high-cardinality keys.** |
| **Composite Key** | Partition Key (Hash Key) **\+** Sort Key (Range Key) | The *combination* of both keys uniquely identifies the item. Items with the same Partition Key are stored together and ordered by the Sort Key. This enables efficient Query operations on the Sort Key. |

### **Attributes (Filling in Gaps \- Data Types)**

Attributes are the fundamental data elements (fields/columns). Since DynamoDB is schema-less, you do not need to pre-define them (except for the Primary Key attributes).  
When inserting data via the AWS CLI or API, you **must** specify the attribute's data type using the following type descriptors:

| Type Descriptor | Data Type | Description | Example (CLI JSON) |
| :---- | :---- | :---- | :---- |
| **S** | String | Unicode with a maximum length of 400KB. | {"S": "John Doe"} |
| **N** | Number | Positive, negative, or zero. Stored as a string. | {"N": "30"} |
| **B** | Binary | Stores unencoded binary data. | {"B": "MTIzNDU="} (Base64 encoded) |
| **SS, NS, BS** | Set | Stores a set of unique Strings, Numbers, or Binaries. | {"NS": \["100", "200", "300"\]} |
| **L** | List | Stores an ordered list of values (similar to an array). | {"L": \[{"S": "A"}, {"N": "1"}\]} |
| **M** | Map | Stores an unordered collection of name-value pairs (JSON object). | {"M": {"city": {"S": "NY"}}} |
| **BOOL** | Boolean | True or False. | {"BOOL": true} |
| **NULL** | Null | Indicates the attribute is null. | {"NULL": true} |

## **4\. Administrative Table Operations**

### **4.1 Creating a DynamoDB Table (CLI)**

While creating a table in the Console is straightforward, here is the AWS CLI command for creating the Test1 table with a composite key:  
\# Create table Test1 with PartitionKey1 (String) and SortKey1 (String)  
aws dynamodb create-table \\  
    \--table-name Test1 \\  
    \--attribute-definitions \\  
        AttributeName=PartitionKey1,AttributeType=S \\  
        AttributeName=SortKey1,AttributeType=S \\  
    \--key-schema \\  
        AttributeName=PartitionKey1,KeyType=HASH \\  
        AttributeName=SortKey1,KeyType=RANGE \\  
    \--provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \\  
    \--table-class STANDARD \\  
    \--region us-east-1

### **4.2 Deleting a DynamoDB Table (CLI)**

To completely remove a table and all its data, use the delete-table command. This action is irreversible.

* **Resource Cleanup Note:** Before deleting a table, you must first delete any associated **Global Secondary Indexes (GSI)** using the update-table command.

\# Deletes the specified table and all associated data  
aws dynamodb delete-table \\  
    \--table-name Test1 \\  
    \--region us-east-1

### **4.3 Listing All Tables (CLI)**

Use list-tables to see all tables in the specified region under your account.  
\# Lists all table names in the specified region (e.g., us-east-1)  
aws dynamodb list-tables \\  
    \--region us-east-1

### **4.4 Describing a Specific Table (CLI)**

Use describe-table to retrieve detailed metadata about a table, including its status, primary key schema, GSIs, and provisioned capacity.  
\# Retrieves detailed information about the structure of Test1  
aws dynamodb describe-table \\  
    \--table-name Test1 \\  
    \--region us-east-1

## **5\. Data Manipulation: Inserting and Updating**

### **5.1 Inserting Data (PutItem)**

The put-item command completely overwrites an item if it already exists.  
aws dynamodb put-item \\  
    \--table-name Test1 \\  
    \--item '{  
        "PartitionKey1": {"S": "Contact"},  
        "SortKey1": {"S": "name\#John Doe"},  
        "id": {"S": "1"},  
        "name": {"S": "John Doe"},  
        "age": {"N": "30"},  
        "email": {"S": "john.doe@example.com"},  
        "tags": {"L": \[{"S": "VIP"}, {"S": "Customer"}\]}  
    }' \\  
    \--region us-east-1

### **5.2 Updating an Item (UpdateItem)**

The update-item command modifies specific attributes of an item without replacing the entire item.  
\# Example: Updating John Doe's age (N) to 31 and adding a new attribute 'city' (S)  
aws dynamodb update-item \\  
    \--table-name Test1 \\  
    \--key '{"PartitionKey1": {"S": "Contact"}, "SortKey1": {"S": "name\#John Doe"}}' \\  
    \--update-expression "SET \#a \= :newAge, \#c \= :newCity" \\  
    \--expression-attribute-names '{"\#a": "age", "\#c": "city"}' \\  
    \--expression-attribute-values '{":newAge": {"N": "31"}, ":newCity": {"S": "New York"}}' \\  
    \--region us-east-1

* **Note:** Using SET with Expression Attribute Names (\#a, \#c) prevents issues if your attribute names conflict with DynamoDB reserved words.

## **6\. Data Manipulation: Reading and Deleting**

### **6.1 Get Item by Primary Key (Exact Match)**

Use get-item for the fastest single-item retrieval, requiring the full primary key.  
\# Retrieves the single item matching both keys exactly  
aws dynamodb get-item \\  
    \--table-name Test1 \\  
    \--key '{"PartitionKey1": {"S": "Contact"}, "SortKey1": {"S": "name\#John Doe"}}' \\  
    \--region us-east-1

## **6.2 Querying by Key Condition (Range and Prefix Matching)**

The query operation is the most efficient way to retrieve multiple items that share the same Partition Key. It is highly efficient and scalable.  
\# Retrieves all items where PartitionKey1 is "Contact" and the SortKey1 starts with "name"  
aws dynamodb query \\  
    \--table-name Test1 \\  
    \--key-condition-expression "PartitionKey1 \= :pkValue AND begins\_with(SortKey1, :skValue)" \\  
    \--expression-attribute-values '{  
        ":pkValue": {"S": "Contact"},  
        ":skValue": {"S": "name"}  
    }' \\  
    \--region us-east-1

### **6.2.1 Handling Pagination (Large Result Sets)**

Both Query and Scan operations can retrieve a maximum of **1MB** of data per request. If the result set is larger, DynamoDB returns a partial result along with an LastEvaluatedKey.

* To retrieve the next page of results, you must include the returned LastEvaluatedKey in your subsequent request using the \--exclusive-start-key parameter. This is how pagination is handled manually in DynamoDB.

\# Subsequent query request to fetch the next 1MB of data  
aws dynamodb query \\  
    \--table-name Test1 \\  
    \--key-condition-expression "PartitionKey1 \= :pkValue" \\  
    \--expression-attribute-values '{":pkValue": {"S": "Contact"}}' \\  
    \--exclusive-start-key '{"PartitionKey1": {"S": "Contact"}, "SortKey1": {"S": "last\_key\_from\_previous\_result"}}' \\  
    \--region us-east-1

### **6.3 Scanning the Table (Avoid for Large Tables)**

scan reads every item in the table. It is resource-intensive and should be avoided for production reads on large tables.  
aws dynamodb scan \--table-name Test1 \--region us-east-1

### **6.4 Deleting an Item (DeleteItem)**

The delete-item command removes an entire item from the table.  
\# Example: Deleting the item for John Doe  
aws dynamodb delete-item \\  
    \--table-name Test1 \\  
    \--key '{"PartitionKey1": {"S": "Contact"}, "SortKey1": {"S": "name\#John Doe"}}' \\  
    \--region us-east-1

### **6.5 Read Consistency Models (Fundamental)**

DynamoDB offers two types of read consistency. This impacts latency and cost.

| Consistency Type | Description | Best Practice |
| :---- | :---- | :---- |
| **Eventually Consistent (Default)** | The response might not reflect the result of a recently completed write operation. It usually takes less than a second for consistency to be achieved. **Cheaper and faster.** | Use this for non-critical data where a slight delay is acceptable (e.g., social media feeds, session carts). |
| **Strongly Consistent** | The response reflects all successful write operations that occurred before the read. **More expensive (consumes 2x the capacity units).** | Use this when absolute data integrity is required (e.g., financial transactions, inventory checks). |

To request a Strongly Consistent Read using the AWS CLI:  
\# Note the "ConsistentRead": true flag is set.  
aws dynamodb get-item \\  
    \--table-name Test1 \\  
    \--key '{"PartitionKey1": {"S": "Contact"}, "SortKey1": {"S": "name\#John Doe"}}' \\  
    \--consistent-read \\  
    \--region us-east-1

## **7\. Transactions (Multi-Item ACID Operations)**

DynamoDB Transactions allow you to group multiple read or write operations together into an all-or-nothing set. Transactions provide **Atomicity, Consistency, Isolation, and Durability (ACID)** across one or more tables within a single AWS account and region.

| Operation | Description | Use Case |
| :---- | :---- | :---- |
| **TransactWriteItems** | Atomically commit a batch of up to 10 write operations (Put, Update, Delete, or ConditionCheck). | Transferring funds between two bank accounts (must succeed or fail together). |
| **TransactGetItems** | Atomically retrieve a batch of up to 10 items. | Reading a composite record (e.g., User item \+ Settings item) to ensure both are consistent. |

**Key Takeaway:** While standard single-item operations are fast, use transactions only when multi-item ACID guarantees are essential, as they incur higher latency and cost.

## **8\. Capacity Unit Math (RCU and WCU)**

Capacity Units (CUs) are the metric DynamoDB uses for throughput and pricing in **Provisioned Mode**.

### **8.1 Read Capacity Units (RCU)**

A single RCU provides:

* **4 KB** of data read per second, using **Eventually Consistent** reads.  
* **2 KB** of data read per second, using **Strongly Consistent** reads.

| Operation | Item Size | RCU Cost (Eventually Consistent) |
| :---- | :---- | :---- |
| GetItem (4 KB item) | 4 KB | 1 RCU |
| GetItem (5 KB item) | 5 KB | 2 RCU (always rounds up to the nearest 4 KB block) |
| Query/Scan (10 items, 2 KB each) | 20 KB total | 5 RCU (20 / 4 \= 5\) |

### **8.2 Write Capacity Units (WCU)**

A single WCU provides:

* **1 KB** of data written per second.

| Operation | Item Size | WCU Cost |
| :---- | :---- | :---- |
| PutItem (1 KB item) | 1 KB | 1 WCU |
| PutItem (1.5 KB item) | 1.5 KB | 2 WCU (always rounds up to the nearest 1 KB block) |

**Note on Transactions:** Transactional operations consume **2x** the capacity units of standard operations.

## **9\. Conditional Writes (Ensuring Data Integrity)**

Conditional operations allow you to perform Put, Update, or Delete only if a specific condition about the item's existing attributes is met. This is vital for managing concurrency.

### **9.1 Preventing Overwrites on Insert (PutItem)**

Use a ConditionExpression to ensure an item with the specified primary key does *not* already exist before performing a PutItem.  
\# Only insert the item if an item with this PK and SK does NOT exist.  
\# The attribute\_not\_exists(PartitionKey1) check is sufficient.  
aws dynamodb put-item \\  
    \--table-name Test1 \\  
    \--item '{  
        "PartitionKey1": {"S": "Contact"},  
        "SortKey1": {"S": "name\#Jane Doe"},  
        "email": {"S": "jane.doe@example.com"},  
        "version": {"N": "1"}  
    }' \\  
    \--condition-expression "attribute\_not\_exists(PartitionKey1)" \\  
    \--region us-east-1

### **9.2 Optimistic Locking (UpdateItem)**

This pattern uses a version number attribute (version) to ensure that two users don't try to update the same item simultaneously based on stale data. The update only succeeds if the current version matches the expected version.  
\# ASSUMPTION: The current item's version is 1\.  
\# This command updates the email and ONLY SUCCEEDS if the existing version is 1\.  
\# It then atomically increments the version to 2\.  
aws dynamodb update-item \\  
    \--table-name Test1 \\  
    \--key '{"PartitionKey1": {"S": "Contact"}, "SortKey1": {"S": "name\#John Doe"}}' \\  
    \--update-expression "SET \#e \= :newEmail, \#v \= \#v \+ :inc" \\  
    \--condition-expression "\#v \= :expectedVersion" \\  
    \--expression-attribute-names '{  
        "\#e": "email",  
        "\#v": "version"  
    }' \\  
    \--expression-attribute-values '{  
        ":newEmail": {"S": "john.updated@example.com"},  
        ":inc": {"N": "1"},  
        ":expectedVersion": {"N": "1"}  
    }' \\  
    \--region us-east-1

* **Result:** If the condition fails (someone else updated it first), DynamoDB returns a ConditionalCheckFailedException.

## **10\. Next Steps and Architectural Best Practices**

| Feature | Description | Best Practice |
| :---- | :---- | :---- |
| **Global Secondary Indexes (GSI)** | Allow querying on non-primary key attributes by creating an entirely new key structure (e.g., query by email). | Use GSIs sparingly and only when the access pattern requires querying by a different attribute. |
| **Local Secondary Indexes (LSI)** | Provides an alternate Sort Key for the *same* Partition Key. | Use LSIs when you need multiple sort orders for the same grouping of data. |
| **Design for Access Patterns** | DynamoDB is not flexible SQL. Design the key structure around how you plan to access the data. | **Principle:** All items that are retrieved together should be stored together (sharing the same Partition Key). |
| **Composite Keys (SK Design)** | Use concatenated strings (e.g., USER\#123 or ORDER\#PENDING) as Sort Keys to store different entity types under the same Partition Key. This is called **Single-Table Design**. | Always look for ways to combine related data into a single table using sophisticated Sort Key schemes. |
| **Capacity Mode** | Choose between **Provisioned** (fixed RCU/WCU, cost-efficient for predictable load) and **On-Demand** (pay-per-request, good for spiky/unpredictable load). | Start with **On-Demand** unless you have a clear, steady traffic profile. |

### **10.1 DynamoDB Streams (Event-Driven Architecture)**

DynamoDB Streams provide a time-ordered sequence of item-level modification records (create, update, delete) for a table.

* **Function:** Captures changes in near real-time.  
* **Use Case:** Triggers event-driven processes, such as updating secondary databases or sending notifications using an AWS Lambda function.

### **10.2 Time-To-Live (TTL)**

TTL is a mechanism that allows you to define an expiration timestamp on specific attributes of your items.

* **Function:** Automatically deletes items that have expired, typically within 48 hours of expiration.  
* **Benefit:** **Cost-Effective Cleanup.** Deletion occurs in the background and does **not** consume any of your table's Write Capacity Units (WCU).  
* **Use Case:** Clearing old session data, expired promotional codes, or log entries.

### **10.3 DynamoDB Accelerator (DAX)**

**DAX** is a fully managed, highly available, in-memory cache for DynamoDB.

* **Benefit:** Reduces read request times from milliseconds to **microseconds** for read-heavy workloads.  
* **Mechanism:** It handles replication, encryption at rest, and request routing, simplifying caching logic for the application.

### **10.4 Multi-Region High Availability (Global Tables)**

DynamoDB supports **Global Tables**, which provide a fully managed, multi-region, and multi-active database.

* **Function:** Automatically replicates data across chosen AWS regions using DynamoDB Streams.  
* **Benefit:** Offers low-latency global access and robust disaster recovery by protecting your data from region-specific outages.

### **10.5 Broader AWS Service Integration**

DynamoDB integrates tightly with other AWS services to build full-stack serverless applications:

* **AWS Lambda:** Triggered by DynamoDB Streams for real-time processing.  
* **AWS Step Functions:** Build complex, scalable workflows and data pipelines.  
* **Amazon EMR / Redshift:** Analyze data changes captured from DynamoDB.  
* **Amazon SageMaker:** Employ machine learning models using DynamoDB data.  
* **AWS Backup:** Provides automated backup and restoration capabilities.

## **11\. Deep Dive: Single-Table Design (STD)**

Single-Table Design (STD) is a best practice for DynamoDB where multiple logical entities (e.g., Users, Orders, Products) are stored in a single table, rather than one table per entity (Multi-Table Design). This is achieved using highly descriptive **Composite Keys**.

#### **Core Principles of STD**

1. **Uniform Key Names:** Use generic attribute names for the Primary Key, typically PK (Partition Key) and SK (Sort Key), regardless of the entity type.  
2. **Entity Delimiting:** Use prefixes in the key values to differentiate entity types (e.g., USER\#123, ORDER\#987). This allows you to combine related items under one key.  
3. **Co-Location:** The goal is to co-locate all data needed for a specific access pattern under a single Partition Key, enabling a single, fast Query operation instead of multiple requests (or expensive Scan/Filter operations).

#### **STD Example: User and Orders**

Consider an application needing to retrieve a User's profile and their open orders.

| Entity Type | PK Value (Partition Key) | SK Value (Sort Key) | Attributes |
| :---- | :---- | :---- | :---- |
| **User** | USER\#\<UserID\> | USER\#\<UserID\> | username, email, address |
| **Order** | USER\#\<UserID\> | ORDER\#\<OrderID\> | status, total, orderDate |

Access Pattern 1: Get User Profile (Exact Match)  
To get the user's profile, the PK and SK are an exact match:  
Query where PK \= USER\#123 AND SK \= USER\#123  
Access Pattern 2: Get User Profile AND All Orders (One Query)  
To get the user's profile and all their orders in a single, efficient request, we leverage the Sort Key prefix:  
Query where PK \= USER\#123 AND SK begins\_with USER\# (or just ORDER\# to skip the profile).  
\# Conceptual Query for User's Profile and All Orders (SK \>= USER\#)  
aws dynamodb query \\  
    \--table-name SingleTable \\  
    \--key-condition-expression "PK \= :pk AND SK \>= :skPrefix" \\  
    \--expression-attribute-values '{  
        ":pk": {"S": "USER\#123"},  
        ":skPrefix": {"S": "USER\#"}  
    }' \\  
    \--region us-east-1

#### **Benefits of Single-Table Design**

* **Transactional Integrity:** Allows use of TransactWriteItems across multiple entities (e.g., updating a User record and creating a new Order record atomically).  
* **Reduced Cost/Overhead:** Fewer services (tables) to manage and potentially fewer requests needed (one Query replaces multiple GetItem calls).  
* **Query Performance:** Maximizes the use of the highly efficient Query operation by grouping related data.

## **12\. Advanced Security and Access Control**

DynamoDB provides robust security features to secure sensitive data:

* **Encryption at Rest (KMS):** All data can be fully encrypted before being written to disks using AWS Key Management Service (KMS).  
* **Granular Access Control (IAM):** Precisely control which users/resources can access which DynamoDB operations using IAM policies.  
  * **Item-Level Permissions:** IAM policies can restrict access down to specific items or even item attributes using condition keys.  
* **Audit Logs:** AWS CloudTrail provides audit logs, giving visibility into all access and modification activities on your tables.

The custom policy example below demonstrates the principle of **Least Privilege**:  
{  
    "Version": "2012-10-17",  
    "Statement": \[  
        {  
            "Effect": "Allow",  
            "Action": "dynamodb:GetItem",  
            "Resource": "arn:aws:dynamodb:us-east-1:177099686723:table/Test1"  
        }  
    \]  
}

Action: dynamodb:GetItem is the least privilege required for simple reads.  
Resource: The ARN clearly scopes the permission to the Test1 table in us-east-1.

## **13\. Resource Cleanup (Best Practice)**

After completing a tutorial or project, ensure all AWS resources are properly deleted to avoid unnecessary charges.

1. **Remove Secondary Indexes:** If you created any GSIs, delete them before deleting the main table.  
   \# Removes an index named AuthorIndex from the Books table  
   aws dynamodb update-table \\  
       \--table-name Books \\  
       \--attribute-definitions AttributeName=Authors,AttributeType=S \\  
       \--global-secondary-index-updates '\[{"Delete": {"IndexName": "AuthorIndex"}}\]' \\  
       \--region us-east-1

2. **Delete the DynamoDB Table:**  
   aws dynamodb delete-table \\  
       \--table-name Books \\  
       \--region us-east-1

3. **Delete IAM Policies:** If you created custom IAM policies, delete them via the IAM CLI (or Console).  
   \# Note: Requires the policy ARN or the policy name depending on the command used  
   aws iam delete-policy \\  
       \--policy-arn arn:aws:iam::123456789012:policy/DynamoDBReadOnlyPolicy  
