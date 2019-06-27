# Horizontal Pod Autoscaler with Nodeless Kubernetes

### Step 1: Create 1-worker Nodeless Kubernetes cluster

[Follow instructions in this repo](https://github.com/elotl/kubeadm-aws) to create a {1 master, 1 Milpa worker} Nodeless Kubernetes cluster.

Grab k8s master and Milpa worker IPs from terraform output:
```
Outputs:

master_ip = 3.86.247.178
milpa_worker_ip = 34.203.206.23
```

### Step 2: Deploy Kubernetes Metrics Server

Log on to Kubernetes master, then deploy Kubernetes Metrics Server:

```
$ ssh -i myechuri-key2.pem ubuntu@3.86.247.178
ubuntu@ip-10-0-100-234:~$
```


```
git clone https://github.com/mattkelly/metrics-server.git
cd metrics-server
git checkout bugfix/deploy-1.8+
kubectl apply -f deploy/1.8+/
```

### Step 3: Run [resource-consumer](https://github.com/kubernetes/kubernetes/tree/master/test/images/resource-consumer) to generate load

```
kubectl run resource-consumer --image=gcr.io/kubernetes-e2e-test-images/resource-consumer:1.4 --expose --service-overrides='{ "spec": { "type": "LoadBalancer" } }' --port 8080 --requests='cpu=100m'
```

Wait for `resource-consumer` to be in `Running` state.
```
$ kubectl get pods

NAME                                 READY   STATUS    RESTARTS   AGE
resource-consumer-6758b8d5dd-5wvdc   1/1     Running   0          78s
```

Grab external-ip of its load balancer.
```
export RESOURCE_CONSUMER_ADDRESS="http://$(kubectl get service resource-consumer -ojsonpath='{.status.loadBalancer.ingress[0].hostname}'):8080"
```

```
$ echo $RESOURCE_CONSUMER_ADDRESS
http://a446b4b66fd724db8823aebbaac29945-1022133427.us-east-1.elb.amazonaws.com:8080
```

### Step 4: Create HPA for `resource-consumer`

```
kubectl autoscale deployment resource-consumer --cpu-percent=50 --min=1 --max=10
```

Wait for percentage utilization to transition from `unknown` to a number (takes upto 20 seconds).
```
ubuntu@ip-10-0-100-234:~/metrics-server$ kubectl get hpa
NAME                REFERENCE                      TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
resource-consumer   Deployment/resource-consumer   <unknown>/50%   1         10        0          4s

ubuntu@ip-10-0-100-234:~/metrics-server$ kubectl get hpa
NAME                REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
resource-consumer   Deployment/resource-consumer   1%/50%    1         10        1          26s
```

### Step 5: Generate load over 50% to trigger horizontal pod autoscaling

Please note that we skipping setup/configure/management of Cluster Autoscaler!

```
curl --data "millicores=60&durationSec=600" $RESOURCE_CONSUMER_ADDRESS/ConsumeCPU
```

Watch deployments, replicasets, pods, hpa.

```
watch --differences kubectl get deployments,replicasets,pods,hpa
```

```
Every 2.0s: kubectl get deployments,replicasets,pods,hpa                                                Thu Jun 27 21:40:10 2019

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/resource-consumer   2/2     2            2           11m

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.extensions/resource-consumer-6758b8d5dd   2         2         2       11m

NAME                                     READY   STATUS    RESTARTS   AGE
pod/resource-consumer-6758b8d5dd-5wvdc   1/1     Running   0          11m
pod/resource-consumer-6758b8d5dd-dj9ls   1/1     Running   0          53s

NAME                                                    REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS
AGE
horizontalpodautoscaler.autoscaling/resource-consumer   Deployment/resource-consumer   61%/50%   1         10        2
8m12s

```

Notice the second pod `resource-consumer-6758b8d5dd-dj9ls` triggered by HPA in response to cpu consumption going over 50%.

### Acknowledgements

This tutorial is built on top of [Matt Kelley's tutorial](https://blog.containership.io/cerebral-vs-kubernetes-cluster-autoscaler/). We are grateful to his work in identifying and fixing bugs in Metrics Server.
