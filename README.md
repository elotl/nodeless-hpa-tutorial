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

Log on to Kubernetes master, verify cluster is up, then deploy Kubernetes Metrics Server:

```
$ ssh -i myechuri-key2.pem ubuntu@3.86.247.178
ubuntu@ip-10-0-100-234:~$
```

```
$ kubectl get nodes -o wide
NAME                           STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP    OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-100-17.ec2.internal    Ready    master   36s   v1.15.0   10.0.100.17    3.86.87.104    Ubuntu 16.04.6 LTS   4.4.0-1087-aws   docker://18.9.7
ip-10-0-100-251.ec2.internal   Ready    <none>   12s   v1.15.0   10.0.100.251   3.86.187.223   Ubuntu 16.04.6 LTS   4.4.0-1087-aws   kiyot://1.2.0
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

Scale up if CPU utilization goes above 50%. Set min replicas to 1, max to 20.
```
kubectl autoscale deployment resource-consumer --cpu-percent=50 --min=1 --max=20
```

Wait for percentage utilization to transition from `unknown` to a number (takes upto 20 seconds).
```
ubuntu@ip-10-0-100-234:~$ kubectl get hpa
NAME                REFERENCE                      TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
resource-consumer   Deployment/resource-consumer   <unknown>/50%   1         20        0          11s

ubuntu@ip-10-0-100-234:~$ kubectl get hpa
NAME                REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
resource-consumer   Deployment/resource-consumer   1%/50%    1         20        1          63s
ubuntu@ip-10-0-100-234:~$ 
```

### Step 5: Generate load over 50% to trigger horizontal pod autoscaling

*Please note that we did not have to setup/configure/monitor/manage Cluster Autoscaler because we are running in Nodeless mode! Yay!*

As `resource-consumer` to consume 600 millicores will be consumed for 300 seconds.

```
curl --data "millicores=600&durationSec=300" $RESOURCE_CONSUMER_ADDRESS/ConsumeCPU
```

Watch deployments, replicasets, pods, hpa.

```
watch --differences kubectl get deployments,replicasets,pods,hpa
```

```
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/resource-consumer   12/12   12           12          101m

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.extensions/resource-consumer-6758b8d5dd   12        12        12      101m

NAME                                     READY   STATUS    RESTARTS   AGE
pod/resource-consumer-6758b8d5dd-4s6pt   1/1     Running   0          3m30s
pod/resource-consumer-6758b8d5dd-5fl2k   1/1     Running   0          3m45s
pod/resource-consumer-6758b8d5dd-5wvdc   1/1     Running   0          101m
pod/resource-consumer-6758b8d5dd-f6z82   1/1     Running   0          4m46s
pod/resource-consumer-6758b8d5dd-lgjwd   1/1     Running   0          3m45s
pod/resource-consumer-6758b8d5dd-n5c4t   1/1     Running   0          3m45s
pod/resource-consumer-6758b8d5dd-pktl8   1/1     Running   0          3m45s
pod/resource-consumer-6758b8d5dd-qggc2   1/1     Running   0          4m46s
pod/resource-consumer-6758b8d5dd-rjl65   1/1     Running   0          4m46s
pod/resource-consumer-6758b8d5dd-scx8c   1/1     Running   0          3m30s
pod/resource-consumer-6758b8d5dd-sps4m   1/1     Running   0          3m30s
pod/resource-consumer-6758b8d5dd-wbqts   1/1     Running   0          3m30s

NAME                                                    REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS
AGE
horizontalpodautoscaler.autoscaling/resource-consumer   Deployment/resource-consumer   11%/50%   1         20        12
19m
```

Notice the 11 new pod provisioned by HPA in response to the load.

Under the covers, Nodeless k8s manifested just-in-time capacity for these pods. There was no Cluster Autoscaler to configure or manage!

### Teardown

Follow [teardown instructions from kubeadm repo](https://github.com/elotl/kubeadm-aws#teardown).

### Acknowledgements

This tutorial is built on top of [Matt Kelly's tutorial](https://blog.containership.io/cerebral-vs-kubernetes-cluster-autoscaler/). We are grateful to his work in identifying and providing workarounds for bugs in Metrics Server.
