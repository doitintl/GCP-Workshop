# Advanced GKE


## Creating a basic cluster

```bash
$ gcloud beta container clusters create gke-workshop
```

When you create a cluster, gcloud adds a context to your kubectl configuration file (~/.kube/config). It then sets it as the current context, to let you operate on this cluster immediately.

```bash
$ kubectl config current-context
```

To test it, try a kubectl command line:

```bash
$ kubectl get nodes
```

## Run a deployment
Let's run a sample deployment to verify that our cluster is working. The kubectl run command is shorthand for creating a Kubernetes deployment without the need for a YAML or JSON spec file.

$ kubectl run hello-web --image=gcr.io/google-samples/hello-app:1.0 \
       --port=8080 --replicas=3

deployment "hello-web" created

$ kubectl get pods -o wide


### What we've covered
- Creating a Kubernetes Engine cluster
- Getting credentials
- Running a simple Deployment
- Using the Kubernetes Engine workload view in the Google Cloud Console


## Node-pools

A Kubernetes Engine cluster consists of a master and nodes. Kubernetes doesn't handle provisioning of nodes, so Google Kubernetes Engine handles this for you with a concept called node pools.

A node pool is a subset of node instances within a cluster that all have the same configuration. They map to instance templates in Google Compute Engine, which provides the VMs used by the cluster. By default a Kubernetes Engine cluster has a single node pool, but you can add or remove them as you wish to change the shape of your cluster.

In the previous example, you created a Kubernetes Engine cluster. This gave us three nodes (three n1-standard-1 VMs, 100 GB of disk each) in a single node pool (called default-pool). Let's inspect the node pool:

We'll cover autoscaling in more detail later and will explain why we had 3 nodes when we created the cluster.

$ gcloud container node-pools list --cluster

### Add more pools 

If you want to add more nodes of this type, you can grow this node pool. If you want to add more nodes of a different type, you can add other node pools.

A common method of moving a cluster to larger nodes is to add a new node pool, move the work from the old nodes to the new, and delete the old node pool.

Let's add a second node pool, and migrate our workload over to it. This time we will use the larger n1-standard-2 machine type, but only create one instance.

```bash
$ gcloud container node-pools create new-pool --cluster <our_cluster_name> \
    --machine-type n1-standard-2 --num-nodes 3
Creating node pool new-pool...done.                                                                                                                                   
Created [https://container.googleapis.com/v1/projects/codelab/zones/europe-west1-d/clusters/gke-workshop/nodePools/new-pool].
NAME         MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
new-pool     n1-standard-1  100           1.11.6-gke.0

$ gcloud container node-pools list --cluster <our_cluster_name>
NAME          MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
default-pool  n1-standard-1  100           1.11.6-gke.0
new-pool      n1-standard-2  100           1.11.6-gke.0

$ kubectl get nodes
NAME                                            STATUS    ROLES     AGE       VERSION
gke-gke-workshop-default-pool-1acc373c-1txb   Ready     <none>    56m       v1.11.6-gke.0
gke-gke-workshop-default-pool-1acc373c-cklc   Ready     <none>    56m       v1.11.6-gke.0
gke-gke-workshop-default-pool-1acc373c-kzjv   Ready     <none>    57m       v1.11.6-gke.0
gke-gke-workshop-new-pool-97a76573-l57x       Ready     <none>    1m        v1.11.6-gke.0
```


Kubernetes does not reschedule Pods as long as they are running and available, so your workload remains running on the nodes in the default pool.

Look at one of your nodes using kubectl describe. Just like you can attach labels to pods, nodes are automatically labelled with useful information which lets the scheduler make decisions and the administrator perform action on groups of nodes.

Replace "[NODE NAME]" with the name of one of your nodes from the previous step.
```
$ kubectl describe node [NODE NAME] | head -n 20
Name:               gke-gke-workshop-default-pool-1acc373c-1txb
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/fluentd-ds-ready=true
                    beta.kubernetes.io/instance-type=n1-standard-1
                    beta.kubernetes.io/masq-agent-ds-ready=true
                    beta.kubernetes.io/os=linux
                    cloud.google.com/gke-nodepool=default-pool
                    failure-domain.beta.kubernetes.io/region=europe-west1
                    failure-domain.beta.kubernetes.io/zone=europe-west1-d
                    kubernetes.io/hostname=gke-gke-workshop-default-pool-1acc373c-1txb
                    projectcalico.org/ds-ready=true
Annotations:        node.alpha.kubernetes.io/ttl=0
                    volumes.kubernetes.io/controller-managed-attach-detach=true
<...>
```
You can also select nodes by node pool using the cloud.google.com/gke-nodepool label. We'll use this powerful construct shortly.

```
$ kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool
```

## Migrating pods to the new Node Pool
To migrate your pods to the new node pool, we will perform the following steps:

Cordon the existing node pool: This operation marks the nodes in the existing node pool (default-pool) as unschedulable. Kubernetes stops scheduling new Pods to these nodes once you mark them as unschedulable.
Drain the existing node pool: This operation evicts the workloads running on the nodes of the existing node pool (default-pool) gracefully.
You could cordon an individual node using the kubectl cordon command, but running this command on each node individually would be tedious. To speed up the process, we can embed the command in a loop. Be sure you copy the whole line - it will have scrolled off the screen to the right!

```
$ for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do kubectl cordon "$node"; done
node "gke-gke-workshop-default-pool-1acc373c-1txb" cordoned
node "gke-gke-workshop-default-pool-1acc373c-cklc" cordoned
node "gke-gke-workshop-default-pool-1acc373c-kzjv" cordoned
```

This loop utilizes the command kubectl get nodes to select all nodes in the default pool (using the cloud.google.com/gke-nodepool=default-pool label), and then it iterates through and runs kubectl cordon on each one.

After running the loop, you should see that the default-pool nodes have SchedulingDisabled status in the node list:

```
$ kubectl get nodes
NAME                                            STATUS                     ROLES     AGE   
gke-gke-workshop-default-pool-1acc373c-1txb   Ready,SchedulingDisabled   <none>    1h      
gke-gke-workshop-default-pool-1acc373c-cklc   Ready,SchedulingDisabled   <none>    1h      
gke-gke-workshop-default-pool-1acc373c-kzjv   Ready,SchedulingDisabled   <none>    1h      
gke-gke-workshop-new-pool-97a76573-l57x       Ready                      <none>    10m     

```

Next, we want to evict the Pods already scheduled on each node. To do this, we will construct another loop, this time using the kubectl drain command:

```
$ for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do kubectl drain --force --ignore-daemonsets --delete-local-data "$node"; done

<...>
pod "hello-web-59d96f7bd6-scknr" evicted
pod "hello-web-59d96f7bd6-v5dbq" evicted
pod "hello-web-59d96f7bd6-p92b6" evicted
node "gke-gke-workshop-default-pool-cf484031-cl8m" drained
<...>
```

```
$ kubectl get pods -o wide
NAME                         READY     STATUS    RESTARTS   AGE       IP          NODE
hello-web-5d9cdb689c-54vnv   1/1       Running   0          2m        10.60.6.5   gke-gke-workshop-new-pool-97a76573-l57x
hello-web-5d9cdb689c-pn25c   1/1       Running   0          2m        10.60.6.8   gke-gke-workshop-new-pool-97a76573-l57x
hello-web-5d9cdb689c-s6g6r   1/1       Running   0          2m        10.60.6.6   gke-gke-workshop-new-pool-97a76573-l57x
```

ou can now delete the original node pool:

```
$ gcloud container node-pools delete default-pool --cluster gke-workshop
The following node pool will be deleted.
[default-pool] in cluster [gke-workshop] in [europe-west1-d]

Do you want to continue (Y/n)?  y

Deleting node pool default-pool...done.                                                                                                                                  
Deleted [https://container.googleapis.com/v1/projects/codelab/zones/europe-west1-d/clusters/gke-workshop/nodePools/default-pool].
```

### What we've covered
- Creating node pools
- Cordoning and draining nodes
- Migrating the cluster from one machine type to another
- Creating multiple node pools with different configurations
- Deleting node pools

## Logs

### Examining Container Logs

First let's create a workload in the cluster to generate some logs. We will create an nginx Deployment and corresponding Service in order to emit some logs.

```
$ kubectl run nginx --image=nginx --expose --port=80 --service-overrides='{"spec":{"type":"LoadBalancer"}}'

service "nginx" created
deployment "nginx" created
```

Next check for the public IP of our nginx Service. When the IP address is returned you can hit Ctrl-C to exit.

It may take a couple minutes for the public IP address to be set.

```bash
$ watch kubectl get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

123.123.123.123

```

In your terminal run the following to generate load on the nginx Pod. Enter the IP address of your service instead of the IP address below.

```bash
$ INGRESS_IP=kubectl get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
$ while true; do curl -s $INGRESS_IP > /dev/null; done
```

This will run in a never ending loop sending requests to our nginx Pod. You should now be able to view the logs for the Pod. View the Stackdriver Logs by selecting Logging > Logs from the menu in the Google Cloud Platform console.


Next, let's view the metrics for our pod in Stackdriver Monitoring. Navigate to Monitoring in the menu for the Google Cloud Platform Console. A new tab should be opened for Stackdriver Monitoring. Within Stackdriver Monitoring navigate to Resources > Kubernetes Engine. You should see a list of Pods. One of the pods should look like nginx-XXXXXXXXX-YYYYY. Click on that pod in the list.

[!] It may take a couple minutes before the Pod shows up in Stackdriver Monitoring



Let's delete this deployment and service:

```
$ kubectl delete deployment,service nginx

deployment "nginx" deleted
service "nginx" deleted
```

### What we've covered
- Viewing container logs in Stackdriver Logging
- Viewing metrics in Stackdriver Monitoring
- Viewing metrics in the Workloads page on the Kubernetes Engine console


## Scaling 

Google Kubernetes Engine includes the ability to scale the cluster based on scheduled workloads. Kubernetes Engine's cluster autoscaler automatically resizes clusters based on the demands of the workloads you want to run.

Let us simulate some additional load to our web application by increasing the number of replicas:

```
$ kubectl scale deployment hello-web --replicas=20
```

deployment "hello-web" scaled
When you use kubectl run, each pod will request a default of "100 milliCPUs", or one tenth of one CPU.

Each node has a maximum capacity for each of the resource types (the amount of CPU and memory) it can provide for Pods. The scheduler ensures that, for each resource type, the sum of the resource requests of the scheduled Containers is less than the capacity of the node.

A one-core machine like an n1-standard-1 has 1000 milliCPUs, with some reserved for running system pods. You can see how much resource your node has for running pods by looking at the "Allocatable" section of kubectl describe node:

Allocatable:
 cpu:     940m
 memory:  2709028Ki
 pods:    110

Even with our larger machine, this is more work than we have space in our cluster to handle:

```
$ kubectl get pods
```

We can see that there are many pods that are stuck with status of *"Pending"*. This means that Kubernetes has not yet been able to schedule that pod to a node.

Copy the name of one of the pods marked Pending, and look at its events with kubectl describe. You should see a message like the following:

```
$ kubectl describe pod hello-web-967542450-wx1m5
<...>
Events:
  Message
  -------
  FailedScheduling        No nodes are available that match all of the following predicates:: Insufficient cpu (3).
  
```

The scheduler was unable to assign this pod to a node because there is not sufficient CPU space left in the cluster. We can add nodes to the cluster in order to make have enough resources for all of the pods in our Deployment.

Cluster Autoscaler can be enabled when creating a cluster, or you can enable it by updating an existing node pool. We will enable cluster autoscaler on our new node pool.

```
$ gcloud container clusters update gke-workshop --enable-autoscaling \
      --min-nodes=0 --max-nodes=5 --node-pool=new-pool
```

Enabling Cluster Autoscaler may take a few minutes, as it requires the Kubernetes master to restart. During this time, kubectl commands may not complete successfully.

Once autoscaling is enabled, Kubernetes Engine will automatically add new nodes to your cluster if you have created new Pods that don't have enough capacity to run; conversely, if a node in your cluster is underutilized, Kubernetes Engine can scale down the cluster by deleting the node.

After the command above completes, we can see that the autoscaler has noticed that there are pods in Pending, and creates new nodes to give them somewhere to go. After a few minutes, you will see a new node has been created, and all the pods are now Running. Hit Ctrl-C when you are done:

```
$ watch kubectl get nodes,pods
```

### What we've covered
- Scaling up Deployments
- The Pending state
- Adding the Cluster Autoscaler to a node pool


## NodeSelector / Taints / Preemptibles

Preemptible VMs are Google Compute Engine VM instances that last a maximum of 24 hours and provide no availability guarantees. Preemptible VMs are priced substantially lower than standard Compute Engine VMs and offer the same machine types and options.

If your workload can handle nodes disappearing, using Preemptible VMs with the Cluster Autoscaler lets you run work at a lower cost. To specify that you want to use Preemptible VMs you simply use the --preemptible flag when you create the node pool. But if you're using Preemptible VMs to cut costs, then you don't need them sitting around idle. So let's create a node pool of Preemptible VMs that starts with zero nodes, and autoscales as needed.

Hold on though: before we create it, how do we schedule work on the Preemptible VMs? These would be a special set of nodes for a special set of work - probably low priority or batch work. For that we'll use a combination of a NodeSelector and taints/tolerations

```bash
$ gcloud beta container node-pools create preemptible-pool \
    --cluster gke-workshop --preemptible --num-nodes 0 \
    --enable-autoscaling --min-nodes 0 --max-nodes 5 \
    --node-taints=pod=preemptible:PreferNoSchedule

Creating node pool preemptible-pool...done.
Created [https://container.googleapis.com/v1/projects/codelab/zones/europe-west1-d/clusters/gke-workshop/nodePools/preemptible-pool].
NAME               MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
preemptible-pool   n1-standard-1  100           1.9.6-gke.0

$ kubectl get nodes
```


We now have two node pools, but the new "preemptible" pool is autoscaled and is sized to zero initially so we only see the three nodes from the autoscaled node pool that we created in the previous section.

Usually as far as Kubernetes is concerned, all nodes are valid places to schedule pods. We may prefer to reserve the preemptible pool for workloads that are explicitly marked as suiting preemption — workloads which can be replaced if they die, versus those that generally expect their nodes to be long-lived.

To direct the scheduler to schedule pods onto the nodes in the preemptible pool we must first mark the new nodes with a special label called a taint. This makes the scheduler avoid using it for certain Pods.

When our application autoscales, the Kubernetes scheduler will prefer nodes that do not have the taint. This is as opposed to requiring that new workloads run on nodes without a taint. This is an easy way to allow us to run pods from system DaemonSets like Calico or fluentd without extra work.


We can then mark pods that we want to run on the preemptible nodes with a matching toleration, which says they are OK to be assigned to nodes with that taint.

Let's create a new workload that's designed to run on preemptible nodes and nowhere else.

```
$ cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: hello-web
  name: hello-preempt
spec:
  replicas: 20
  selector:
    matchLabels:
      run: hello-web
  template:
    metadata:
      labels:
        run: hello-web
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        name: hello-web
        ports:
        - containerPort: 8080
          protocol: TCP
        resources:
          requests:
            cpu: "50m"
      tolerations:
      - key: pod
        operator: Equal
        value: preemptible
        effect: PreferNoSchedule
      nodeSelector:
        cloud.google.com/gke-preemptible: "true"
EOF

deployment "hello-preempt" created
```

```
$ watch kubectl get nodes,pods -o wide
```
