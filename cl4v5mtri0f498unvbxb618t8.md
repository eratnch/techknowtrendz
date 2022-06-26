## Taking Amazon EKS Anywhere for a spin

# Prelude #

Before we discuss about EKS Anywhere, it's useful to have a basic idea about Amazon EKS and more importantly Amazon EKS Distro.

Amazon Elastic Kubernetes Service a.k.a. Amazon EKS is a managed kubernetes service from AWS to run and scale Kubernetes applications on AWS cloud platform. EKS uses Amazon EKS Distro otherwise known as EKS-D to create reliable and secure Kubernetes clusters.

EKS Distro includes binaries and containers of open-source Kubernetes, etcd (cluster configuration database), networking, and storage plugins, tested for compatibility.

Maintaining and running your own Kubernetes clusters takes a lot of effort for teams in tracking updates, figuring out compatibility between different kubernetes versions and simply keeping up-to-date with upstream kubernetes release cadence. This is where EKS-D comes to the rescue. EKS Distro reduces the need to track updates, determine compatibility, and standardize on a common Kubernetes version across teams.


# WHAT is EKS Anywhere

With that basic understanding of EKS and EKS-D, let's now take a closer look at EKS-Anywhere. 

Amazon EKS Anywhere is a new deployment option for Amazon EKS that enables you to easily create and operate Kubernetes clusters on-premises with your own virtual machines. EKS Anywhere is an open-source deployment option for Amazon EKS that builds on the strengths of EKS-D and allows teams to create and operate Kubernetes clusters on-premises. EKS Anywhere is based on the overarching design principle that supports BYOI (Bring Your Own Infrastructure) model when it comes to deploying kubernetes clusters. It supports deploying production grade kubernetes clusters on VMWare's vSphere and plans to add support for bare metal in 2022.


# WHAT problem does it solve

The salient use cases that EKS Anywhere solves are 


- Hybrid cloud consistency
-  Disconnected environment
-  Application modernization
-  Data sovereignty

The core benefit of using EKS Anywhere is that it offers customers a consistent and reliable mechanism of running Amazon's kubernetes distribution within their own on-premises infrastructure. Some enterprises have a mix of deployment architecture with some kubernetes workload running in cloud on AWS EKS while some other applications still running in on-premise kubernetes clusters. EKS Anywhere offers strong operational consistency with Amazon EKS so teams can standardize their Kubernetes operations across a hybrid cloud environment based on a unified toolset.

Businesses that have a large on-premises footprint and want to modernize their applications can leverage EKS Anywhere to simplify the creation and operation of on-premises Kubernetes clusters and focus more on developing and modernizing applications. In addition, customers who want to keep their data within private data centers due to legal reasons can benefit by using trusted Amazon EKS Kubernetes distribution and tools to where their data needs to be.


# WHY should one use it

Kubernetes adoption is growing every day and the tooling around it also keeps on piling up. While this provides customers with multiple options, it is quite challenging to ensure that teams pick the right tools for the job and doesn't add complexity in their operations workflow. Keeping pace with upstream kubernetes release cadence without breaking existing applications is also a non-trivial task. 

Teams that are operating kubernetes clusters on-premises typically need to take on a lot of operational challenges such as creating and upgrading clusters in a timely manner with upstream releases, maintaining and resolving version mismatches between kubernetes releases and integrating a variety of third party tools to perform cluster operations. The same applies to hybrid cluster set ups and leads to unnecessary complexity, fragmented tooling and support options, and inconsistencies between the cloud and on-premises clusters that make it hard to manage applications across environments.

With Amazon EKS Anywhere, teams have Kubernetes operational tooling that is consistent with Amazon EKS and is optimized to simplify cluster installation with default configurations for the operating system and networking needed to operate Kubernetes on-premises.  If you're someone who wants to reduce operational complexity, adopt a consistent and reliable workflow of managing kubernetes clusters across cloud and on-premises, leverage latest tooling and security hardened updated kubernetes distribution to operate on then EKS Anywhere might be a worthy option for you.



# Kicking The Tires 

EKS Anywhere allows you to create and manage production kubernetes cluster on VMWare vSpehere. However, if you don't have a vSphere environment at your disposal, EKS Anywhere also supports creating development clusters locally with Docker provider.

Here, I will walk you through the cluster creation process on a virtual machine using the Docker provider. This set up is for local use and not recommended for Production purposes. However the concepts around the cluster creation and management workflow across providers are the same. 

## Clusters and more Clusters

 This is the path we're going to take for the purposes of this post. 

### Cluster Management Workflow

The EKS Anywhere cluster creation process makes it easy not only to bring up a cluster initially, but also to update configuration settings and to upgrade Kubernetes versions going forward. The cluster creation process  involves stepping through different types of clusters.

- Bootstrap cluster - A temporary kubernetes clusters that's ephemeral in nature and is solely used for creating a management cluster.

- Management cluster - A Kubernetes cluster that manages the lifecycle of Workload Clusters.

- Workload cluster - A Kubernetes cluster whose lifecycle is managed by a Management Cluster.

> To manage the lifecycle of a Workload kubernetes cluster we need to have a Management kubernetes cluster in place first. And to have a Management cluster, we need to spin up a bootstrap kubernetes cluster to start the cluster creation workflow.

Think of the Bootstrap cluster as a launchpad for EKS Anywhere to start the Workload cluster creation process. Once it's able to create the Management cluster, from that point on, Management cluster continues the process and takes over the lifecycle management of the Workload cluster. The essence of this design paradigm is to make kubernetes clusters "self-aware" and be able to manage its lifecycle themselves without the need of a launchpad (i.e. Bootstrap cluster). A common practice is to delete the bootstrap cluster once its job is done and repurpose the infrastructure to save resource.

 An obvious question about the above scenario is how to spin up the bootstrap cluster and how to avoid a chicken and egg situation where we attempt to create bootstrap cluster using EKS Anywhere even before there's a launchpad cluster in place.

Enter [KinD] (https://kind.sigs.k8s.io/) or Kubernetes in Docker. KinD can create  kubernetes clusters that run kubernetes as docker containers. EKS Anywhere runs a `KinD` cluster on an administrative workstation or virtual machine to act as a `bootstrap cluster`.

Let's roll up our sleeves now and see EKS Anywhere in action!

### Prerequisites

To start with, prepare your `Administrative` Workstation with following pre-requisites.

- Docker 20.x.x
- Ubuntu (20.04.2 LTS). If you're on a Mac use `10.15`
- 4 CPU cores
- 16GB memory
- 30GB free disk space

> Make sure that your local workstation or virtual machine in cloud meets all of the above requirements. 

I used a virtual machine in cloud running Ubuntu 20.04 LTS with above configurations.

### Docker
 [Install Docker on Ubuntu](https://docs.docker.com/engine/install/ubuntu/).
Check the docker version to make sure it's 20.x.x. 

```
$ docker version
Client: Docker Engine - Community
 Version:           20.10.17
 API version:       1.41
 Go version:        go1.17.11
 Git commit:        100c701
 Built:             Mon Jun  6 23:02:57 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.17
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.17.11
  Git commit:       a89b842
  Built:            Mon Jun  6 23:01:03 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.6
  GitCommit:        10c12954828e7c7c9b6e0ea9b0c02b01407d3ae1
 runc:
  Version:          1.1.2
  GitCommit:        v1.1.2-0-ga916309
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

### kubectl

You need `kubectl` installed to connect to kubernetes clusters from your workstation. If you don't have it installed, use the `snap` commands to install it.

```
sudo snap install kubectl --classic
```

### eksctl
Install the latest release of eksctl. The EKS Anywhere plugin requires eksctl version 0.66.0 or newer. 
```
curl "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
    --silent --location \
    | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

```

### eksctl anywhere plugin 
Install the `eksctl-anywhere` plugin.
```
export EKSA_RELEASE="0.9.1" OS="$(uname -s | tr A-Z a-z)" RELEASE_NUMBER=12
curl "https://anywhere-assets.eks.amazonaws.com/releases/eks-a/${RELEASE_NUMBER}/artifacts/eks-a/v${EKSA_RELEASE}/${OS}/amd64/eksctl-anywhere-v${EKSA_RELEASE}-${OS}-amd64.tar.gz" \
    --silent --location \
    | tar xz ./eksctl-anywhere
sudo mv ./eksctl-anywhere /usr/local/bin/

```
Verify your installed version.
```
$ eksctl anywhere version
v0.9.1
```

## Create your local EKS Anywhere cluster

Now that you have all the tools installed, let's proceed with creating the `local` EKS Anywhere cluster using the `Docker` provider.

First we create a cluster configuration and save it in a file.

```
$ CLUSTER_NAME=rc-dev
ubuntu@ip-10-0-0-12:~$ eksctl anywhere generate clusterconfig $CLUSTER_NAME \
>    --provider docker > $CLUSTER_NAME.yaml
```

Check the configuration file.
```
$ cat rc-dev.yaml 
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: rc-dev
spec:
  clusterNetwork:
    cniConfig:
      cilium: {}
    pods:
      cidrBlocks:
      - 192.168.0.0/16
    services:
      cidrBlocks:
      - 10.96.0.0/12
  controlPlaneConfiguration:
    count: 1
  datacenterRef:
    kind: DockerDatacenterConfig
    name: rc-dev
  externalEtcdConfiguration:
    count: 1
  kubernetesVersion: "1.22"
  managementCluster:
    name: rc-dev
  workerNodeGroupConfigurations:
  - count: 1
    name: md-0

---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: DockerDatacenterConfig
metadata:
  name: rc-dev
spec: {}

---
```
By default, EKS Anywhere creates a kubernetes cluster with one control plane and one worker node. Currently it installs the kubernetes `1.22` version. 

Another configuration worth noticing is the default cni provider which is `cilium`.

You can customize and alter these settings to suit your workload cluster requirement. 

### Start Small

We're going to start with installing our first EKS Anywhere cluster with bare minimum set up with 1 control plane and 1 worker node. Later on, we'd increase the worker node count.

Once we have the configuration file, all you need to do is

```
$ time eksctl anywhere create cluster -f $CLUSTER_NAME.yaml
Performing setup and validations
Warning: The docker infrastructure provider is meant for local development and testing only
✅ Docker Provider setup is valid
✅ Validate certificate for registry mirror
✅ Create preflight validations pass
Creating new bootstrap cluster
Provider specific pre-capi-install-setup on bootstrap cluster
Installing cluster-api providers on bootstrap cluster
Provider specific post-setup
Creating new workload cluster
Installing networking on workload cluster
Installing cluster-api providers on workload cluster
Installing EKS-A secrets on workload cluster
Moving cluster management from bootstrap to workload cluster
Installing EKS-A custom components (CRD and controller) on workload cluster
Installing EKS-D components on workload cluster
Creating EKS-A CRDs instances on workload cluster
Installing AddonManager and GitOps Toolkit on workload cluster
GitOps field not specified, bootstrap flux skipped
Writing cluster config file
Deleting bootstrap cluster
🎉 Cluster created!

real	5m1.553s
user	0m2.941s
sys	0m2.084s
```
Within approx. 5 mins you have the workload cluster ready for use without much hassle; that's pretty neat!

By default, the console log above highlights the different stages of the workload cluster creation lifecycle; however if you want to see more logs, then add the `-v` parameter in the `eksctl anywhere` cluster create command to tun on the verbose mode. 

You can check the bootstrap cluster by issuing the below command. Install `KinD` if you don't have it on your workstation by following [these](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) instructions.

```
$ kind get clusters
rc-dev
```


Now it's time to verify the cluster and check kubernetes version installed.

``` 
# export the kubeconfig to point to the cluster
$ export KUBECONFIG=${PWD}/${CLUSTER_NAME}/${CLUSTER_NAME}-eks-a-cluster.kubeconfig

# check nodes of the k8s cluster
$ kubectl get nodes
NAME                           STATUS   ROLES                  AGE   VERSION
rc-dev-gzktk                   Ready    control-plane,master   15m   v1.22.6-eks-bb942e6
rc-dev-md-0-7c4c7f595d-5lcq8   Ready    <none>                 14m   v1.22.6-eks-bb942e
```


## Spin Up a multi node cluster

While the basic cluster with a control plane and a worker node is nice, having the ability to spin up a multi node kubernetes cluster is fantastic. Let's see if EKS Anywhere is up to the task.

In order to increase the number of nodes in the cluster, you need to modify the cluster configuration file. There's no option to pass this as a parameter to the `eksctl anywhere` command; at least for now.

```
$ cp rc-dev.yaml rc-dev-multinode.yaml

#change metadata and name of the cluster configuration
$ sed -i 's/rc-dev/rc-dev-multinode/g' rc-dev-multinode.yaml
```
And finally, change the `workerNodeGroupConfigurations.count` value to 3. Now let's deploy the cluster.

```
$ eksctl anywhere create cluster -f rc-dev-multinode.yaml 
```

## Verify cluster

Export the kubeconfig file as before and check nodes status

```
$ kubectl get nodes
NAME                                  STATUS   ROLES                  AGE     VERSION
$ kubectl get nodes
NAME                                     STATUS   ROLES                  AGE     VERSION
rc-dev-multinode-7dwtd                   Ready    control-plane,master   2m40s   v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-dsfrw   Ready    <none>                 2m9s    v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-gpx2x   Ready    <none>                 2m10s   v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-tx6b4   Ready    <none>                 2m10s   v1.22.6-eks-bb942e6
```
Also, check for all system pods if they're in `running` state.

```
$ kubectl get po -A
NAMESPACE                           NAME                                                             READY   STATUS    RESTARTS   AGE
capd-system                         capd-controller-manager-6466d54c9d-j9klq                         1/1     Running   0          112s
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-7c6bcf98b7-tb2zx       1/1     Running   0          2m1s
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-68b4c848dc-zz6gs   1/1     Running   0          115s
capi-system                         capi-controller-manager-64647f455d-x5zx7                         1/1     Running   0          2m3s
cert-manager                        cert-manager-8674857d7b-5hmgq                                    1/1     Running   0          2m41s
cert-manager                        cert-manager-cainjector-f5b94ccdf-9bglv                          1/1     Running   0          2m41s
cert-manager                        cert-manager-webhook-84fbd6fb68-6xkkj                            1/1     Running   0          2m40s
eksa-system                         eksa-controller-manager-79b4b76bb8-lbc79                         2/2     Running   0          92s
etcdadm-bootstrap-provider-system   etcdadm-bootstrap-provider-controller-manager-664f699b7c-q44db   1/1     Running   0          2m
etcdadm-controller-system           etcdadm-controller-controller-manager-59dc96c7b9-hpnnp           1/1     Running   0          118s
kube-system                         cilium-858bk                                                     1/1     Running   0          2m51s
kube-system                         cilium-89r45                                                     1/1     Running   0          2m51s
kube-system                         cilium-bxfv6                                                     1/1     Running   0          2m51s
kube-system                         cilium-operator-7698596ff4-2k264                                 1/1     Running   0          2m51s
kube-system                         cilium-operator-7698596ff4-cgjqn                                 1/1     Running   0          2m51s
kube-system                         cilium-z4zgt                                                     1/1     Running   0          2m51s
kube-system                         coredns-55467bc785-n4mv2                                         1/1     Running   0          3m15s
kube-system                         coredns-55467bc785-qgdhx                                         1/1     Running   0          3m15s
kube-system                         kube-apiserver-rc-dev-multinode-7dwtd                            1/1     Running   0          3m18s
kube-system                         kube-controller-manager-rc-dev-multinode-7dwtd                   1/1     Running   0          3m18s
kube-system                         kube-proxy-2npkx                                                 1/1     Running   0          3m16s
kube-system                         kube-proxy-7j7vm                                                 1/1     Running   0          2m56s
kube-system                         kube-proxy-8ntxj                                                 1/1     Running   0          2m56s
kube-system                         kube-proxy-cwgtn                                                 1/1     Running   0          2m55s
kube-system                         kube-scheduler-rc-dev-multinode-7dwtd                            1/1     Running   0          3m18s
     

```

To verify that a cluster control plane is up and running, use the `kubectl` command to show that the control plane pods are all running.

```
$ kubectl get po -A -l control-plane=controller-manager
NAMESPACE                           NAME                                                             READY   STATUS    RESTARTS   AGE
capd-system                         capd-controller-manager-6466d54c9d-j9klq                         1/1     Running   0          20m
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-7c6bcf98b7-tb2zx       1/1     Running   0          20m
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-68b4c848dc-zz6gs   1/1     Running   0          20m
capi-system                         capi-controller-manager-64647f455d-x5zx7                         1/1     Running   0          20m
etcdadm-bootstrap-provider-system   etcdadm-bootstrap-provider-controller-manager-664f699b7c-q44db   1/1     Running   0          20m
etcdadm-controller-system           etcdadm-controller-controller-manager-59dc96c7b9-hpnnp           1/1     Running   0          20m
```




Once you've verified that all nodes in the workload cluster are in `ready` state and the pods  are in `Running` status, you can go ahead and deploy some test workloads onto the cluster. 


## Deploy sample workload 

To deploy a sample test workload in your shiny new multinode workload cluster, do

```
$ kubectl apply -f "https://anywhere.eks.amazonaws.com/manifests/hello-eks-a.yaml"
deployment.apps/hello-eks-a created
service/hello-eks-a created

```

Check all kubernetes resources in the `default` namespace.

```
$ kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/hello-eks-a-9644dd8dc-znwzr   1/1     Running   0          37s

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/hello-eks-a   NodePort    10.106.132.175   <none>        80:30687/TCP   37s
service/kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP        6m32s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-eks-a   1/1     1            1           37s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-eks-a-9644dd8dc   1         1         1       37s 
```

To access the default web-page of the sample workload, forward the deployment port to your localhost
```
$ kubectl port-forward deploy/hello-eks-a 8000:80
Forwarding from 127.0.0.1:8000 -> 80
Forwarding from [::1]:8000 -> 80
```

From a second terminal, try accessing the webpage by doing `curl localhost:8000`


![Screen Shot 2022-06-26 at 12.35.28 AM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656229177222/UrfVS0j3Y.png align="left")


## Scale your cluster

Currently, the only way you can scale the workload cluster is by manually incrementing the number of control plane and worker nodes in the cluster configuration file and upgrading the cluster. 

Increment the worker node count to 5 as shown below

```
$ cat rc-dev-multinode-more.yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: rc-dev-multinode
spec:
  clusterNetwork:
    cniConfig:
      cilium: {}
    pods:
      cidrBlocks:
      - 192.168.0.0/16
    services:
      cidrBlocks:
      - 10.96.0.0/12
  controlPlaneConfiguration:
    count: 1
  datacenterRef:
    kind: DockerDatacenterConfig
    name: rc-dev-multinode
  externalEtcdConfiguration:
    count: 1
  kubernetesVersion: "1.21"
  managementCluster:
    name: rc-dev-multinode
  workerNodeGroupConfigurations:
  - count: 5
    name: md-0

---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: DockerDatacenterConfig
metadata:
  name: rc-dev-multinode
spec: {}

---
```

Let's give it a try to see if we can upgrade the workload cluster to add more worker nodes.

```
$ eksctl anywhere upgrade cluster -f rc-dev-multinode-more.yaml 
Performing setup and validations
✅ Docker Provider setup is valid
✅ Validate certificate for registry mirror
✅ Control plane ready
✅ Worker nodes ready
✅ Nodes ready
✅ Cluster CRDs ready
✅ Cluster object present on workload cluster
✅ Upgrade cluster kubernetes version increment
✅ Validate immutable fields
✅ Upgrade preflight validations pass
Ensuring etcd CAPI providers exist on management cluster before upgrade
Upgrading core components
Pausing EKS-A cluster controller reconcile
Pausing Flux kustomization
GitOps field not specified, pause flux kustomization skipped
Creating bootstrap cluster
Installing cluster-api providers on bootstrap cluster
Moving cluster management from workload to bootstrap cluster
Upgrading workload cluster
Moving cluster management from bootstrap to workload cluster
Applying new EKS-A cluster resource; resuming reconcile
Resuming EKS-A controller reconciliation
Updating Git Repo with new EKS-A cluster spec
GitOps field not specified, update git repo skipped
Forcing reconcile Git repo with latest commit
GitOps not configured, force reconcile flux git repo skipped
Resuming Flux kustomization
GitOps field not specified, resume flux kustomization skipped
Writing cluster config file
🎉 Cluster upgraded!
```

Let's check the nodes to verify the state of the cluster.

```
$ kubectl get nodes
NAME                                     STATUS   ROLES                  AGE   VERSION
rc-dev-multinode-7dwtd                   Ready    control-plane,master   43m   v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-dsfrw   Ready    <none>                 43m   v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-gpx2x   Ready    <none>                 43m   v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-stkx8   Ready    <none>                 10m   v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-tx6b4   Ready    <none>                 43m   v1.22.6-eks-bb942e6
rc-dev-multinode-md-0-86786b8dfb-vqpf9   Ready    <none>                 10m   v1.22.6-eks-bb942e6
```
And sure enough, we have all 5 worker nodes in `ready` status.


You can scale workload clusters in a semi automatic way by storing your cluster config manifest in git and then having a CI/CD system deploy your changes. Or you can use a GitOps controller to apply the changes. EKS Anywhere allows cluster management by incorporating GitOps. It uses Flux to manage clusters with GitOps. More on this in a next post.
 

## Adding Integration to your EKS Anywhere Cluster
Standing up a workload cluster is awesome; however as mentioned earlier, Kubernetes can quickly become operation extensive and needs the flexibility of integrating with a wide array of tools. And EKS Anywhere doesn't disappoint.

EKS Anywhere offers custom integration for certain third-party vendor components, namely Ubuntu TLS, Cilium, and Flux. It also provides flexibility for you to integrate with your choice of tools in other areas. This is really great and frees up the teams from being locked in with a certain vendor and enables them to swap out default components with tools of their choice; for instance, swapping out cilium with a different CNI such as calico.

Some of the key integration components compatible with EKS Anywhere clusters are


- Load Balancer - KubeVip, MetalLB
- Local container repository - Harbor
- Monitoring 	- Prometheus , Grafana , Datadog , or NewRelic
- Logging 	- Splunk or Fluentbit
- Secret Management - Hashicorp Vault
- Policy agent - Open Policy Agent (OPA)
- Service mesh - Istio, Linkerd
- Cost management 	- KubeCost
- Etcd backup and restore - Valero

# Cluster API Kubernetes (CAPI)

Any introduction to EKS Anywhere will not be complete without mentioning Cluster API kubernetes project.

[Kubernetes Cluster API](https://cluster-api.sigs.k8s.io/) or CAPI is a Kubernetes SIG (Special Interest Group) project focused on providing declarative APIs and tooling to simplify provisioning, upgrading, and operating multiple Kubernetes clusters. Two of the main goals of CAPI project are

- To manage the lifecycle (create, scale, upgrade, destroy) of Kubernetes-conformant clusters using a declarative API.
- To work in different environments, both on-premises and in the cloud.

To add some context, the notion of Bootstrapping cluster, Management cluster and Workload cluster in the context of managing kubernetes cluster lifecycle was first championed by CAPI.  

EKS Anywhere works under the same principles and uses CAPI underneath to implement some of these features. It uses an infrastructure provider model for creating, upgrading, and managing Kubernetes clusters that leverages the Kubernetes Cluster API project. The first supported EKS Anywhere provider, VMware vSphere, is implemented based on the Kubernetes Cluster API Provider vSphere (CAPV) specifications. Similarly, EKS Anywhere supports Cluster API for Docker Provider (CAPD) for creating development and test workload clusters.

The EKS Anywhere project wraps Cluster API, various other CLIs and plugins (eksctl cli, anywhere plugin, kubectl, aws-iam-authenticator)  and bundles them in a single package to simplify the creation of workload clusters.

# Epilogue
Amazon EKS Anywhere aims to solve the pain points of managing the lifecycle of  kubernetes clusters in on-premise set up and provides a consistent and reliable workflow for creating and managing kubernetes clusters across deployment models and providers. With its currently supported VMWare's vSphere provider and upcoming support for bare metal, I am keen to explore its potential and study its adoption across customers and teams.

Let me know your thoughts in the comments.










