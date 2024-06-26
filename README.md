# AKS Static Egress with private egress IPs

Simple demonstration of static egress using the open-source `kube-egress-gateway` project. The configuration used is to use private egress IPs from a small subnet in the same VNET as the AKS cluster.
Pods can still access the Internet via default Azure networking route or static private egress IP(s) when accessing specific private RFC-1918 destinations (e.g. this could be a on-prem destination). This would allow predictable and a more narrow set of private IPs for egress traffic from the AKS cluster for use in firewall rules to allow traffic to on-premises or other private destinations. Prevously, you would need to whitelist the whole AKS subnet CIDR range.  The static egress feature allows you to opt-in workloads to use static egress IPs for specific destinations.

This demo is using an AKS cluster and a target network (another VNET) with a container instance deployed. The AKS VNET and target VNET are peered.  This demonstrates a simplified scenario but could be extended to a real on-premises network or other private destinations.

## Create the AKS cluster with a dedicated VMSS node pool for the egress gateway

```sh
az login
az account set --subscription "your-subscription-id"

# Configure these to suit your environment:
LOCATION=australiasoutheast
CLUSTER=egress-demo
RG_NAME=egress-demo
EGRESS_NODE_POOL=egressgw
DEFAULT_SUBNET_NAME=default
EGRESS_SUBNET_NAME=egress
VNET_NAME=$CLUSTER-vnet
VNET_PREFIX=10.224.0.0/12
DEFAULT_SUBNET_PREFIX=10.224.0.0/16
EGRESS_SUBNET_PREFIX=10.225.0.0/28
POD_CIDR=192.168.0.0/16

az group create -n $RG_NAME -l $LOCATION

az network vnet create -g $RG_NAME -n $VNET_NAME --address-prefixes $VNET_PREFIX --subnet-name $DEFAULT_SUBNET_NAME --subnet-prefixes $DEFAULT_SUBNET_PREFIX
DEFAULT_SUBNET_ID=$(az network vnet subnet show -g $RG_NAME --vnet-name $VNET_NAME --name $DEFAULT_SUBNET_NAME --query id -o tsv)

# Add a second subnet for the egress IPs (network is 10.224.0.0/12)
# 10.225.0.0/28 is 14 addresses after the first subnet 10.224.0.0/16.
az network vnet subnet create -g $RG_NAME --vnet-name $VNET_NAME -n $EGRESS_SUBNET_NAME --address-prefixes $EGRESS_SUBNET_PREFIX
EGRESS_SUBNET_ID=$(az network vnet subnet show -g $RG_NAME --vnet-name $VNET_NAME -n $EGRESS_SUBNET_NAME --query id -o tsv)

# Create an AKS cluster with default VNET and subnet
az aks create \
    -g $RG_NAME \
    -n $CLUSTER \
    --node-count 2 \
    --enable-managed-identity \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --outbound-type loadBalancer \
    --pod-cidr $POD_CIDR \
    --vnet-subnet-id $DEFAULT_SUBNET_ID \
    --generate-ssh-keys

# Add the egress gateway node pool, using the egress subnet
az aks nodepool add \
    -g $RG_NAME \
    --cluster-name $CLUSTER \
    -n $EGRESS_NODE_POOL \
    --node-count 2 \
    --node-taints "kubeegressgateway.azure.com/mode=true:NoSchedule" \
    --labels "kubeegressgateway.azure.com/mode=true" \
    --os-type Linux \
    --vnet-subnet-id $EGRESS_SUBNET_ID

az aks get-credentials -g $RG_NAME -n $CLUSTER
az aks install-cli

kubectl get nodes
```

## Create User-Managed Identity for the egress gateway controller

```sh
EGRESS_IDENTITY_NAME="staticegress-msi"
az identity create -g $RG_NAME -n $EGRESS_IDENTITY_NAME

IDENTITY_CLIENT_ID=$(az identity show -g $RG_NAME -n $EGRESS_IDENTITY_NAME -o tsv --query "clientId")
IDENTITY_RESOURCE_ID=$(az identity show -g $RG_NAME -n $EGRESS_IDENTITY_NAME -o tsv --query "id")

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
NODE_RESOURCE_GROUP="$(az aks show -n $CLUSTER -g $RG_NAME --query nodeResourceGroup -o tsv)"
EGRESS_VMSS_NAME=$(az vmss list -g $NODE_RESOURCE_GROUP --query [].name -o tsv | grep $EGRESS_NODE_POOL)
SYSTEM_VMSS_NAME=$(az vmss list -g $NODE_RESOURCE_GROUP --query [].name -o tsv | grep -v $EGRESS_NODE_POOL)

RG_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME"
NODE_RG_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$NODE_RESOURCE_GROUP"
EGRESS_VMSS_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$NODE_RESOURCE_GROUP/providers/Microsoft.Compute/virtualMachineScaleSets/$EGRESS_VMSS_NAME"

# Assign Network Contributor role on scope vmss resource group and vnet resource group to the identity
az role assignment create --role "Network Contributor" --assignee $IDENTITY_CLIENT_ID --scope $RG_ID
az role assignment create --role "Network Contributor" --assignee $IDENTITY_CLIENT_ID --scope $NODE_RG_ID

# Assign Virtual Machine Contributor role on scope gateway vmss to the identity
az role assignment create --role "Virtual Machine Contributor" --assignee $IDENTITY_CLIENT_ID --scope $EGRESS_VMSS_ID

# Assign the user-managed identity to the egress gateway VMSS
# (This is currently required for the kube-egress-gateway controller to use the identity to assign the egress IPs but won't be
# needed with the managed AKS static egress feature when it becomes available.)
az vmss identity assign --identities $IDENTITY_RESOURCE_ID  -g $NODE_RESOURCE_GROUP -n $SYSTEM_VMSS_NAME
```

## Download the Azure Cloud Config templated and update with your details

```sh
wget https://raw.githubusercontent.com/Azure/kube-egress-gateway/main/docs/samples/sample_azure_config_msi.yaml -O azure_config.yaml

echo "Update the following values in azure_config.yaml:"
echo "tenantId: $TENANT_ID"
echo "subscriptionId: $SUBSCRIPTION_ID"
echo "location: $LOCATION"
echo "vnet name: $VNET_NAME"
echo "vnet resource group: $RG_NAME"
echo "egress vmss resource group: $NODE_RESOURCE_GROUP"
echo "egress subnet name: $EGRESS_SUBNET_NAME"
echo "userAssignedIdentityID: $IDENTITY_CLIENT_ID"
```

Edit `azure_config.yaml` and update with your values.

Or run the command as per below to create the file:

cat << EOF > azure_config.yaml
config:
  azureCloudConfig:
    cloud: "AzurePublicCloud"
    tenantId: "$TENANT_ID"
    subscriptionId: "$SUBSCRIPTION_ID"
    useManagedIdentityExtension: true
    userAssignedIdentityID: "$IDENTITY_CLIENT_ID"
    userAgent: "kube-egress-gateway-controller"
    resourceGroup: "$NODE_RESOURCE_GROUP"
    location: "$LOCATION"
    gatewayLoadBalancerName: "kubeegressgateway-ilb"
    loadBalancerResourceGroup: "$NODE_RESOURCE_GROUP"
    vnetName: "$VNET_NAME"
    vnetResourceGroup: "$RG_NAME"
    subnetName: "$EGRESS_SUBNET_NAME"
EOF

## Install Static Egress Gateway Helm Chart

```sh
git clone https://github.com/Azure/kube-egress-gateway.git

helm install \
  kube-egress-gateway ./kube-egress-gateway/helm/kube-egress-gateway \
  --namespace kube-egress-gateway-system \
  --create-namespace \
  --set common.imageRepository=mcr.microsoft.com/aks \
  --set common.imageTag=v0.0.8 \
  -f ./azure_config.yaml
```

## Deploy a container instance (internal environment) in another vnet

```sh
TARGET_VNET_NAME="my-target-vnet"

az network vnet create --resource-group $RG_NAME --name $TARGET_VNET_NAME --location $LOCATION --address-prefix 10.0.0.0/16

az container create \
  --name appcontainer \
  --resource-group $RG_NAME \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --vnet $TARGET_VNET_NAME \
  --vnet-address-prefix 10.0.0.0/16 \
  --subnet apps \
  --subnet-address-prefix 10.0.3.0/24

TARGET_IP="$(az container show -n appcontainer -g $RG_NAME --query ipAddress.ip -o tsv)"
```

## Peer both vnets

```sh
az network vnet peering create --name staticegress-to-target --resource-group $RG_NAME --vnet-name $VNET_NAME --remote-vnet $TARGET_VNET_NAME --allow-vnet-access
az network vnet peering create --name target-to-staticegress --resource-group $RG_NAME --vnet-name $TARGET_VNET_NAME --remote-vnet $VNET_NAME --allow-vnet-access
```

## Deploy a pod in the AKS cluster to test accessing the internal container instance *without* static egress

```sh
kubectl apply -f demo-ns.yaml
kubectl apply -f app-nostaticegress.yaml
POD1=$(kubectl get pods -n demo -l app=app1 -o name)
kubectl exec -ti $POD1 -n demo -- bash

echo $TARGET_IP
# 10.0.3.4

curl -i http://10.0.3.4
curl -i http://10.0.3.4
curl -i http://10.0.3.4
curl -i http://10.0.3.4
exit

az container logs --resource-group $RG_NAME --name appcontainer
# ::ffff:10.224.0.4 - - [21/Mar/2024:20:33:39 +0000] "GET / HTTP/1.1" 200 1663 "-" "curl/7.88.1"
#
# (Private IP is from the AKS subnet)
```

Note that the source IP is from the AKS subnet.

## Deploy a static egress configuration

```sh
export EGRESS_VMSS_NAME
export NODE_RESOURCE_GROUP
envsubst '$EGRESS_VMSS_NAME,$NODE_RESOURCE_GROUP' < staticGatewayConfig.yaml | kubectl apply -f -
kubectl describe StaticGatewayConfiguration myegressgateway -n demo
```

## Deploy a pod in the AKS cluster to test accessing the internal container instance *with* static egress

```sh
kubectl apply -f app-staticegress.yaml
POD2=$(kubectl get pods -n demo -l app=app2 -o name)
kubectl exec -ti $POD2 -n demo -- bash

curl -i http://10.0.3.4
curl -i http://10.0.3.4
curl -i http://10.0.3.4
curl -i http://10.0.3.4

az container logs --resource-group $RG_NAME --name appcontainer
# ::ffff:10.225.0.7 - - [21/Mar/2024:20:34:54 +0000] "GET / HTTP/1.1" 200 1663 "-" "curl/7.88.1"
# (Private IP is from the static egress subnet)
```

## Cleanup

```sh
az group delete -n $RG_NAME --yes --no-wait
```

## References

- [kube-egress-gateway](https://github.com/Azure/kube-egress-gateway) - GitHub
