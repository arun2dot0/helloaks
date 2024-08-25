# Getting Started

Simple Application to deploy to Cloud

### Build the Image
```
./gradlew clean build  
docker build -t hello-app:1.0 .  
docker run --rm -p 8080:8080  hello-app:1.0 
curl  http://localhost:8080/hello ( in other window ) 
```

### Azure steps
Create Resource Group , I am using ak-test-resource-group and location westus
```
az provider register --namespace Microsoft.ContainerService
az provider show -n Microsoft.ContainerService
az aks create \
  --resource-group ak-test-resource-group \
  --name helloAKSCluster \
  --node-count 1 \
  --enable-addons monitoring \
  --location westus \
  --generate-ssh-keys

Note : Change location if the resource group fails to create 

```

### Local setup to use Kubectl with Azure 
```commandline

brew update && brew install azure-cli
az login
az account set --subscription <your-subscription-id>
az aks get-credentials --resource-group  ak-test-resource-group  --name helloAKSCluster
```

### Push the Image to Registry

I couldn't get the image from dockerhub to work with Azure cluster . There are some examples
that use Docker hub registry with Azure , however it didn't work with my docker hub repository.

Using the Azure Container registry (ACR).

```commandline

az acr create --resource-group ak-test-resource-group  --name myacr --sku Basic

docker buildx build --platform linux/amd64,linux/arm64 -t myacr.azurecr.io/hello-app:2.0 --push .    

az acr repository list --name myacr --output table

az acr login --name myacr
az aks update -n helloAKSCluster -g ak-test-resource-group  --attach-acr myacr

 az ad sp list --display-name http://acr-service-principal   
 SP_PASSWORD=$(az ad sp create-for-rbac --name http://acr-service-principal --scopes $(az acr show --name myacr --query id --output tsv) --role acrpull --query password --output tsv)
 
 SP_APP_ID=<app id from the list>
kubectl create secret docker-registry acr-auth --docker-server=myacr.azurecr.io --docker-username=$SP_APP_ID --docker-password=$SP_PASSWORD

```

### Validate 

```commandline

az acr repository show-tags --name myacr --repository myacr.azurecr.io
docker pull myacr.azurecr.io/hello-app:1.0
az acr repository list --name myacr --output table
```


### Application is ready - Test it out

You can use Nodeport by changing service.yaml from ClusterIP to LoadBalancer

```commandline

cd deployment
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

az aks show --resource-group ak-test-resource-group --name helloAKSCluster -o table
```

There is no ingress so try out
http://externalip:nodePort/hello

#### Using Ingress 

```commandline
az aks create --resource-group ak-test-resource-group --name helloAKSCluster --location westus --enable-app-routing --generate-ssh-keys

az aks approuting enable --resource-group ak-test-resource-group --name helloAKSCluster

az aks get-credentials --resource-group ak-test-resource-group --name helloAKSCluster

kubectl apply -f ingress.yaml

Get ip address of ingress using  kubectl get ingress 
```

With ingress try out curl http://externalip/hello

### Clean up 
```commandline
az aks stop --name helloAKSCluster --resource-group ak-test-resource-group
az aks show --name helloAKSCluster --resource-group ak-test-resource-group

```

### Start again
```commandline
az aks start --name helloAKSCluster --resource-group ak-test-resource-group
az aks show --name helloAKSCluster --resource-group ak-test-resource-group

```
### Nodepool scaling  

```commandline
az aks nodepool scale \
--resource-group ak-test-resource-group \
--cluster-name helloAKSCluster \
--name nodepool1 \
--node-count 0


az aks nodepool update \
--resource-group ak-test-resource-group  \
--cluster-name helloAKSCluster \
--name myNodePool \
--enable-cluster-autoscaler \
--min-count 0 \
--max-count 3

```

