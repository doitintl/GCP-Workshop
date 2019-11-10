 When you create a cluster, gcloud adds a context to your kubectl configuration file (~/.kube/config). It then sets it as the current context, to let you operate on this cluster immediately.

```
$ kubectl config current-context
```

To test it, try a kubectl command line:

```
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

```
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


