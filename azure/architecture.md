# Azure Architecture for Demo

``` mermaid
graph TD
    subgraph VNet["Azure VNet (10.100.0.0/16)"]
        PublicSubnet["Public Subnet (10.100.1.0/24)"]
        PrivateSubnet["Private Subnet (10.100.2.0/24)"] 
    end
```

## code for network

```sh
# Variables
# Variables
RESOURCE_GROUP="cloud-coe-nogada-rg"
LOCATION="koreacentral"
SUBSCRIPTION="CloudBiz"
VNET_NAME="NogadaVNet"
VNET_CIDR="10.100.0.0/16"
PUBLIC_SUBNET_NAME="PublicSubnet"
PUBLIC_SUBNET_CIDR="10.100.1.0/24"
PRIVATE_SUBNET_NAME="PrivateSubnet"
PRIVATE_SUBNET_CIDR="10.100.2.0/24"

# Set the Subscription
az account set --subscription "$SUBSCRIPTION"

# Create a Resource Group
# az group create --name $RESOURCE_GROUP --location $LOCATION

# Create a Virtual Network
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --address-prefix $VNET_CIDR \
  --location $LOCATION

# Create the Public Subnet
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $PUBLIC_SUBNET_NAME \
  --address-prefix $PUBLIC_SUBNET_CIDR

# Create the Private Subnet
# az network vnet subnet create \
#   --resource-group $RESOURCE_GROUP \
#   --vnet-name $VNET_NAME \
#   --name $PRIVATE_SUBNET_NAME \
#   --address-prefix $PRIVATE_SUBNET_CIDR
```


## code for AKS
> locate aks in subnet 
>

### Env
```sh
export RANDOM_ID="$(openssl rand -hex 3)"
export RESOURCE_GROUP="cloud-coe-nogada-rg"
export REGION="koreacentral"
export AKS_CLUSTER_NAME="nogada-aks"
export MY_DNS_LABEL="nogadadnslabel"
```

### install aks

```sh
PUBLIC_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $PUBLIC_SUBNET_NAME \
  --query id --output tsv)


# Export variables
export RESOURCE_GROUP="cloud-coe-nogada-rg"
export REGION="koreacentral"
export AKS_CLUSTER_NAME="nogada-aks"
export MY_DNS_LABEL="nogadadnslabel"
export VNET_NAME="NogadaVNet"
export PUBLIC_SUBNET_NAME="PublicSubnet"
export SSH_KEY_PATH="~/.ssh/ds-nogada.pub"
export INGRESS_STATIC_IP_NAME="nogada-ip"

# Fetch the subnet ID for the Public Subnet
PUBLIC_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $PUBLIC_SUBNET_NAME \
  --query id --output tsv)

# Create the AKS cluster
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --location $REGION \
  --dns-name-prefix $MY_DNS_LABEL \
  --network-plugin kubenet \
  --vnet-subnet-id $PUBLIC_SUBNET_ID \
  --node-vm-size Standard_D4s_v3 \
  --node-count 1 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --kubernetes-version "1.30.6" \
  --no-wait


az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME \
    --admin \
    --overwrite-existing
```

### Ingress setting

```sh
# Create Static IP
az network public-ip create --resource-group $RESOURCE_GROUP --name $INGRESS_STATIC_IP_NAME --sku Standard --allocation-method static


# Role binding
CLIENT_ID=$(az aks show --name $AKS_CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.principalId -o tsv)
RG_SCOPE=$(az group show --name $RESOURCE_GROUP --query id -o tsv)
az role assignment create --assignee ${CLIENT_ID} --role "Network Contributor" --scope ${RG_SCOPE}

# set ingress controller

az aks approuting enable --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME 
az aks approuting update --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --nginx AnnotationControlled

kubectl apply -f - <<EOF
apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: NginxIngressController
metadata:
  name: nginx-static
spec:
  ingressClassName: nginx-static
  controllerNamePrefix: nginx-static
  loadBalancerAnnotations: 
    service.beta.kubernetes.io/azure-pip-name: $INGRESS_STATIC_IP_NAME
    service.beta.kubernetes.io/azure-load-balancer-resource-group: $RESOURCE_GROUP
EOF
```

```sh

# Test
kubectl create ns sam

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sam-app
  namespace: sam
spec:
  ingressClassName: nginx-static
  rules:
  - host: sam.nogada.kubepia.net
    http:
      paths:
      - backend:
          service:
            name: aks-helloworld
            port:
              number: 80
        path: /
        pathType: Prefix
EOF

```

### Node Pool 구성

```sh

az aks nodepool add --resource-group $RESOURCE_GROUP --cluster-name $AKS_CLUSTER_NAME --name mypool --node-count 3 --node-vm-size Standard_D4as_v4 --mode User --no-wait

```

### Storage 구성(nfs, disk)

```sh


```

### Setup Prometheus
[github_ref](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)


```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update


```
