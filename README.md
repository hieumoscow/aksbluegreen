# AKS Upgrade
This article intents to enumarate different upgrade strategies that will help to maintain AKS and ensure the clusters are complaint with [AKS support policies](https://docs.microsoft.com/en-us/azure/aks/support-policies).

## Upgrade Strategies


|                           | In-place Upgrade | Blue/Green (New Node Pool) | Blue/Green (New Cluster) |
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
| **How**       | AKS Upgrade API | New Node Pool <br /> Nodes Labels and Selectors| Automation + GLB (Traffic Manager) |
| **Upgrade Duration** | Depends on Cluster Size & surge value <br /> The larger the cluster the longer the upgrade | Shortest (Node Labels) Short with the right automation | Shortest (Node Labels) |
| **Risk** | Subject to Failure | Safe - with the risk that the control plane upgrade will fail <br /> No impact on applications | Safest |
| **When to recommend** | Small Production Cluster <br /> Non-Mission Critical <br /> Workloads <br /> Non-Prod Environments | Large Clusters <br /> Mission Critical Workloads | Large Clusters <br /> Mission Critical Workloads 

# Blue Green Node Pool upgrade for AKS

```bash
PREFIX=aksbluegreen1
RG=$PREFIX-rg
CLUSTER=$PREFIX-aks
VNET=$PREFIX-vnet
LOC=southeastasia
BLUE_VERSION=1.18.14
GREEN_VERSION=1.19.6
VERSION=$BLUE_VERSION

az group create -n $RG -l $LOC

az network vnet create -g $RG -n $VNET -l $LOC --address-prefixes 10.100.0.0/22

az network vnet subnet create -g $RG --vnet-name $VNET --name system --address-prefixes 10.100.0.0/24

az network vnet subnet create -g $RG --vnet-name $VNET --name user-blue --address-prefixes 10.100.1.0/24

az network vnet subnet create -g $RG --vnet-name $VNET --name user-green --address-prefixes 10.100.2.0/24

SYSTEM_SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name $VNET --name system --query id -o tsv)

az aks create -g $RG -n $CLUSTER -l $LOC \
--max-pod 30 --generate-ssh-keys --network-plugin azure \
--enable-managed-identity -c 1 \
--vnet-subnet-id $SYSTEM_SUBNET_ID -k $BLUE_VERSION

BLUE_SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name $VNET --name user-blue --query id -o tsv)

az aks nodepool add -g $RG -n bluepool \
-k $BLUE_VERSION --cluster-name $CLUSTER --vnet-subnet-id $BLUE_SUBNET_ID

az aks get-credentials -g $RG -n $CLUSTER -admin

az aks upgrade -g $RG -n $CLUSTER --control-plane-only -k $GREEN_VERSION

GREEN_SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name $VNET --name user-blue --query id -o tsv)

az aks nodepool add -g $RG -n greenpool \
-k $GREEN_VERSION --cluster-name $CLUSTER --vnet-subnet-id $GREEN_SUBNET_ID

```


## Before We Get Started

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  labels:
    addonmanager.kubernetes.io/mode: EnsureExists
    kubernetes.io/cluster-service: "true"
  name: default
parameters:
  cachingmode: ReadOnly
  kind: Managed
  storageaccounttype: StandardSSD_LRS
provisioner: kubernetes.io/azure-disk
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```


## RabbitMQ 

There have been multiple ways to install RabbitMQ on Kubernetes, and RabbitMQ Operator is one of them. RabbitMQ Operator for Kubernetes becomes GA in November 2020 that can provision single-node or multiple-node clusters on top of Kubernetes. Thanks to the nature of operator that can automatically reconcile the deployed clusters with desired state defined by user, Day 2 operation of RabbitMQ clusters has never been easier. Especially, upgrading nodes(ex. Kubernetes version, OS patch and etc) online using operator will greatly reduce the toil compared to doing the same on traditional Virtual Machine based cluster. 

RabbitMQ Operator uses "RabbitmqCluster" Custom Resource Definition to define the state of cluster, and it provides many examples out of the box. https://github.com/rabbitmq/cluster-operator/tree/main/docs/examples


### Installing RabbitMQ Operator

RabbitMQ Operator can be installed in two ways, using yaml or using kubernetes plugin. Here, we will take yaml approach. If you're intersted in kubectl plug-in, you can find the link. 


```bash
kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"
```

### Provision Multiple-node RabbitMQ Cluster

We will create 3 nodes cluster for production. Setup is based on the example provided by operator for production. Look at #1, it's using topologySpreadConstraints to evenly distribute the pods across zone. #2, NodeAffinity is used to achieve blue green deployment using nodepool. 


```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-cluster
spec:
  replicas: 3
  service:
    type: LoadBalancer
  resources:
    requests:
      cpu: 4
      memory: 10Gi
    limits:
      cpu: 4
      memory: 10Gi
  rabbitmq:
    additionalConfig: |
      cluster_partition_handling = pause_minority
      vm_memory_high_watermark_paging_ratio = 0.99
      disk_free_limit.relative = 1.0
  persistence:
    storageClassName: default
    storage: "200Gi"
  override:
    statefulSet:
      spec:
        template:
          spec:
            containers: []
            topologySpreadConstraints:  # 1. topologySpreadConstraints for better pod distribution
            - maxSkew: 1
              topologyKey: "topology.kubernetes.io/zone"
              whenUnsatisfiable: DoNotSchedule
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: rabbitmq-cluster
            affinity:    # 2. nodeAffinity for userpool based blue green deployment
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: agentpool
                      operator: In
                      values:
                      - userpool2
```


### Upgrading Kubernetes version



