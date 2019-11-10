# GCP-Workshop
Selection of GCP Workshops

# Day 1 

## Projects Folders and organizations 
- Billing Acccount
  - [Setup Account](https://codelabs.developers.google.com/codelabs/gcp-aws-accounts-and-billing-v2/index.html?index=..%2F..index#0)
- Organization  
  - [Resource Hierarchy](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy#resource-hierarchy-detail)
  - Folders
    - [Create Folder](https://cloud.google.com/resource-manager/docs/creating-managing-folders#creating-folders)
  - Projects
    - [Create project](https://cloud.google.com/resource-manager/docs/creating-managing-folders#creating_a_project_in_a_folder)
    

## IAM

- User Account / Service Account
  - [Create a Service Account](https://cloud.google.com/iam/docs/creating-managing-service-accounts)
  - [Create a Service Account Key](https://cloud.google.com/iam/docs/creating-managing-service-account-keys#creating_service_account_keys)
- Roles - Primitive/BuiltIn/Custom
  - [Understand Roles](https://cloud.google.com/iam/docs/understanding-roles#predefined_roles)
  - [Create a Custom IAM Role and Assing it to an IAM User](https://cloud.google.com/iam/docs/granting-roles-to-service-accounts#granting_access_to_a_service_account_for_a_resource)


## GCP Compute Basics
- Cloud Shell
- Google Compute command-line tool
  - [Install Google Cloud SDK (gcloud)](https://cloud.google.com/sdk/docs/downloads-versioned-archives) 
- Create an instance
  - [Create instance using the console/gcloud](https://codelabs.developers.google.com/codelabs/cloud-create-a-vm/index.html?index=..%2F..index#0)
- Deploy container
  - Deploy a container using the console
  - Deploy a container using gcloud
- Preemptible VMs (GPUs)
  - Create a Preemptiable Instance using gcloud

## GCP's network infrastructure (regions, zones)

- [GCP Regions and Network](https://cloud.google.com/about/locations/?tab=regions)
- [Detailed network Infrastructure](https://peering.google.com/#/infrastructure)
- VPC
  - Create a VPC
- Firewall
  - [Deploy a webserver an open it to the world](https://codelabs.developers.google.com/codelabs/cloud-compute-engine/index.html?index=..%2F..index#0)
- Network Tags
  - Create a network tag for 8080 ports
- Load Balancer
  - [Summary of GCP Load Balancers] (https://cloud.google.com/load-balancing/docs/choosing-load-balancer#summary_of_cloud_load_balancers)
  - [Choosing a load balancer](https://cloud.google.com/load-balancing/images/choose-lb.svg)
  - [Create a webserver group and load balancer](https://codelabs.developers.google.com/codelabs/cloud-webapp-hosting-gce/index.html?index=..%2F..index#0)


## GKE/K8S foundation
- Introduction and Environment Setup
  - Create a cluster
  - Create a cluster from a template
- GKE Architecture Review
  - Zonal / Multizone /Regional

- Storage and Networking Concepts
  - VPC Native
  - Master Authorized Networks
  - Private Cluster
  
- GKE Maintenance
  - Upgrades
  - Repair
  - Cluster 
  
- Real-time Monitoring and Reporting
- Troubleshooting Tactics
  - Troubleshooting cluster
    - Stackdriver Logs
    - [Node troubleshooting (toolbox)](https://cloud.google.com/kubernetes-engine/docs/troubleshooting#ConnectivityIssues)
    - [Serial Outout](https://cloud.google.com/compute/docs/instances/viewing-serial-port-output#viewing_serial_port_output)

- Resource and Security Management

- [Tips and tricks for working with GKE](tips.md)

- [Kubernetes Basics](https://codelabs.developers.google.com/codelabs/cloud-orchestrate-with-kubernetes/#0)


# Day 2

## GKE deep dive
- Interface Capabilities (WebUI, CLI, API)
- Troubleshooting applications
  - Stackdriver logs
- Scaling Applications  
  - Scaling deployments
  - Cluster auto scaler
  - Preemptible VMs/Nodes
  - NodeSelector
  
  
- Build and Deployment Capabilities
- Continuous Integration/Deployment Capabilities
  - [Continuous Deployment with Cloud Build](https://codelabs.developers.google.com/codelabs/cloud-builder-gke-continuous-deploy/index.html?index=..%2F..index#0)
- Troubleshooting Application Deployments
- [Working with Pub/Sub](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform) 
- Working with Cloud SQL
  - [Connecting to CloudSQL](https://codelabs.developers.google.com/codelabs/connecting-to-cloud-sql/index.html?index=..%2F..index#0) 
- Notes about Firestore / Spanner
