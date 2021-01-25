# RabbitMQ 

There are multiple ways to install RabbitMQ on Kubernetes, and RabbitMQ Operator is one of them. RabbitMQ Operator for Kubernetes becomes GA in November 2020 that can provision single-node or multiple-node clusters on top of Kubernetes. Thanks to the nature of operator that can automatically reconcile the deployed clusters with desired state defined by user, Day 2 operation of RabbitMQ clusters has never been easier. Especially, upgrading nodes(ex. Kubernetes version, OS patch and etc) online using operator can greatly reduce the toil compared to doing the same on traditional Virtual Machine based cluster. 

RabbitMQ Operator uses "RabbitmqCluster" Custom Resource Definition to define the state of cluster, and it provides many examples out of the box. https://github.com/rabbitmq/cluster-operator/tree/main/docs/examples


## Installing RabbitMQ Operator

RabbitMQ Operator can be installed in two different ways, either by using yaml or using kubernetes plugin. Here, we will take yaml approach. If you're intersted in kubectl plug-in, you can find the link. 


```bash
kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"
```

It creates the namespace 'rabbitmq-system' and install all the necessary resosources including operator.  

```
$ kubectl get deploy -n rabbitmq-system
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq-cluster-operator   1/1     1            1           5d2h
```

## Provision Multiple-node RabbitMQ Cluster

We will create 3 nodes cluster for production that requires bit of more setup as seen below, #1 and #2. To minimize the impact of upgrade which requires the reboot of the underlying virtual machine, We will #1 use topologySpreadConstraints to evenly distribute the pods across zone, and #2 NodeAffinity to achieve blue green deployment by switching nodepool. We will discuss in detail about switching nodepool later. 


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

## Upgrading Kubernetes version
