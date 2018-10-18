# Azure Kubernetes Service (AKS)

## Azure requirements

See [AKS documentation](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest)

    az provider register --name Microsoft.Network --wait
    az provider register --name Microsoft.Compute --wait
    az provider register --name Microsoft.Storage --wait
    az provider register --name Microsoft.ContainerService --wait

Install **kubectl**

    az aks install-cli

You may need to add kubectl in your PATH:

    PATH=/usr/local/Cellar/kubernetes-cli/1.11.1/bin:$PATH

## Create your AKS cluster

    export CLUSTER_NAME="TestCluster"
    export RESOURCE_GROUP_NAME="aks1"

Create a new resource group:

    az group create -l westeurope -n $RESOURCE_GROUP_NAME
    
Create a Kubernetes cluster:

    az aks create --name $CLUSTER_NAME \
                  --resource-group $RESOURCE_GROUP_NAME \
                  --ssh-key-value ~/.ssh/id_rsa.pub \
                  --node-count 3 \
                  --node-vm-size Standard_D2s_v3 \
                  --kubernetes-version 1.11.3 \
                  --output table

Register k8s credential and use it as default

    az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP_NAME --output table

Set your default k8s context:

    kubectl config use-context $CLUSTER_NAME

## Show cluster resources:

    az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP_NAME --output table

View kubernetes dashboard:

    az aks browse --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME

