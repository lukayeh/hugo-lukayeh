---
title: "Creating a simple Python API using Serverless and AWS"
date: 2022-07-26
draft: false
tags:
 - "python"
 - "serverless"
 - "lambda"
 - "aws"
categories:
 - "techstuff"
---

In this tutorial we'll be walking through how to setup a simple serverless Python API using the serverless framework:

[serverless](https://www.serverless.com/)

![Serverless Logo](https://camo.githubusercontent.com/108c301af486eeb9afde6ec9d6c98aaa5b1b2c14becaab3b781c22b851687e9c/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6173736574732e6769746875622e7365727665726c6573732f726561646d652d7365727665726c6573732d6672616d65776f726b2e676966 "Serverless")

>The Serverless Framework – Build applications on AWS Lambda and other next-gen cloud services, that auto-scale and only charge you when they run. This lowers the total cost of running and operating your apps, enabling you to build more and manage less.

>The Serverless Framework is a command-line tool that uses easy and approachable YAML syntax to deploy both your code and cloud infrastructure needed to make tons of serverless application use-cases. It's a multi-language framework that supports Node.js, Typescript, Python, Go, Java, and more. It's also completely extensible via over 1,000 plugins that can add more serverless use-cases and workflows to the Framework.

The whole point of going serverless is you reduce your overhead on managing stateful Virtual Machines, using something like the Serverless framework you don't have to concern yourself with provisioning the server and auto-scaling to support performance.

This tutorial is going to give you a helping hand in getting started with the framework which you can hopefully then expand upon. 

## Prerequisites
For this tutorial, you will need:

- An AWS account.
- The AWS CLI (2.0+) installed, and configured for your AWS account.
- npm installed.
- serverless installed.

**Note** This tutorial is performed on a macbook air m1 so many steps are catered towards this.

## Getting Started

Firstly let's get npm installed `brew install npm`

Now let's install the serverless package: `npm install -g serverless`

You should be able to check the version installed:
```
serverless -v
Framework Core: 3.21.0
Plugin: 6.2.2
SDK: 4.3.2
```

## Creating the python api

So let's first create our `app.py` within our projects directory:
```python
from flask import Flask, jsonify, make_response

app = Flask(__name__)

@app.route("/")
def hello_from_root():
    return jsonify(message='Hello from root!')

@app.route("/hello")
def hello():
    return jsonify(message='Hello from path!')

@app.errorhandler(404)
def resource_not_found(e):
    return make_response(jsonify(error='Not found!'), 404)
```

Here we're just defining some simple routes for `/` and `/hello` with the intention of returning a json output!

As always we'll want to accompany this with a `requirements.txt` so create that next:
```python
Flask==1.1.4
Werkzeug==1.0.1
markupsafe==2.0.1
```

Create and activate a dedicated virtual environment with the following command:
```bash
python3 -m venv ./venv
source ./venv/bin/activate
```

Next install your `requirements.txt`
```
pip3 install -r requirements.txt
```

Run `flask run` from within the root directory:

```bash
flask run                  
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000 (Press CTRL+C to quit)
127.0.0.1 - - [05/Aug/2022 11:13:22] "GET / HTTP/1.1" 200 -
```

Open a secondary command line and curl the endpoints:

```bash
$ curl http://127.0.0.1:5000
{"message":"Hello from root!"}

$ curl http://127.0.0.1:5000/dev
{"error":"Not found!"}

$ curl http://127.0.0.1:5000/hello
{"message":"Hello from path!"}
```

So we have now validated it works locally (always good to know!)

Time to proceed to the next step :) 

## Getting started with Serverless framework

A service, aka a project, is the Framework's unit of organization.

A service is configured via a serverless.yml file where you define your functions, the events that trigger them, and the AWS resources to deploy.

For this particular application we're going to use the plugin `serverless-wsgi`

> WSGI is the Web Server Gateway Interface. It is a specification that describes how a web server communicates with web applications, and how web applications can be chained together to process one request.


Now it's time to create your `serverless.yml`. 

```yml
service: aws-python-flask-api # Name of the service!

frameworkVersion: '3' # Framework version

custom: 
  wsgi:
    app: app.app

provider:
  name: aws
  runtime: python3.9

functions:
  api:
    handler: wsgi_handler.handler
    events:
      - httpApi: '*'

plugins:
  - serverless-wsgi
  - serverless-python-requirements
```

I'll break down the blocks now, this one simply names the service!:

```yml
service: aws-python-flask-api # Name of the service!
```

To configure version pinning define a `frameworkVersion` property in your serverless.yml. Whenever you run a Serverless command from the CLI it checks if your current Serverless version is matching the frameworkVersion range. The CLI uses Semantic Versioning so you can pin it to an exact version or provide a range. In general we recommend to pin to an exact version to ensure everybody in your team has the exact same setup and no unexpected problems happen.

```yml
frameworkVersion: '3' # Framework version
```

Below block configures the AWS provider and the runtime to use for your application (this example uses python3.9)
```yml
provider:
  name: aws
  runtime: python3.9
```

This block defines the functions for the application:
```yml
functions:
  api:
    handler: wsgi_handler.handler
    events:
      - httpApi: '*'
```

And finally the below block defines the required plugins:
```yml
plugins:
  - serverless-wsgi
  - serverless-python-requirements
```
Serverless loads all the core plugins, and then the custom plugins in the order you've defined them.

Now before we get all carried away you'll see you've defined some plugins but not installed them so lets get them installed..

```bash
$ serverless plugin install -n serverless-wsgi
✔ Plugin "serverless-wsgi" installed  (1s)
```

```bash
$ serverless plugin install -n serverless-python-requirements
✔ Plugin "serverless-python-requirements" installed  (10s)
```

## Deploying the serverless application

Deploying your framework is as simple as running: `sls deploy`

You can also run: `serverless deploy` both will work

```bash
Running "serverless" from node_modules

Deploying aws-python-flask-api to stage dev (us-east-1)
Using Python specified in "runtime": python3.9
Packaging Python WSGI handler...

⠦ Creating CloudFormation stack (0/2) (20s)
```

It'll take some time now to deploy the stack (not too long hopefully...)

Once it's deployed you should see a output like below (mine took around 2.5 minutes)
```bash
Running "serverless" from node_modules

Deploying aws-python-flask-api to stage dev (us-east-1)
Using Python specified in "runtime": python3.9
Packaging Python WSGI handler...

✔ Service deployed to stack aws-python-flask-api-dev (132s)

endpoint: ANY - https://abcdef.execute-api.us-east-1.amazonaws.com
functions:
  api: aws-python-flask-api-dev-api (10 MB)

Improve API performance – monitor it with the Serverless Dashboard: run "serverless"
```

Now curl the endpoint provided:
```bash
curl https://abcdef.execute-api.us-east-1.amazonaws.com
{"message":"Hello from root!"}

curl https://abcdef.execute-api.us-east-1.amazonaws.com/hello
{"message":"Hello from path!"}
```

Fantastic it works!!!

If you check in AWS you should see that it's created as expected:

![](https://i.imgur.com/zeEjYMW.png)


## Adjusting your api..

Now let's say we want to add another route to the project, let's do that. Update `app.py` to appear as below:
```python
from flask import Flask, jsonify, make_response

app = Flask(__name__)

@app.route("/")
def hello_from_root():
    return jsonify(message='Hello from root!')

@app.route("/hello")
def hello():
    return jsonify(message='Hello from path!')

@app.route("/goodbye")
def goodbye():
    return jsonify(message='Goodbye from path!')

@app.errorhandler(404)
def resource_not_found(e):
    return make_response(jsonify(error='Not found!'), 404)
```

Run `sls deploy` again

You should see it deploys successfully, now time to curl the new endpoint:
```bash
$ curl https://abcdef.execute-api.us-east-1.amazonaws.com/goodbye
{"message":"Goodbye from path!"}
```

## Cleaning it up

Serverless provides a really simple way to cleanup your resources all you need to do is run `serverless remove`

```
serverless remove         

Running "serverless" from node_modules
Removing aws-python-flask-api from stage dev (us-east-1)

✔ Service aws-python-flask-api has been successfully removed (31s)
```
 
## Conclusion

Now you can make adjusts as you see fit and play around some more there's plenty more tutorials out there to further your knowledge on serverless.

Source code is availabe here: https://github.com/lukayeh/serverless-api-202208051050
















