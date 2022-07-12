---
title: "Using Terraform Workspaces to manage multi-env deployments"
date: 2022-04-14
draft: false
tags:
 - "terraform"
 - "tech"
categories:
 - "techstuff"
---

Hi! This blog post will cover working with Terraform Workspaces that allow you to manage multiple enviroments under the same code, this is one of many ways to do this with terraform (see using `--var-file=development.tfvars`) and I was interested in figuring out how this would work, I can't say it's the right way but I can say it's fairly straight forward however I have yet to use this at a large scale, so lets have a plan!

## Preparation 

Firstly make sure you have terraform installed, a correctly configured AWS CLI (so if you've done a few AWS Terraform tutorials you should be in a good state.

Check terraform is installed: `terraform -v`
```terraform -v
Terraform v1.0.11
on darwin_amd64
```

## Lets do this

First lets create the directory to hold our terraform code with a `mkdir`
`mkdir terraform-workspaces-blog`

We want to create two workspaces, you can list your workspaces with this command:
`terraform workspace list`
You'll see you have one currently setup:
```
terraform workspace list
* default
````
Lets create two more:
`terraform workspace new development`
and
`terraform workspace new production`
Now your list will look like this:
```  
default
development
* production
 ```
You'll notice you've been automatically switched to the most recently created that's fine we can move around these workspaces with `terraform workspace select <name>`

Terraform has some good documentation around workspaces here: https://www.terraform.io/language/state/workspaces#using-workspaces

Now lets try and get some code into this folder we created earlier, start a main.tf file and populate it with the below:
```
locals  {
	workspace_path = "./workspaces/${terraform.workspace}.yaml"
	defaults = file("${path.module}/config.yaml")
	workspace = fileexists(local.workspace_path) ? file(local.workspace_path) : yamlencode({})
	settings = merge(
		yamldecode(local.defaults),
		yamldecode(local.workspace)
	)
}

output  "project_settings"  {
	value = local.settings
}
```
The “locals” block will load a default file and a workspace file (if it exists) and then merge them together. This exposes `local.settings` to be used in the rest of the terraform module.

As you can see I haven't created the workspace files yet so lets do that now!
`mkdir workspaces`
Below commands should set up your files:
```
cat <<EOT >> workspaces/development.yaml
---
dcid: dev
instance_size: t2.xlarge
EOT
```

```
cat <<EOT >> workspaces/production.yaml
---
dcid: prod
instance_size: t2.xlarge
EOT
```

Setup the default values for when you haven't setup a workspace:
```
cat <<EOT >> config.yaml
---
dcid: default
instance_size: t2.medium
EOT
```
Now we are at this stage we can run the code and see what the output is:
```
 terraform plan

Changes to Outputs:
  + project_settings = {
      + dcid          = "prod"
      + instance_size = "t2.xlarge"
    }
```
Because my workspace is set to production I get that output, now if i switch to development using `terraform workspace select development` you can see my output changes to name the dcid as "dev"
```
Changes to Outputs:
  + project_settings = {
      + dcid          = "dev"
      + instance_size = "t2.xlarge"
    }
```
Pretty cool huh!
So right now you're folder structure should look like this:
```
.
├── config.yaml
├── main.tf
├── terraform.tfstate.d
│   ├── development
│   └── production
└── workspaces
    ├── development.yaml
    └── production.yaml
```
which  is great, now lets make this abit more advanced...
### Taking it a step further
So in the previous section we setup some basic terraform workspace code to give us a neat output, now we want to make this do something within AWS because that's where all the cool kids are these days.
so lets modify our main.tf and add these lines to the top:
```
terraform  {
	required_version = "~> 1"
	required_providers {
		aws = {
			source = "hashicorp/aws"
		}
	}
}

provider  "aws"  {
	region = "eu-west-1"
}
```
This basically says use version 1 or greater and says to install the aws provider.

Above the output block I've added this:
```
resource  "aws_s3_bucket"  "bucket"  {

	bucket = "my-tf-${local.settings.dcid}-bucket"

	tags = {
		Name = "My ${local.settings.dcid} bucket"
		Environment = local.settings.dcid
	}
}
```
This will use the local.settings defined earlier and grab the dcid from your workspace! as you can imagine you can get pretty creative with this by creating all sorts of configuration variances.

Run a `terraform init` 

Now run a `terraform plan` 

You should see a s3 bucket is going to get created:
```
  # aws_s3_bucket.bucket will be created
  + resource "aws_s3_bucket" "bucket" {
.
.
.
Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + project_settings = {
      + dcid          = "dev"
      + instance_size = "t2.xlarge"
    }

───────────────────
```
With the name `       + bucket                      = "my-tf-dev-bucket"`

As per below:
```
terraform plan | grep -A5 'resource "aws_s3_bucket" "bucket" {'
  + resource "aws_s3_bucket" "bucket" {
      + acceleration_status         = (known after apply)
      + acl                         = (known after apply)
      + arn                         = (known after apply)
      + bucket                      = "my-tf-dev-bucket"
      + bucket_domain_name          = (known after apply)
 ```
 Now switch workspaces again, `terraform workspace select production`
 Run your plan again:
 ```
   + resource "aws_s3_bucket" "bucket" {
      + acceleration_status         = (known after apply)
      + acl                         = (known after apply)
      + arn                         = (known after apply)
      + bucket                      = "my-tf-prod-bucket"
      + bucket_domain_name          = (known after apply)
 ```
 The bucket name has changed!!! Hazzar!!


 For your reference your main.tf should now look like this
```
terraform {
  required_version = "~> 1"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}


locals {
    workspace_path = "./workspaces/${terraform.workspace}.yaml" 
    defaults       = file("${path.module}/config.yaml")
    workspace = fileexists(local.workspace_path) ? file(local.workspace_path) : yamlencode({})
    settings = merge(
        yamldecode(local.defaults),
        yamldecode(local.workspace)
    )
}

resource "aws_s3_bucket" "bucket" {
  bucket = "my-tf-${local.settings.dcid}-bucket"

  tags = {
    Name        = "My ${local.settings.dcid} bucket"
    Environment = local.settings.dcid
  }
}

output "project_settings" {
    value = local.settings
}
 ```

That's as far as I'm taking this tutorial for now but as you can see its got alot of potential particular if you start to look at this from a configure multiple instances perspective.