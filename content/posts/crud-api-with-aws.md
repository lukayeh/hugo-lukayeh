---
title: "CRUD API with AWS Lambda, DynamoDB and API Gateway Using Terraform"
date: 2022-07-26
draft: false
tags:
 - "dynamodb"
 - "terraform"
 - "lambda"
 - "aws"
categories:
 - "techstuff"
---

## Prerequisites
For this tutorial, you will need:

- The Terraform CLI (1.0.1+) installed.
- An AWS account.
- The AWS CLI (2.0+) installed, and configured for your AWS account.
- A copy of https://github.com/lukayeh/lambda-api-202207261251

## Introduction

This tutorial will enable you to provision AWS Lamdba, Api Gateway and DynamoDB to enable a serverless API that creates, reads, updates, and deletes items from a DynamoDB table.

The functionality was based on following this AWS tutorial:  [Tutorial: Build a CRUD API with Lambda and DynamoDB
](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-dynamo-db.html#http-api-dynamo-db-create-table).

Before proceeding make sure to pull down https://github.com/lukayeh/lambda-api-202207261251 as this tutorial will be a simple walk through then a run of the code, and less like my usual step by step! 

## The terminology

If like me you need a reminder on the terminology I'll dump some information around the key terms we'll be using.

### API Gateway

>Amazon API Gateway helps developers to create and manage APIs to back-end systems running on Amazon EC2, AWS Lambda, or any publicly addressable web service. With Amazon API Gateway, you can generate custom client SDKs for your APIs, to connect your back-end systems to mobile, web, and server applications or services.

### Lambda

>AWS Lambda is a serverless, event-driven compute service that lets you run code for virtually any type of application or backend service without provisioning or managing servers. You can trigger Lambda from over 200 AWS services and software as a service (SaaS) applications, and only pay for what you use.

### DynamobDB

>Amazon DynamoDB is a fully managed, serverless, key-value NoSQL database designed to run high-performance applications at any scale. DynamoDB offers built-in security, continuous backups, automated multi-Region replication, in-memory caching, and data export tools.

### NoSQL

>NoSQL databases (aka "not only SQL") are non-tabular databases and store data differently than relational tables. NoSQL databases come in a variety of types based on their data model. The main types are document, key-value, wide-column, and graph. They provide flexible schemas and scale easily with large amounts of data and high user loads.

## Explaining the code

The code tree looks like the following:
```
.
├── http-crud-application
│   ├── http-crud-tutorial-function.zip
│   └── index.js
├── readme.md
└── terraform
    ├── apigateway.tf
    ├── cloudwatch.tf
    ├── dynamodb.tf
    ├── dynamodbpolicy.json
    ├── lambda.tf
    ├── main.tf
    ├── outputs.tf
    ├── terraform.tfstate
    ├── terraform.tfstate.backup
    ├── terraform.tfvars
    └── variables.tf
```

### ./http-crud-application

Firstly lets take a look at `./http-crud-application`

This directory contains the source code for your lamdba function, check out `index.js` this is chock full of javascript goodness and I won't go into too much detail about it but this is where the magic happens.

**if** you want to adjust this file, zip it up to ensure it's deployment via the terraform code base:

```
zip http-crud-tutorial-function.zip index.js
updating: index.js (deflated 65%)
```

a few things to bare in mind here *[credit](https://bobbyhadz.com/blog/aws-lambda-cannot-find-module):

- Zipping the wrong files, e.g. zipping a directory instead of the contents of the directory. AWS lambda expects to extract the zip file and find your handler, not a directory with your handler in it.

- Having a wrong folder structure when using layers. There is a language specific folder structure you have to follow when using lambda layers, e.g. nodejs/node_modules for Node.js layers.

### ./terraform

This folder contains all that terraform code goodness that you'd expect.

```
.terraform.lock.hcl
apigateway.tf
cloudwatch.tf
dynamodb.tf
dynamodbpolicy.json
lambda.tf
main.tf
outputs.tf
terraform.tfstate
terraform.tfstate.backup
terraform.tfvars
variables.tf
```

**.terraform.lock.hcl**
This file is a dependency lock file for terraform more details [here](https://www.terraform.io/language/files/dependency-lock)

**apigateway.tf**
Contains the apigateway terraform code, incudling route creation via this means:

```
resource "aws_apigatewayv2_route" "lambda_delete_route" {
  api_id    = aws_apigatewayv2_api.lambdagateway.id
  route_key = "DELETE /items/{id}"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_get_integration.id}"
}
```

Any additional routes must resemble the above block with the new `route_key` for example:

```
resource "aws_apigatewayv2_route" "lambda_custom_route" {
  api_id    = aws_apigatewayv2_api.lambdagateway.id
  route_key = "DELETE /custom/{id}"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_get_integration.id}"
}
```

**cloudwatch.tf**

Creates the aws cloudwatch log group to store the logs from both the API and the Lambda function.

**dynamodb.tf**

Creates the dynamodb instance. 

**dynamodbpolicy.json**

The dynamo db IAM policy consumed by lambda.tf!

**lambda.tf**

Creates the lamdba function, this section in particular grabs the source code defined under `http-crud-tutorial-function`
```
resource "aws_lambda_function" "http-crud-tutorial-function" {
  function_name    = "http-crud-tutorial-function"
  filename         = "../http-crud-application/http-crud-tutorial-function.zip"
  role             = aws_iam_role.lambda_role.arn
  runtime = "nodejs16.x"
  handler = "index.handler"
  timeout          = 10
}
```

So if you plan to adjust the source and the location update `filename`.

Another thing to note here verify that the lambda function's handler is set to index.handler (an index.js file exporting a handler function), this one caught me out and I was stuck on `Cannot find module` for a good few hours!!!

**main.tf**

`main.tf` will contain the main set of configuration for your module

**outputs.tf**

Output values make information about your infrastructure available on the command line, and can expose information for other Terraform configurations to use.

**terraform.tfvars**

This sets up some rudimentary variables, tfvars files are the best and most common way to manage variables in Terraform.

**variables.tf**

Defines the variables we set above!!!

## Running the code

Navigate to the `./terraform` directory.

Perform a `terraform init`

Perform a `terraform plan`

```
Plan: 16 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + base_url = (known after apply)

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.
```

Review the plan then proceed with your `terraform apply`. If successful you'll see the below (or similar):

```
Apply complete! Resources: 9 added, 0 changed, 2 destroyed.

Outputs:

base_url = "https://abcdef123.execute-api.us-east-1.amazonaws.com/"
```

Notice you're provided with the base_url, you should now be able to run GET and PUT against this.

## Testing the API

**To create or update an item**

Use the following command to create or update an item. The command includes a request body with the item's ID, price, and name.

`curl -v -X "PUT" -H "Content-Type: application/json" -d "{\"id\": \"123\", \"price\": 12345, \"name\": \"myitem\"}" https://abcdef123.execute-api.us-east-1.amazonaws.com/items`

Response: `"Put item 123"%`

**To get all items**

Use the following command to list all items.

`curl -v https://abcdef123.execute-api.us-east-1.amazonaws.com/items`

Response: `{"Items":[{"price":12345,"id":"123","name":"myitem"}],"Count":1,"ScannedCount":1}%`

**To get an item**

Use the following command to get an item by its ID.

`curl -v https://abcdef123.execute-api.us-east-1.amazonaws.com/items/123`

Response: ``{"Item":{"price":12345,"id":"123","name":"myitem"}}%`

**To delete an item**

Use the following command to delete an item.

`curl -v -X "DELETE" https://abcdef123.execute-api.us-east-1.amazonaws.com/items/123`

Response: `"Deleted item 123"%`


Get all items to verify that the item was deleted.

`curl -v https://abcdef123.execute-api.us-east-1.amazonaws.com/items`