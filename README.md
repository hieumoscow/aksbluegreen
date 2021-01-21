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
