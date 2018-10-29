# node-helloworld
Sample app for demos

## Setup a default registry

```sh
export ACR_NAME=demo42
az configure --defaults acr=$ACR_NAME
```
## Clone the repo
```sh
git clone github.com/
```

## Local Build
```sh
# Build
az acr build -t helloworld:{{.Build.ID}} . 
#List Images
az acr repository show-tags --repository helloworld
```
 Common Environment Variables
```sh
# Replace these values for your configuration
# I've left our values in, as we use this for our demos, providing some examples
export ACR_NAME=demo42
export RESOURCE_GROUP=$ACR_NAME
# fully qualified url of the registry. 
# This is where your registry would be
# Accounts for registries in dogfood or other clouds like .gov, Germany and China
export REGISTRY_NAME=${ACR_NAME}.azurecr.io/ 
export AKV_NAME=$ACR_NAME # name of the keyvault
export GIT_TOKEN_NAME=demo42-git-token # keyvault secret name
```

## Create an ACR Task
- Populate your GIT Personal Access Token
  ```sh
  export PAT=$(az keyvault secret show \
                --vault-name $AKV_NAME \
                --name $GIT_TOKEN_NAME \
                --query value -o tsv)
  ```
- Create the task
  ```sh
az acr task create \
  -n helloworld-multistep \
  -t helloworld:{{.Run.ID}} \
  -t helloworld:latest \
  -f acr-task.yaml \
  --arg REGISTRY_NAME=$REGISTRY_NAME/ \
  --context https://github.com/demo42/helloworld.git \
  --git-access-token $PAT \
  --set CLUSTER_NAME=demo42-staging-eus \
  --set CLUSTER_RESOURCE_GROUP=demo42-staging-eus \
  --set-secret SP=$(az keyvault secret show \
            --vault-name ${AKV_NAME} \
            --name demo42-serviceaccount-user \
            --query value -o tsv) \
  --set-secret PASSWORD=$(az keyvault secret show \
            --vault-name ${AKV_NAME} \
            --name demo42-serviceaccount-pwd \
            --query value -o tsv) \
  --set-secret TENANT=$(az keyvault secret show \
            --vault-name ${AKV_NAME} \
            --name demo42-serviceaccount-tenant \
            --query value -o tsv) \
  --registry $ACR_NAME 
```

Run a quick-task
```sh
az acr run -f acr-task.yaml .
```
- Commit a code change
  
  Monitor the current builds
  ```sh
  watch -n1 az acr build-task list-builds 
  ```

- View the current executing builds

  ```sh
  az acr build-task logs
  ```

## Base Image Updates

- Update the base image

  ```sh
  docker build -t baseimages/node:9 \
    -f node-jessie.Dockerfile \
    .
  ```
- Switch the dockerfile to -alpine

  Update the base image for Apline
  ```sh
  docker build -t jengademos.azurecr.io/baseimages/node:9-alpine -f node-alpine.Dockerfile .
  docker push jengademos.azurecr.io/baseimages/node:9-alpine
  ```

## Update Demo42 Backcolor
  ```sh
  docker tag jengademos.azurecr.io/baseimages/microsoft/aspnetcore-runtime:linux-2.1-azure \
    jengademos.azurecr.io/baseimages/microsoft/aspnetcore-runtime:linux-2.1
  docker push \
    jengademos.azurecr.io/baseimages/microsoft/aspnetcore-runtime:linux-2.1
  ```
## Deploy to AKS

- Get the cluster you're working with
  ```sh
  az aks list
  ```

- get credentials for the cluster

  ```sh
  az aks get-credentials -n [name] -g [group]
  ```
- Set vaiables

  ```sh
  export HOST=http://demo42-helloworld.eastus.cloudapp.azure.com/
  export ACR_NAME=jengademos
  export TAG=aa42
  export AKV_NAME=jengademoskv
  ```

- Deploy with Helm

  Set the 
  ```sh
  helm install ./release/helm/ -n helloworld \
  --set helloworld.host=$HOST \
  --set helloworld.image=jengademos.azurecr.io/demo42/helloworld:$TAG \
  --set imageCredentials.registry=$ACR_NAME.azurecr.io \
  --set imageCredentials.username=$(az keyvault secret show \
                                         --vault-name $AKV_NAME \
                                         --name $ACR_NAME-pull-usr \
                                         --query value -o tsv) \
  --set imageCredentials.password=$(az keyvault secret show \
                                         --vault-name $AKV_NAME \
                                         --name $ACR_NAME-pull-pwd \
                                         --query value -o tsv)
```
## Helm Package, push
helm package \
    --version 1.0.1 \
    ./helm/helloworld

az acr helm push \
    ./helloworld-1.0.1.tgz \
    --force -o table

## Update the local cache
az acr helm repo add

helm fetch demo42/helloworld

helm repo list

## Upgrade
```sh
helm upgrade helloworld ./helm/helloworld/ \
  --reuse-values \
  --set helloworld.image=demo42.azurecr.io/helloworld:$TAG
```
## Create the webhook header
  Create a value in keyvault to save for future reference
  ```sh
  az keyvault secret set \
    --vault-name $AKV_NAME \
    --name demo42-helloworld-webhook-auth-header \
    --value "Authorization: Bearer "[value]
  ```

## Create ACR Webhook for deployments
  ```sh
  az acr webhook create \
    -r $ACR_NAME \
    --scope demo42/helloworld:* \
    --actions push \
    --name demo42HelloworldEastus \
    --headers Authorization=$(az keyvault secret show \
                              --vault-name $AKV_NAME \
                              --name demo42-helloworld-webhook-auth-header \
                              --query value -o tsv) \
    --uri http://jengajenkins.eastus.cloudapp.azure.com/jenkins/generic-webhook-trigger/invoke
  ```
  