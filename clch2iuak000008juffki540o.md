# Reduce cross-AZ traffic costs on EKS using topology aware hints

From a high availability standpoint, it's considered a best practice to spread workloads across multiple nodes in an EKS cluster. In addition to having multiple replicas of the application, one should also consider spreading the workload across multiple Availability Zones to attain high availability and improve reliability. This ensures fault-tolerance and avoids application downtime in the event of a worker node failure. One way to achieve this kind of deployment in EKS is to use `podAnitiAffinity`. For example, the below manifest tells EKS scheduler to deploy each replica of the Pod on a node that's in a separate Availability Zone(AZ).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-az
  labels:
    app: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-server
            topologyKey: kubernetes.io/zone
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

The above manifest makes use of topologyKey `kubernetes.io/zone` . It tells the Kubernetes scheduler not to schedule two Pods in same AZ.

One of the other approaches that can be used to spread Pods across AZs is to use [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) which was GA-ed in Kubernetes 1.19. This mechanism aims to spread pods evenly onto multiple node topologies.

While both of these approaches provide high-availability and resiliency for application workloads, customers incur costs for data transfers in inter-AZ traffic routing within an EKS cluster. For large EKS clusters running hundreds of nodes and thousands of pods, the data transfer costs for cross-AZ traffic can be significant.

## Enter *Topology Aware Hints*

To address cross-AZ data transfer costs (which comes up during many EKS conversations on cost optimzation), pods running in a cluster must be able to perform topology-aware routing based on Availability Zone. And this is precisely what [Topology Aware Hints](https://kubernetes.io/docs/concepts/services-networking/topology-aware-hints/) helps achieve. Topology Aware Hints provides a mechanism to help keep traffic within the zone it originated from. Prior to topology-aware-hints, Service [topology-keys](https://kubernetes.io/docs/concepts/services-networking/service-topology/#examples) could be used for similar functionality. This was deprecated in kubernetes 1.21 in favor of topology-aware-hints which was introduced in kubernetes 1.21 and became "beta" in Kubernetes 1.23. With [EKS 1.24](https://aws.amazon.com/about-aws/whats-new/2022/11/amazon-eks-eks-distro-support-kubernetes-version-1-24/) however, this is enabled by default and EKS users and customers can leverage this feature to keep kubernetes service traffic within the same AZ.

Let's dive in further and see this in action!

For the purposes of this blogpost, let's create a three-node EKS cluster.

Type the following command in your cloud9 terminal.

```yaml
cat <<EOF>> eks-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: topology-demo-cluster
  region: us-west-2
  version: "1.24"
managedNodeGroups:
  - name: appservers
    instanceType: t3.xlarge
    desiredCapacity: 3
    minSize: 1
    maxSize: 4
    labels: { role: appservers }
    volumeSize: 8
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        xRay: true
        cloudWatch: true
        albIngress: true
    ssh: 
      enableSsm: true
EOF
eksctl create cluster -f eks-config.yaml
```

Once the cluster is created, check the status of the worker nodes and their distribution across AZs.

```bash
kubectl get nodes -L topology.kubernetes.io/zone
NAME                                           STATUS   ROLES    AGE   VERSION               ZONE
ip-192-168-4-149.us-west-2.compute.internal    Ready    <none>   36h   v1.24.7-eks-fb459a0   us-west-2b
ip-192-168-48-125.us-west-2.compute.internal   Ready    <none>   36h   v1.24.7-eks-fb459a0   us-west-2c
ip-192-168-75-68.us-west-2.compute.internal    Ready    <none>   36h   v1.24.7-eks-fb459a0   us-west-2d
```

We have each worker node in a separate AZ deployed in our EKS cluster. Let's now try to run a sample application in this cluster.

Use the below application manifest to deploy three replicas of our sample application to deploy in the newly created EKS cluster as below

```yaml
cat <<EOF>> app-manifest.yaml
apiVersion: v1
kind: Namespace
metadata:
   name: topology-demo-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: getaz
  namespace: topology-demo-ns
  labels:
    app: getaz
spec:
  replicas: 3
  selector:
    matchLabels:
      app: getaz
  template:
    metadata:
      labels:
        app: getaz
      namespace: topology-demo-ns
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: getaz
      containers:
      - name: getaz-container
        image: getazcontainer:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: web-port
        resources:
          requests:
            cpu: "256m"
---
apiVersion: v1
kind: Service
metadata:
  name: getazservice
  namespace: topology-demo-ns
spec:
  selector:
    app: getaz
  ports:
    - port: 80
      targetPort: web-port
      protocol: TCP
EOF
kubectl apply -f app-manifest.yaml
```

The application manifest creates -

* a Namespace named "topology-demo-ns"
    
* a Deployment named "getaz" with three Pods. Each Pod runs a container named "getaz-container".
    
* a Service named "getazservice".
    

The "getaz" Pods and Service "getazservice" all run in the "topology-demo-ns" namespace.

In the above example, we're using the Pod `topologySpreadConstraints` with `maxSkew` set to 1 and `whenUnsatisfiable` set to "DoNotSchedule" to deploy each replica of our sample application in a separate AZ. This example leverages a well-known node label called `topology.kubernetes.io/zone` that worker nodes in an EKS cluster is assigned to by default, as the `topologyKey` in the pod topology spread. To get the labels on a worker node in the EKS cluster that we spun up, use the below command

```bash
$ kubectl describe node ip-192-168-48-125.us-west-2.compute.internal
Name:               ip-192-168-48-125.us-west-2.compute.internal
Roles:              <none>
Labels:             alpha.eksctl.io/cluster-name=topology-demo-cluster
                    alpha.eksctl.io/nodegroup-name=appservers
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t3.xlarge
                    beta.kubernetes.io/os=linux
                    eks.amazonaws.com/capacityType=ON_DEMAND
                    eks.amazonaws.com/nodegroup=appservers
                    eks.amazonaws.com/nodegroup-image=ami-0b149b4c68ab69dce
                    eks.amazonaws.com/sourceLaunchTemplateId=lt-0a47ee5069d44e8d4
                    eks.amazonaws.com/sourceLaunchTemplateVersion=1
                    failure-domain.beta.kubernetes.io/region=us-west-2
                    failure-domain.beta.kubernetes.io/zone=us-west-2c
                    k8s.io/cloud-provider-aws=8d60a23f89f8b00a31bfef5d05edc662
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-192-168-48-125.us-west-2.compute.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=t3.xlarge
                    role=appservers
                    topology.kubernetes.io/region=us-west-2
                    topology.kubernetes.io/zone=us-west-2c
```

In the `topologySpreadConstraints` section of the example manifest,

**maxSkew** defines the degree to which Pods may be distributed unevenly. This field must be filled out, and the value must be greater than zero. Its semantics vary depending on the value of *whenUnsatisfiable* field.

**whenUnsatisfiable** specifies how to handle a Pod placement that does not satisfy the spread constraint:

\- *DoNotSchedule* (the default value) instructs the scheduler not to schedule it.

\- *ScheduleAnyway* instructs the scheduler to continue scheduling it while prioritizing Nodes with the lowest skew.

Let's check the status and spread of our application pods.

```bash
kubectl get po -n topology-demo-ns -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE                                           NOMINATED NODE   READINESS GATES
getaz-9685bbd44-65wcn   1/1     Running   0          2m    192.168.63.154   ip-192-168-48-125.us-west-2.compute.internal   <none>           <none>
getaz-9685bbd44-kf7gs   1/1     Running   0          2m    192.168.69.57    ip-192-168-75-68.us-west-2.compute.internal    <none>           <none>
getaz-9685bbd44-tjqkd   1/1     Running   0          2m    192.168.24.149   ip-192-168-4-149.us-west-2.compute.internal    <none>           <none>
```

We see from above output that each replica is running on a separate node and since each node is running in a separate AZ, effectively we have three pods each running in a different AZ in the EKS cluster.

For getting detail information about `topologySpreadConstraints`, you can use `kubectl explain Pod.spec.topologySpreadConstraints` command. You can mix and match these attributes to achieve different spread topologies.

Let us now check the `Service` that got created by deploying the app-manifest.yaml file.

```bash
kubectl -n topology-demo-ns get svc
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
getazservice   ClusterIP   10.100.9.165   <none>        80/TCP    53m

kubectl -n topology-demo-ns describe svc getazservice 
Name:              getazservice
Namespace:         topology-demo-ns
Labels:            <none>
Annotations:       <none>
Selector:          app=getaz
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.100.9.165
IPs:               10.100.9.165
Port:              <unset>  80/TCP
TargetPort:        web-port/TCP
Endpoints:         192.168.24.149:3000,192.168.63.154:3000,192.168.69.57:3000
Session Affinity:  None
Events:            <none>
```

We have a `Service` named "getazservice" of `ClusterIP` type deployed. The service doesn't have any `Annotations` set on it.

As next step, let's deploy a test container that we're going to use to call "getazservice" and check if there are any inter-AZ calls we can spot.

Use the below command to deploy a curl container and ensure `curl` is installed.

```bash
kubectl run curl-debug --image=radial/busyboxplus:curl -l "type=debug" -n topology-demo-ns -it --tty  sh
# check if curl is installed
curl --version
#exit the container
exit
```

Once the debug container is running, create a bash script to call "getazservice" in a loop and print the Availability Zone of the pod that responded to the call.

```bash
kubectl exec -it --tty -n topology-demo-ns $(kubectl get pod -l  "type=debug" -n topology-demo-ns -o  jsonpath='{.items[0].metadata.name}') sh
 #create a test script and call service
 cat <<EOF>> test.sh
 n=1
 while [ \$n -le 5 ]
 do
     curl -s getazservice.topology-demo-ns
     sleep 1
     echo "---"
     n=\$(( n+1 ))
 done
EOF
 chmod +x test.sh
 clear
 ./test.sh
 #exit the test container
 exit
```

The above execution of the test script in the debug container should produce an output like below which shows that the calls to the "getazservice" Service and its backing Pods are distributed across AZs.

```bash
us-west-2d---
us-west-2b---
us-west-2d---
us-west-2c---
us-west-2d---
```

The load-balancing and forwarding logic of the service call in this case is based on `kube-proxy` mode. EKS by default implements "iptables" mode of `kube-proxy`. When the `curl-debug` container sends the `curl` request to the "getazservice" virtual IP, the packet is then processed by the iptables rules on that worker node which are configured by the `kube-proxy`. Then a Pod backing the "getazservice" `Service` gets chosen at random by default. For detailed documentation on different kube-proxy modes(iptables, ipvs) please refer to [Kubernetes Documentation](https://kubernetes.io/docs/reference/networking/virtual-ips/).

To avoid this "randomness" of routing and reduce the cost of inter-AZ traffic routing and network latency, topology-aware-hints can be activated for the `Service` to ensure that the service call is routed to a Pod that resides in the same AZ as that of the Pod which the request originated from.

To enable topology-aware routing, simply add the `service.kubernetes.io/topology-aware-hints annotation` to "auto" for the "getazservice" as below and re-deploy the manifest.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: getazservice
  namespace: topology-demo-ns
  annotations:
    service.kubernetes.io/topology-aware-hints: auto
spec:
  selector:
    app: getaz
  ports:
    - port: 80
      targetPort: web-port
      protocol: TCP
```

When we describe the Service, we see the `Annotation` associated with it.

```bash
kubectl -n topology-demo-ns describe svc getazservice 
Name:              getazservice
Namespace:         topology-demo-ns
Labels:            <none>
Annotations:       service.kubernetes.io/topology-aware-hints: auto
Selector:          app=getaz
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.100.9.165
IPs:               10.100.9.165
Port:              <unset>  80/TCP
TargetPort:        web-port/TCP
Endpoints:         192.168.24.149:3000,192.168.63.154:3000,192.168.69.57:3000
Session Affinity:  None
Events:            <none>
```

If we run the same test as before with the debug container, this time we should see an output similar to below

```bash
us-west-2b---
us-west-2b---
us-west-2b---
us-west-2b---
us-west-2b---
```

This shows that the calls to "getazservice" is getting consistently picked up by the backing Pod that resides in the same AZ as the requester Pod. Topology aware routing in this case is enabled by `EndPointSlice` Controller and the `kube-proxy` components. `EndPointSlice` API in Kubernetes provides a way to track network endpoints within a cluster. `EndpointSlices` offer a more scalable and extensible alternative to `Endpoints` and is available since Kubernetes 1.21. When calculating the endpoints for a `Service` that's annotated with `service.kubernetes.io/topology-aware-hints: auto` , the `EndpointSlice` controller considers the topology (region and zone) of each `Service` endpoint and populates the `hints` field to allocate it to a zone. Once the "hints" are populated, `kube-proxy` can then consume these hints, and use them to influence how the traffic is routed (favoring topologically closer endpoints).

This solution reduces inter-AZ traffic routing and in turn lowers the cross-AZ data transfer costs in an EKS cluster. By enabling "intelligent" routing, it also helps reduce the network latency. While this approach works well in most cases, sometimes the `EndpointSlice` controller allocates endpoints from a different zone to ensure more even distribution of endpoints between zones. This results in some traffic being routed to other zones. Thus, when using Topology-Aware-hints, its important to have application pods balanced across AZs using Topology Spread Constraints to avoid imbalances in the amount of traffic handled by each pod. Additionally, there are some other [safeguards and constraints](https://kubernetes.io/docs/concepts/services-networking/topology-aware-hints/#safeguards) that one should be aware of before using this approach. As alternative solutions, one can use Service Mesh technologies like Istio or Linkerd to achieve topology-aware routing; however service mesh based solutions present additional complexities for the cluster operators to manage. In comparison, using topology-aware-hints is much simpler to implement, is supported out-of-the-box in EKS 1.24 and works great in reducing cross-AZ traffic costs within an EKS cluster.