# Automated Data Discovery &  Access Control

## Background & Problem Statement

The understanding of data is very important for the enterprises. This helps them to add required access control on the data and ensure also helps them to meet required regulatory and compliance related requirements. Key challenges for companies is to understand what kind of data is stored and how to create permissions / access control to restrict the data access. For e.g. how companies can know which tables and columns has confidential information and how can they restrict access to sensitive information only for specific users groups.

This article will explain how AWS partner and customers can use AWS Lake Formation APIs to create policies, tag resources & create catalog. We can also use services like Amazon Macie or a partner product like BigID to discover confidential information. 


## What is AWS Lake Formation?

AWS Lake Formation is a fully managed service that makes it easier for you to build| secure| and manage data lakes. Lake Formation simplifies and automates many of the complex manual steps that are usually required to create data lakes. These steps include collecting, cleansing, moving, and cataloging data, and securely making that data available for analytics and machine learning. You point Lake Formation at your data sources, and Lake Formation crawls those sources and moves the data into your new Amazon Simple Storage Service (Amazon S3) data lake.


## Solution

Following diagram will provide details on the proposed solution. 
[Image: Overview.png]
This article will guide you through the following tasks. The above diagram will provide overview of these steps. 

* Get input from user. This input will be table name.
* Use AWS Glue APIs to get know the table name and location of the data store.
* Identify some attributes as PII and confidential. This will be hard-coded values as of now, and it can be replaced by partner solution or Amazon Macie to discover confidential information. 
* Create and assign tags to the AWS Lake Formation tables

## Assumptions

Following are the assumptions for this article.

* AWS lake formation is already created and the customer has already loaded data & created tables in it.
* Customer has added some tags in tables and columns

## Step 1 - Lake Formation Setup

In the lake formation, we have created a table using glue crawler. The name of the table is **customer_information** & this table is present in **lfdatabase** database of AWS LakeFormation. This article assumes that you already have created tables, database etc. in the AWS lakeformation. Hence, we will not go in details of how to create tables, database and load data in the AWS LakeFormation.

Following is the structure of Table - Customer Information. It has some general or public columns and some confidential columns like SSN & medical issues.
[Image: image.png]The table name customer_information will be passed to the program to assign tags. 

## Step 2 - Get User input & connect to AWS Glue APIs to get the location information

Following is a link of AWS Glue related boto APIs.

<https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/glue.html>

Following is a sample code to connect to AWS Glue & get table information.


```
#Author : Chintan Sanghavi
#Date: Sept 2021

#This python code will connect to AWS Glue and will retrieve table specific information. 
# In addition to this, it will also tag resources in Lake Formation tables. 

# Import required libraries. 
import boto3

# get boto client for AWS Glue. 
glue_client = boto3.client('glue')

# Table information
table_information = glue_client.get_table(DatabaseName='lfdatabase', Name='customer_information')

# Print table information
print(table_information)
```

The above code will connect to AWS glue table customer_information & will get information like location, creator etc. For the purpose of this article, we are more interested in getting location information.

The output from above-mentioned code will be the following:

```
{'Table': {'Name': 'customer_information', 
 'DatabaseName': 'lfdatabase', 
 'Owner': 'owner', 
 'Retention': 0, 
 'StorageDescriptor': 
  {'Columns': [
    {'Name': 'sr_no', 'Type': 'bigint'},
    {'Name': 'name', 'Type': 'string'},
    {'Name': 'address', 'Type': 'string'},
    {'Name': 'ssn', 'Type': 'string'},
    {'Name': 'phone', 'Type': 'string'},
    {'Name': 'zip_code', 'Type': 'bigint'},
    {'Name': 'kyc', 'Type': 'string'},
    {'Name': 'interest', 'Type': 'string'},
    {'Name': 'medical_issues', 'Type': 'string'}
   ], 
 **'Location': 's3://chintan20/lfstaging/', 
**   'InputFormat': 'org.apache.hadoop.mapred.TextInputFormat', 

.... Output Trimmed for simplicity

}
```

As you can see in the output, there is a json tag for location. This location will provide the location of S3 bucket where data is stored for this table.

The following code will extract location information from the above output.

```
#Following code will extract json tags. It will accept json object and key to be extracted. 

def json_extract(obj, path):

    # Nested function to get path
    def extract(obj, path, ind, arr):
        key = path[ind]
        if ind + 1 < len(path):
            if isinstance(obj, dict):
                if key in obj.keys():
                    extract(obj.get(key), path, ind + 1, arr)
                else:
                    arr.append(None)
            elif isinstance(obj, list):
                if not obj:
                    arr.append(None)
                else:
                    for item in obj:
                        extract(item, path, ind, arr)
            else:
                arr.append(None)
        if ind + 1 == len(path):
            if isinstance(obj, list):
                if not obj:
                    arr.append(None)
                else:
                    for item in obj:
                        arr.append(item.get(key, None))
            elif isinstance(obj, dict):
                arr.append(obj.get(key, None))
            else:
                arr.append(None)
        return arr
    if isinstance(obj, dict):
        return extract(obj, path, 0, [])
    elif isinstance(obj, list):
        outer_arr = []
        for item in obj:
            outer_arr.append(extract(item, path, 0, []))
        return outer_arr

print("Location of Table Storage is =  ",  json_extract(table_information, ["Table", "StorageDescriptor", "Location"])[0] )

```


The output of this code will be the following. This will provide the underlying S3 location. 

```

Location of Table Storage is =  s3://chintan20/lfstaging/

```

## Step # 3 - Add data related information and identify tags

Once we have received the S3 location, we can use Amazon Macie or other partner products or internal logic to tag specific table and data as PII or public data.

The implementation of this will be part of the next article. As of now, let's assume that this table stores PII & private information, since it has SSN and medical issues.


## Step 4 - Add these tags to the AWS Lake Formation tables and columns

In this step, we will create AWS LakeFormation tags if they are not already assigned to table and columns.

We will create following tags:

1. At table level: Tag Name will be Data_Classification & value will be PII
2. At column level: All columns except SSN & Medical Issues will have data_classification tag as Public
3. SSN Column will have classification as PII
4. Medical Issues will have data_classification as Private

The following code will connect to the AWS LakeFormation client and will retrieve tags as well as create new tags if they are not already created.


```
 Created by Chintan
# Date: 13th Oct 2021
# Purpose: To create & assign tags to the AWS LakeFormation tables

lakeformation_client = boto3.client('lakeformation')

table_information = lakeformation_client.list_lf_tags() # This will get all tags present in the lake formation. 

print(table_information)

numberOfLFTags = json_extract(table_information, ["LFTags"])

print("Number of LF Tags =  ",  numberOfLFTags)

if numberOfLFTags[0] == None: 
 print("There are no tags")
 # Create tags relevant to Data Classification
 lakeformation_client.create_lf_tag(
      TagKey='data_classification',
      TagValues=[
          'Sensitive',
   'Private',
   'Internal',
   'Public'
       ]
  ) 
else:
 for tag in numberOfLFTags: 
  print("Current Tag in Processing is ", tag)
  tagName = json_extract(tag, ["TagKey"])
  print("Tag Name =", tagName[0][0])
  if tagName[0][0] == "data_classification" :
   print("Data classification tag is already present. Hence, no need to add new tag")
  else:
   print("No tag present and need to add new tag")
   lakeformation_client.create_lf_tag(
        TagKey='data_classification',
        TagValues=[
            'Sensitive',
     'Private',
     'Internal',
     'Public'
        ]
   ) 

```


As shown in the image below, the tag will be created if tag is not present in the AWS Lake Formation.
[Image: data_classifications.png]
If tag is already present in the AWS Lake Formation then the output will be as shown in the following image.
[Image: output1.png]Now, we have identified the presence of tag in the AWS lakeformation. Now, we need to identify if the tag is assigned to tables and columns or not. In our case, the table will have data_classification tag as sensitive and all columns except SSN and Medical Issues will have data classification tag as public. SSN will have data classification tag as Sensitive and medical issues will have data classification tag as Private.

We will be using following two APIs of AWS Lake Formation to find out presence of these tags.

1. [get_resource_lf_tags](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/lakeformation.html#LakeFormation.Client.get_resource_lf_tags): This will provide list of tags assigned to the resource. 
2. [add_lf_tags_to_resource](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/lakeformation.html#LakeFormation.Client.add_lf_tags_to_resource): This will add a new tag to the resource.


Following lines of code will get the tags assigned to the databases

```
# Get resource lf tag API will provide list of all tags assigned to the resources. 
resource_tags = lakeformation_client.get_resource_lf_tags(
    Resource={
        'Database': {
            'Name': 'lfdatabase'
        },
        'Table': {
            'DatabaseName': 'lfdatabase',
            'Name': 'customer_information',
        },
    'TableWithColumns': {
             'DatabaseName': 'lfdatabase',
             'Name': 'customer_information',
            'ColumnNames': [
                'ssn'
            ]
        },
    },
    ShowAssignedLFTags=True|False
)
```

In the above code, we are passing database as a resource. The following code can be used to get the list of tags on tables and columns. 

```
resource_tags = lakeformation_client.get_resource_lf_tags(
    Resource={
        'Table': {
            'DatabaseName': 'lfdatabase',
            'Name': 'customer_information',
        }
    },
    ShowAssignedLFTags=True|False
)
```


Once we have received a list of all tags, we can iterate through it to find out if the data classification related tag is assigned or not. If data classification related tag is not assigned, then you can get **add_lf_tags_to_resource** API to add new tags to the resources. Following is a sample code to add new tags to database & columns. 


```
# Following is a sample code to add tags to the database. 

        Resource={
                'Table': {
                'DatabaseName': 'lfdatabase',
                'Name': 'customer_information',
                },
            },
            LFTags=[
                {
                'TagKey': 'data_classification',
                'TagValues': [
                    'Sensitive',
                    ]
                },
            ]
        )
```



```
# code to add tags to the columns. Here I am adding Sensitive tag to SSN column. 
    response = lakeformation_client.add_lf_tags_to_resource(
    Resource={
        'TableWithColumns': {
        'DatabaseName': 'lfdatabase',
        'Name': 'customer_information',
        'ColumnNames': [
            'ssn',
        ]
        },
    },
    LFTags=[
        {
        'TagKey': 'data_classification',
        'TagValues': [
            'Sensitive',
        ]
        },
    ]
    )
```


Once tags are added, you will be able to see these tags in the lake formation console as shown in the image below. 
[Image: assigned_tags.png]
## Summary

The AWS Lake Formation & AWS Glue provides various APIs to perform automations like assigning tags or creating catalog etc. These APIs can be used by partners to enhance & differentiate their solution. 
