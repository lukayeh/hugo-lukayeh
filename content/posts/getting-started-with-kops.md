---
title: "Getting started with Kops in AWS"
date: 2022-11-06
draft: false
tags:
 - "kubernetes"
 - "kops"
 - "tech"
categories:
 - "techstuff"
---

This post will walk you through building a kubernetes cluster in AWS using the tooling `kOps`. 

kOps is the easiest way to get a production grade Kubernetes cluster up and running. We like to think of it as kubectl for clusters.

kOps helps you create, destroy, upgrade and maintain production-grade, highly available, Kubernetes clusters from the command line. AWS (Amazon Web Services) is currently officially supported, with Digital Ocean and OpenStack in beta support.

We'll also then quickly throw a service in there and expose it so you can see it end to end!

## Step 1: Prerequisites

All prerequisites are supplied by brew, please see other installation guides if you're using something outside of Macos.

Install kubectl, Kubectl is the command line tool that enables you to execute commands against your Kubernetes cluster. 
```console
$ brew update; brew install kubectl
```

Install Kops, Kops is an official tool for managing production-grade Kubernetes clusters on AWS. 
```console
$ brew install kops
```

Validate installation of kops by checking the version number:
```console
$ kops version                 
Client version: 1.25.2
```

Install the AWS Cli, this will allow you to interact with AWS resources via a python distributed package:
```console
brew install awscli
```

Once installed, you'll need to configure awscli to work with your account, run `aws configure` to do so.

## Step 2: Configure Route53 DNS zones

DNS is a prerequsite for a functioning Kops cluster, using DNS is far easier than trying to remember a set of IP addresses and allows you to easily scale your clusters.

For this tutorial we're going to use Route 53 Hosted Zones utitlising the `dev.$DOMAIN` setup. 

>A hosted zone is a container for records, and records contain information about how you want to route traffic for a specific domain, such as example.com, and its subdomains (acme.example.com, zenith.example.com). A hosted zone and the corresponding domain have the same name. There are two types of hosted zones:
> * Public hosted zones contain records that specify how you want to route traffic on the internet. For more information, see Working with public hosted zones.
> * Private hosted zones contain records that specify how you want to route traffic in an Amazon VPC. For more information, see Working with private hosted zones.

Use the below command to setup a hosted-zone. 

```console
$ DOMAIN=lukayeh.com
$ aws route53 create-hosted-zone --name dev.$DOMAIN --caller-reference 1
```

Validate this has been created by running the below:
```json
$ aws route53 list-hosted-zones | jq -r                                                        
{
  "HostedZones": [
    {
      "Id": "/hostedzone/REDACTED",
      "Name": "dev.lukayeh.com.",
      "CallerReference": "1",
      "Config": {
        "PrivateZone": false
      },
      "ResourceRecordSetCount": 2
    }
  ]
}
```

Here dev is the subdomain,  and `lukayeh.com` is my parent domain. In order to route requests to your subdomain you'll need to update your parent domain with a NS record with the the values from your subdomain, I won't go into detail on how to do that but this article [here](https://medium.com/@deep_blue_day/how-to-create-a-subdomain-in-amazon-route-53-81918654f5bf) will get you there!

Try running `dig NS dev.yourdomain.com`. If it responds with 4 NS records that point to your Hosted Zone, then everything is working properly.

## Step 3: Create S3 Buckets for Kube configuration

This next step will be creating the S3 buckets to store your clusters configuration, kops will use this storage to persist configuration, think state with terraform!

```console
$ aws s3 mb s3://storage.dev.$DOMAIN
make_bucket: storage.dev.lukayeh.com
```

Kops needs a environment variable setting to allow utilisation of this S3 storage, run the below do to this:

```console
$ export KOPS_STATE_STORE=s3://storage.dev.$DOMAIN
```

Validate with:
```console
$ env | grep KOPS_STATE_STORE                                         
KOPS_STATE_STORE=s3://storage.dev.lukayeh.com
```

You _could_ export the variable in your console_profile if you really want to but for the sake of this example I won't bother going into detail there.

## Step 4: Building your cluster

So far you've created what are mostly prerequsites and now it's time to put all that hard work to good use and standup your first kops cluster :tada:

With your s3 congigured earlier in your enviroment variables, you can run the below:

```console
$ kops create cluster --zones=us-east-1c dev.$DOMAIN
```

Output will kick off but you will eventually see something along the lines of:
```console
 * list clusters with: kops get cluster
 * edit this cluster with: kops edit cluster dev.lukayeh.com
 * edit your node instance group: kops edit ig --name=dev.lukayeh.com nodes-us-east-1c
 * edit your master instance group: kops edit ig --name=dev.lukayeh.com master-us-east-1c

Finally configure your cluster with: kops update cluster --name dev.lukayeh.com --yes --admin
```

At this point your cluster has not yet been stood up, infact this was very much a dry-run setting up your clusters configuration.

Now it's time to build the cluster:
```console
$ kops update cluster --name dev.lukayeh.com --yes --admin
```

After a minute or so, your cluster will be ready, you'll see this information here which reflects that its currently standing up:
```
Cluster is starting.  It should be ready in a few minutes.

Suggestions:
 * validate cluster: kops validate cluster --wait 10m
 * list nodes: kubectl get nodes --show-labels
 * ssh to the master: ssh -i ~/.ssh/id_rsa ubuntu@api.dev.lukayeh.com
 * the ubuntu user is specific to Ubuntu. If not using Ubuntu please use the appropriate user based on your OS.
 * read about installing addons at: https://kops.sigs.k8s.io/addons.
 ``` 

If you validate immediately you'll see something like:
```
W1106 18:58:41.902643   29864 validate_cluster.go:232] (will retry): cluster not yet healthy
```

Which just means you'll need a wait a moment for the cluster to be stood up, patience youngling!

After probably around 10 minutes (enough time to make a brew), you'll get the following:
```
Your cluster dev.lukayeh.com is ready
```

### Step 4.1: Download kOps config spec file

Kops operates off of a config spec file that is generated during the create phase. It is uploaded to the amazon s3 bucket that is passed in during create.

If you download the config spec file on a running cluster that is configured the way you like it, you can just pass that config spec file in to the create command and have kOps create the cluster for you, `kops create -f file.yaml` in a completely unattended manner.

To get the yaml configuration of your cluster you can run:

```console
$ kops get -o yaml > kops.yaml
```

This will export the entire cluster for future usage!

## Step 5: Connecting kubectl to your cluster

Okay so now your cluster has been stood up time to connect to it using kubectl:
```console
$ kops export kubecfg dev.$DOMAIN
```

Kubectl should now be pointing at your cluster!

Run a `kubectl get nodes` command and you'll see your nodes:
```console                                                                    
NAME                  STATUS   ROLES           AGE    VERSION
i-0898946028e62efb3   Ready    control-plane   3m9s   v1.25.3
i-09840c85222583f7b   Ready    node            119s   v1.25.3
```

## Step 6: Deploy a sample application

Okay so just for funsies let's deploy a simple hello-world application, first create the below yaml file:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: app
          image: brightbox/rails-hello-world
          ports:
            - name: web
              containerPort: 3000
              protocol: TCP
```

Create the namespace `example`:
```console
$ kubectl create namespace example
namespace/example created
```

Now create the deployment:
```console
$ kubectl create -f deployment.yaml 
```

Wait a few moments and monitor the deployment:
```console
$ kubectl get deployments -n example -w
```

Check the pods exist:
```console
$ kubectl -n example get pods     
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-69f8bb9dc4-rt82c   1/1     Running   0          43s
```

Now create the service yaml file:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: hello-world
  namespace: example
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "KubernetesCluster=dev.lukayeh.com"
spec:
  type: LoadBalancer
  selector:
    app: hello-world
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 3000
```

and create it: 
```console
$ kubectl create -f service.yaml
service/hello-world created
```

Check the status:
`kubectl describe svc hello-world -n example`

**Note** You may hit a error that I did that was something like:
```
An error occurred fetching load balancer data: User: arn:aws:iam::000000000000:user/xxxxxxxx is not authorized to perform: elasticloadbalancing:DescribeLoadBalancers
```
if you do I had to modify the IAM policy for role `masters.dev.lukayeh.com` to add `"elasticloadbalancing:Describe*"` I believe this is a bug! :shrug:

Monitor the service with `kubectl get svc -n example -w`

Once your service is up you should get a external IP:
```console
$ kubectl get service/hello-world -n example        
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)        AGE
hello-world   LoadBalancer   100.64.175.77   redactedstring-1190367269.us-east-1.elb.amazonaws.com   80:32322/TCP   12m
```

Now hit your url the below command should get you there:
```console
$ curl http://`kubectl get service/hello-world -n example --output jsonpath='{.status.loadBalancer.ingress[0].hostname}'`
```

Output should be something like:
```html
<!DOCTYPE html>
<html>
  <head>
    <title>HelloWorld</title>

    <link rel="stylesheet" media="all" href="/assets/application-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855.css" data-turbolinks-track="reload" />
    <script src="/assets/application-19f7aded4e7e773e7aaded6553cd10f08d882d38782ae6c1ab7bd9d20d8bd39b.js" data-turbolinks-track="reload"></script>
  </head>

  <body>
    <h1>Hello World!</h1>
<p>Ruby version is 2.5.1</p>
<p>The current time is 2022-11-06 19:57:55 +0000</p>

  </body>
</html>
```

That's it for now.

## Step 7: Cleanup!

Let's clean it all up:
```console
$ kubectl delete svc hello-world -n example
$ kubectl delete deployment hello-world -n example
$ kops delete cluster --name dev.lukayeh.com --yes
```

That should do it!

## Conclusion

So in conclusion hopefully you've gone and stood up a kops cluster using the tooling provided within AWS, deployed a demo application and exposed it using a ELB now this is simple stuff so you can imagine how this expands at a production level!
























