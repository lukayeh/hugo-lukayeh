---
title: "Getting Started with Terragrunt"
date: 2022-04-24
draft: false
tags:
 - "terraform"
 - "tech"
 - "terragrunt"
categories:
 - "techstuff"
---

# Introduction

This blog post is going to cover my initial journey into Terragrunt and deploying a simple S3 Bucket to AWS as per my previous examples as this is the simplest way to play with Terraform/grunt/whatever 


# Lets DO IT

## Prerequisites

As this tutorial utilises AWS make sure you have setup a AWS account and ran `aws configure` with the correct credentials, this page is helpful for this kinda thing:

https://www.cyberciti.biz/faq/osx-installing-the-aws-command-line-interface-using-brew/ 

### Install Terragrunt

You can install Terragrunt on macOS using Homebrew: 

`brew install terragrunt`

## Setup some terraform code

Firstly lets get some standard terraform code setup to create a S3 bucket.

Create the below file:

**main.tf**
```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "b" {
  bucket = "my-tf-test-bucket"

  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}
```

Perform a `terraform init`, `terraform plan`

Your output should suggest its going to create 1 item which is your bucket!

`Plan: 1 to add, 0 to change, 0 to destroy.`

Okay cool we know now this code is going to function as we expect, feel free to `apply` and `destroy`  however I'm assuming as you're reading this I assume you're well aware of the basics for terraform (otherwise get your google hat on youngling)

Inside your folder create `terragrunt.hcl`

`touch terragrunt.hcl`

Create a directory for your environment:

`mkdir -p environments/dev`

Then create your src directory where your main.tf and providers will live:

```bash
mkdir src
touch src/main.tf
touch src/provider.tf
```

Inside `srv/provider.tf` add the following:
```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}
```

Notice this isn't too disimilar to ouroriginal main.tf.

In the same vain take your resource for the s3 bucket and move it to `srv/main.tf`

```terraform
resource "aws_s3_bucket" "b" {
  bucket = "my-tf-test-bucket"

  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}
```

In `environments/dev`, create a file named `terragrunt.hcl` with the below content:

**environments/dev/terragrunt.hcl**

```terraform
include {
    path = find_in_parent_folders()
}
```

This smart little bit of code instructs Terragrunt to use any .hcl configuration files it finds in parent folders.

Add a terraform block to give Terragrunt a source reference:
```terraform
include {
    path = find_in_parent_folders()
}

terraform {
    source = "../../src"
}
```

Now change to your environments/dev folder and run: `terragrunt init`

This sets up a bit of context, including environment variables, then runs terraform init.

run `terragrunt plan` and you'll see your s3 bucket is going to get created:
```terraform
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.b will be created
  + resource "aws_s3_bucket" "b" {
      + acceleration_status         = (known after apply)
.
.
.
Plan: 1 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────

```

In the root terragrunt.hcl let's add a variable:
**terragrunt.hcl**
```terraform
inputs = {
    env_name = "develop"
}
```

adjust the `srv/main.tf` file to add this variable:

**srv/main.tf**
```terraform
variable "env_name" {}

resource "aws_s3_bucket" "b" {
  bucket = "my-tf-${var.env_name}-bucket"

  tags = {
    Name        = "My ${var.env_name} bucket"
    Environment = "${var.env_name}"
  }
}
```

Now run your `terragrunt plan` again and you'll see it would create a bucket named `my-tf-develop-bucket`

```terraform
 # aws_s3_bucket.b will be created
  + resource "aws_s3_bucket" "b" {
      + acceleration_status         = (known after apply)
      + acl                         = "private"
      + arn                         = (known after apply)
      + bucket                      = "my-tf-develop-bucket"
```

That's cool we inherit a variable from the root directory but wouldn't it make more sense to have it by more dynamic!

In the root terragrunt.hcl file, update the input to use path_relative_to_include(), and pass the value as the env_name variable:
```terraform
inputs = {
    env_name = path_relative_to_include()
}
```

Running `terragrunt plan` again you can see its pulling in the folder names! How freaking cool is this:
```
 # aws_s3_bucket.b will be created
  + resource "aws_s3_bucket" "b" {
      + acceleration_status         = (known after apply)
      + acl                         = "private"
      + arn                         = (known after apply)
      + bucket                      = "my-tf-environments/dev-bucket"
```
BUT its really not that nice to keep the "environments" part, in fact I hate it lets get rid of that bad boy!

Update the terragrunt.hcl to strip  "environements/" from env_name:
```terraform
locals {
    env_name = replace(path_relative_to_include(), "environments/", "")
}

inputs = {
    env_name = local.env_name
}
```

What the heck did I just do you ask? you added a locals block to create a local variable and used the built-in replace function to remove the unwanted parts of the relative path. Then, you updated the inputs block to use the local variable, makes total sense right?!

Running `terragrunt plan` again...
```terraform
  # aws_s3_bucket.b will be created
  + resource "aws_s3_bucket" "b" {
      + acceleration_status         = (known after apply)
      + acl                         = "private"
      + arn                         = (known after apply)
      + bucket                      = "my-tf-dev-bucket"
```

Now we're sucking diesel !

It's time!

`terragrunt apply`

and you'll see its successfully created your bucket which you should also be able to view via the S3 AWS console:

![(https://imgur.com/a/qL9xB6N)](https://i.imgur.com/3okSmcd.png)

## Move your state to remote storage

Okay that's cool we have now done some terragrunting and was it worth it? Not sure yet but lets also setup our remote state!

Since Terragrunt allows you to configure multiple environments, you should store state files in their own S3 buckets so they don't overwrite each other.

In your root `terragrunt.hcl`, add a remote_state block that tells Terragrunt where to place your file in S3:
```terraform
remote_state {
    backend = "s3"
    generate = {
        path = "backend.tf"
        if_exists = "overwrite_terragrunt"
    }
    config = {
        bucket = "lukayeh-terraform" # Amazon S3 bucket required

        key     = "envs/${local.env_name}/terraform.tfstate"
        region  = "eu-west-1"
        encrypt = true
        profile = "default" # Profile name required
    }
}
```

run `terragrunt plan` you might be asked `Do you want to copy existing state to the new backend?` to which I answered yes!

Once its completed, you should see in your s3 state bucket you have a folder called `envs/` with a folder called `dev` thats NEAT!

## Create a new environment

Now that you've configured your development environment, create another that reuses most of your work.

Under environments, create a folder named `test`. In it, create a file called terragrunt.hcl:

`mkdir test && cd test`

`vi terragrunt.hcl`
```terraform
include {
    path = find_in_parent_folders()
}

terraform {
    source = "../../src"
}
```
Now notice run a `terragrunt plan` and the name of the bucket has magically changed!

```terraform

  # aws_s3_bucket.b will be created
  + resource "aws_s3_bucket" "b" {
      + acceleration_status         = (known after apply)
      + acl                         = "private"
      + arn                         = (known after apply)
      + bucket                      = "my-tf-test-bucket" << THIS OMG
```

Now, you've created two environments, dev and test, but they're the same, other than their name.

In src/main.tf, add new a variable for us to play with:

```terraform
variable "env_name" {}

variable "label" {default = "hairy_bikers"}

resource "aws_s3_bucket" "b" {
  bucket = "my-tf-${var.env_name}-bucket"

  tags = {
    Name        = "My ${var.env_name} bucket"
    Environment = "${var.env_name}"
    Labels      = "${var.label}"
  }
}
```

Running `terragrunt plan` again you'll see the labels are being set from the src:
```terraform
      + tags                        = {
          + "Environment" = "test"
          + "Labels"      = "hairy_bikers"
          + "Name"        = "My test bucket"
        }
```

In test/terragrunt.hcl, add values for your variable:

```terraform
include {
    path = find_in_parent_folders()
}

terraform {
    source = "../../src"
}

inputs = {
    labels = "I'm overriding your stuff from test" 
}
```

Another `plan` and you'll see the tags have now changed:

```bash
      + tags                        = {
          + "Environment" = "test"
          + "Labels"      = "I'm overriding your stuff from test"
          + "Name"        = "My test bucket"
        }
```

So now you have terragrunt code that functions very similarly to the workspaces tutorial I just ran but I can imagine you can see the similarities and I'll let you decide which you prefer.

Code for this can be found [here](https://github.com/lukayeh/terragrunt-202204241935).

![](https://c.tenor.com/Riuk0A0Mn-AAAAAC/its-over-ferris-bueller.gif)

### Acknowledgements 

Huge credit to this post for helping me on this journey https://developer.newrelic.com/terraform/terragrunt-configuration/ 

