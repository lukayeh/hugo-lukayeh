---
title: "Using Horizontal Pod autoscaling in Kubernetes to horizontally scale your app"
date: 2022-07-12
draft: false
tags:
 - "kubernetes"
 - "tech"
categories:
 - "techstuff"
---

<img src="https://www.kubecost.com/images/hpa-overview.png" width="500" height="350" />

Hello there! This blog post will detail how to use `HorizontalPodAutoscaler` to horizontally scale a pod within kubernetes, for this demonstration I am going to be using minikube installed on my local macbook, therefore as a prerequisite please ensure you have docker and minikube setup and functioning as expected (I may come back to how to do this in a future blog post but otherwise google is your friend!)

## Preparation 

Firstly make sure you have minikube installed:

```
 minikube version
minikube version: v1.25.2
commit: 362d5fdc0a3dbee389b3d3f1034e8023e72bd3a7
```

Start your minikube instance:

 ```
 minikube start
😄  minikube v1.25.2 on Darwin 12.4 (arm64)
✨  Automatically selected the docker driver
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
🐳  Preparing Kubernetes v1.23.3 on Docker 20.10.12 ...
    ▪ kubelet.housekeeping-interval=5m
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

Check kubectl is working properly:
```
kubectl get pods
No resources found in default namespace.

kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
minikube   Ready    control-plane,master   40s   v1.23.3
```

Enable metrics server for minikube:
```
minikube addons enable metrics-server

    ▪ Using image k8s.gcr.io/metrics-server/metrics-server:v0.4.2
🌟  The 'metrics-server' addon is enabled
```

Details on how to do this outside of minikube can be found here: https://github.com/kubernetes-sigs/metrics-server#readme



**NOTE**
If like me you have difficulties accessing the minikube instance on port 80 or any port for that matter, it may be you're using a m1 device, I had to start minikube with this:

`minikube start --driver=docker --ports=127.0.0.1:30080:30080 --disable-optimizations`

*FYI I had to use --disable-optimizations due to this: https://github.com/kubernetes/minikube/issues/13898*

Check the ports have been forwarded here:

```
docker port minikube
22/tcp -> 127.0.0.1:50820
2376/tcp -> 127.0.0.1:50821
30080/tcp -> 127.0.0.1:30080
32443/tcp -> 127.0.0.1:50817
5000/tcp -> 127.0.0.1:50818
8443/tcp -> 127.0.0.1:50819
```

## Run and expose the php container 

First lets create the directory to hold our terraform code with a `mkdir`
`mkdir kubeautoscale-$(date +"%Y%m%d%H%M")`

Create the below file within your directory `php-apache.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 100Mi
            cpu: 100m
          requests:
            memory: 100Mi
            cpu: 100m

---

apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  type: NodePort
  selector:
    run: php-apache
  ports:
  - protocol: TCP
    port: 80
    nodePort: 30080

```

Okay that looks good to me how's it looking to you? Good? Great let's apply this sucker.

```
kubectl apply -f php-apache.yaml

deployment.apps/php-apache created
service/php-apache created
```

Give it a moment it's probably still creating...you can check with...

`kubectl get pod php-apache-7656945b6b-trg6r -w`

`-w` means watch, so you know it watches!

Eventually you should see your pod alive and well:

```
NAME                          READY   STATUS    RESTARTS   AGE
php-apache-7656945b6b-trg6r   1/1     Running   0          62s
```

Check the deployment:

```
kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   1/1     1            1           105s
```

Check the service:
```
kubectl get services
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        2m19s
php-apache   NodePort    10.101.86.82   <none>        80:30080/TCP   75s
```

So far so good, lets curl the service make sure its responding:

```
curl http://127.0.0.1:30080
OK!%
```

Awesome it's responding!!!

## Scaling time

Now it's time to scale like spiderman up the side of a building in New York when fighting..you know what just tell me to stop!

We can create the autoscaler using kubectl. There is kubectl autoscale subcommand, part of kubectl which we're going to utilise here!

Our intention is to create a HorizontalPodAutoscaler that maintains between 1 and 10 replicas of the Pods controlled by the php-apache Deployment that we created in the first step of these instructions.

HPA controller will increase and decrease the number of replicas (by updating the Deployment) to maintain an average CPU utilization across all Pods of 50%. The Deployment then updates the ReplicaSet - this is part of how all Deployments work in Kubernetes - and then the ReplicaSet either adds or removes Pods based on the change to its .spec.

The below command takes care of the autoscale deployment:

`kubectl autoscale deployment php-apache --cpu-percent=20 --min=1 --max=10`

Lets check the status of it:

```
kubectl get hpa
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   <unknown>/20%   1         10        0          14s
```

If you see `<unknown>` you may want to wait 5 minutes when I did I got:

```
 kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/20%          1         10        1          2m45s
```

You can also check the issue if it doesn't resolve by doing `kubectl describe hpa` and getting your google on!!

### Time to increase the load

Time to give this pod a heart attack! 

**Note**
Run this next part in a seperate terminal!

Open a new terminal window and run the below, what we're doing here is creating a different Pod to act as a client to our php-apache pod. The container within the client Pod runs in an infinite loop, sending queries to the php-apache service.

```
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

You should see the output now:
```
If you don't see a command prompt, try pressing enter.
OK!OK!OK!OK!
```

Its going to start hitting it hard and fast :sunglasses:

In your other console run: `kubectl get hpa php-apache --watch`

You should start to see the CPU ramp up

You can check with:
```
kubectl top pod    
NAME                         CPU(cores)   MEMORY(bytes)
load-generator-2             0m           0Mi
php-apache-ff74b65f6-7ng2k   0m           3Mi
```

Within <3 minutes I was at 29% already:
```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   29%/20%   1         10        1          3m38s
```

1 minute later you'll see the replicas are growing:
```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   26%/20%   1         10        8          5m44s
```

As a result, the Deployment was resized to 8 replicas:

```
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   8/8     8            8           7m33s
```

Note: It may take a few minutes to stabilize the number of replicas. Since the amount of load is not controlled in any way it may happen that the final number of replicas will differ from this example.

### Time to reduce the load

In the terminal where your busybox container is running (the one hammering php) crtl+c the console:

Wait a few minutes (I waited 10 minutes) for the load to drop:

```
kubectl get hpa php-apache --watch             ✔  at minikube ⎈  at 14:58:10  

NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   1%/20%    1         10        1          15m
```

Notice replicas is back down to `1`

```
kubectl get deployment php-apache

NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   1/1     1            1           18m
```

One final thing I found that is pretty cool is that you can export your HPA configuration into YAML to commit to source:

`kubectl get hpa php-apache -o yaml > hpa.yaml`

(Note I haven't used the above yet so if that doesn't work then pfft!)


## Conclusion

So this post ran you through some of the basics of running a scalable php you can take this further and use other metrics to scale see [here](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics
)
## Credits

Big shout out to the below articles for guiding me through this one:
* https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
* https://www.techtarget.com/searchitoperations/tutorial/Kubernetes-performance-testing-tutorial-Load-test-a-cluster











