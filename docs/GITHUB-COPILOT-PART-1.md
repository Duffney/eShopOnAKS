---
short_title: Using Copilot to Lower the Kubernetes Learning Curve
description: Discover how to use GitHub Copilot to quickly write Dockerfiles and Kubernetes manifests for your application.
type: workshop
authors: Josh Duffney
contacts: '@joshduffney'
banner_url: 
duration_minutes: 30
audience: devs, devops, sre
level: intermediate
tags: github copilot, kubernetes, azure, docker, aks 
published: false
wt_id: 
sections_title:
  - Introduction
---

# Using Copilot to Lower the Kubernetes Learning Curve

In this workshop, you will use GitHub Copilot to quickly write Dockerfiles and Kubernetes manifests the eShopOnWeb application. You will then deploy the application to Azure Kubernetes Service (AKS).

eShopOnWeb is a sample application that demonstrates how to build a cloud-native, microservices-based, and containerized application. The application is a fictitious online store that sells various types of products. The application is built using ASP.NET Core and is hosted on Azure Kubernetes Service (AKS). The application is composed of several microservices that are deployed as Docker containers to the AKS cluster. 

## Objectives

In this workshop, you will:
- Use GitHub Copilot to write Dockerfiles and Kubernetes manifests for the eShopOnWeb application.
- Write a Kubernetes kustomization file to deploy the application to AKS.
- Deploy the application to AKS.

## Prerequisites

| | |
|----------------------|------------------------------------------------------|
| GitHub account       | [Get a free GitHub account](https://github.com/join) |
| Azure account        | [Get a free Azure account](https://azure.microsoft.com/free) |
| Azure CLI            | [Install the Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) |
| Docker Desktop       | [Install Docker](https://docs.docker.com/get-docker/) |
| kubectl              | [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) |

Before you begin, you will need an Azure account with the following resources: a resource group, an Azure Container Registry, and an Azure Kubernetes Service cluster attached to the container registry. 

You can use the following commands to create these resources:

<details>
<summary>Deploy Azure Environment</summary>

```bash
rg=<resource group name>

az group create --name $rg --location eastus

az acr create --resource-group $rg --name ${rg}acr --sku Basic

az aks create --resource-group $rg --name ${rg}aks --location eastus --attach-acr ${rg}acr --generate-ssh-keys

az aks get-credentials --resource-group $rg --name ${rg}aks --overwrite-existing
```

</details>

---

## Creating the Dockerfile

The Dockerfile is a text file that contains all the commands a user could call on the command line to assemble an image. Using docker build users can create an automated build that executes several command-line instructions in succession. 

### Writing the Dockerfile for Web App

Create an empty file `Dockerfile` in the `src/Web` directory.

```bash
touch src/Web/Dockerfile
```

Use comments to create a multi-stage Dockerfile for the Web project:

```Dockerfile
# Build stage
# Use an .NET dsk 7.0 runtime as build image

# Set the working directory to /app

# Copy the current directory contents into the container at /app

# Set the working dir to /app/src/Web

# Restore packages

# Publish a release build to /app/out

# Runtime stage
# Use aspnet 7.0 as runtime image

# Set the working directory to /app

# Copy the published app from the build stage

# Run the app
```


<div class="tip" data-title="tip">

> Place your cursor at the beginning of each blank line and allow GitHub Copilot to suggest the appropriate command.

</div>

<details>
<summary>Example Copilot suggestions</summary>

```dockerfile
# Build stage
# Use a .net sdk 7.0 image as build
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
# Set the working directory to /app
WORKDIR /app
# Copy the current directory contents into the container at /app
COPY . .
# Set the working directory to /app/src/Web
WORKDIR /app/src/Web
# Restore packages
RUN dotnet restore
# Publish a release build to the /app/out directory
RUN dotnet publish -c Release -o out

# Runtime stage
# Use aspnet 7.0 as runtime image
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime

# Set the working directory to /app
WORKDIR /app
# Copy the published app from the build image
COPY --from=build /app/src/Web/out .
# Run the app
ENTRYPOINT ["dotnet", "Web.dll"]
```

</details>


### Writing the Dockerfile for API

Create an empty file `Dockerfile` in `src/PublicApi`.

```bash
touch src/PublicApi/Dockerfile
```

Use comments to create a multi-stage Dockerfile for the PublicApi project:

```dockerfile
# Build stage
# Use an .NET dsk 7.0 runtime as build image
# Set the working directory to /app
# Copy the current directory contents into the container at /app
# Set the working directory to /app/src/PublicApi
# Restore packages
# Publish a release build to the /app/publish directory

# Runtime stage
# Use aspnet 7.0 AS runtime
# Expose port 80
# Expose port 443
# Set the working directory to /app
# Copy the published app from the build image
# Run the application when the container launches
```

<div class="tip" data-title="tip">

> Place your cursor at the end of each comment block and allow GitHub Copilot to suggest the appropriate command. Hit return to accept the suggestion and to move to the next line. If needed, begin to type the command and allow Copilot to suggest the rest.

</div>

<details>
<summary>Example Copilot suggestions</summary>

```dockerfile
# Build stage
# Use an .NET dsk 7.0 runtime as build image
# Set the working directory to /app
# Copy the current directory contents into the container at /app
# Set the working directory to /app/src/PublicApi
# Restore packages
# Publish a release build to the /app/publish directory
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /app
COPY . .
WORKDIR /app/src/PublicApi
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish PublicApi.csproj

# Runtime stage
# Use aspnet 7.0 AS runtime
# Expose port 80
# Expose port 443
# Set the working directory to /app
# Copy the published app from the build image
# Run the application when the container launches
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime
EXPOSE 80
EXPOSE 443
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "PublicApi.dll"]
```

</details>

### Building and Pushing the Container Images

Use the following steps to build and push the container images to the Azure Container Registry:

```bash
ACR_NAME=youracrname
docker build . -f src/Web/Dockerfile -t $ACR_NAME.azurecr.io/web:v1.0.0
docker build . -f src/PublicApi/Dockerfile -t $ACR_NAME.azurecr.io/api:v1.0.0
```

```bash
docker push $ACR_NAME.azurecr.io/web:v1.0.0
docker push $ACR_NAME.azurecr.io/api:v1.0.0
```

If necessary, login to the Azure Container Registry with the `az acr login` command.

---

## Writing the Kubernetes Configurations

Kubernetes manifests are YAML files that describe the desired state of your application and how to deploy it. In the following steps, you will create a deployment manifest for the eShopWeb appliaction by writing a manifest for the database, web and publicApi projects.

### Creating the Database Deployment Manifest

Create a `manifests` directory in the root of the project.

```bash
mkdir manifests
```

Under the `manifests` directory, create the `deployment-db.yaml` file.

```bash
touch manifests/deployment-db.yaml
```

Use comments to create a deployment manifest for the database:

```yaml
# define a deployment using the `mcr.microsoft.com/mssql/server:2019-latest` image 
# metadata: name db label app: db

---
# Create a service
# spec type: ClusterIP selector db protocol: TCP
# port: 1433 targetPort: 1433
```

Replace generated value of the `SA_PASSWORD` with `@someThingComplicated1234`.

<div class="tip" data-title="tip">

> Place your cursor at the end of each comment block and allow GitHub Copilot to suggest the appropriate command. Hit return to accept the suggestion and to move to the next line. If needed, begin to type the command and allow Copilot to suggest the rest.

</div>

<details>
<summary>Example Copilot suggestions</summary>

```yaml
# define a deployment using the `mcr.microsoft.com/mssql/server:2019-latest` image 
# metadata: name db label app: db
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: mcr.microsoft.com/mssql/server:2019-latest
        ports:
        - containerPort: 1433
        env:
        - name: ACCEPT_EULA
          value: "Y"
        - name: SA_PASSWORD
          value: "@someThingComplicated1234"
        - name: MSSQL_PID
          value: "Developer"
---
# Create a service
# spec type: ClusterIP selector db protocol: TCP
# port: 1433 targetPort: 1433
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  type: ClusterIP
  selector:
    app: db
  ports:
  - protocol: TCP
    port: 1433
    targetPort: 1433
```

</details>

### Creating the Web Deployment Manifest

Under the `manifests` directory, create the `deployment-web.yaml` file.

```bash
touch manifests/deployment-web.yaml
```

Use comments to create a deployment manifest for the Web project:

```yaml
# Define a deployment using notavalidregistry.azurecr.io/web:v1.0.0
# metadata: name web label app: web
# env: ASPNETCORE_ENVIRONMENT ASPNETCORE_URLS ConnectionStrings__CatalogConnection ConnectionStrings__IdentityConnection
# ports: containerPort: 80
# resources: {}

---
#  ClusterIP for the web app on port 80 with protocol TCP
```

<details>
<summary>Example Copilot suggestions</summary>

```yaml
# Define a deployment using notavalidregistry.azurecr.io/web:v1.0.0
# metadata: name web label app: web
# env: ASPNETCORE_ENVIRONMENT ASPNETCORE_URLS ConnectionStrings__CatalogConnection ConnectionStrings__IdentityConnection
# ports: containerPort: 80
# resources: {}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: notavalidregistry.azurecr.io/web:v1.0.0
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Development"
        - name: ASPNETCORE_URLS
          value: "http://+:80"
        - name: ConnectionStrings__CatalogConnection
          value: Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.CatalogDb;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True; 
        - name: ConnectionStrings__IdentityConnection
          value: Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.Identity;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True;
        ports:
        - containerPort: 80
        resources: {}
---
#  ClusterIP for the web app on port 80 with protocol TCP
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web
```

</details>

Update the environment variable with the appropriate values:

| Environment Variable | Value |
| --- | --- |
| ASPNETCORE_ENVIRONMENT | Development |
| ASPNETCORE_URLS | http://+:80 |
| ConnectionStrings__CatalogConnection | Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.CatalogDb;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True; |
| ConnectionStrings__IdentityConnection | Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.Identity;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True; |

### Creating the PublicApi Deployment Manifest

Under the `manifests` directory, create the `deployment-api.yaml` file.

```bash
touch manifests/deployment-api.yaml
```

Use comments to create a deployment manifest for the PublicApi project:

```yaml
# Define a deployment using notavalidregistry.azurecr.io/api:v1.0.0

---
# ClusterIP for the api app on port 80 no labels
```

<details>
<summary>Example Copilot suggestions</summary>

```yaml
# Define a deployment using notavalidregistry.azurecr.io/api:v1.0.0
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: notavalidregistry.azurecr.io/api:v1.0.0
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Development"
        - name: ASPNETCORE_URLS
          value: "http://+:80"
        - name: ConnectionStrings__CatalogConnection
          value: Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.CatalogDb;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True;
        - name: ConnectionStrings__IdentityConnection
          value: Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.Identity;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True;
        ports:
        - containerPort: 80
        resources: {}
---
# ClusterIP for the api app on port 80 no labels
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: api
```

</details>

---

## Deploying to AKS

Now, that you have the Kubernetes manifests, you can deploy the application to AKS. But first, you have to create a kutomization file that will be used to deploy the application.

Create a file `./manifests/kustomize.yml` with the following content:

```bash
touch manifests/kustomization.yaml
```

Type out the contents of the kustomize file as shown below and accept the suggestions from Copilot. Replace the `newName` and `newTag` values with the appropriate values from your ACR instance.

<details>
<summary>Example Copilot suggestions</summary>

```yaml
resources:
    - deployment-web.yaml
    - deployment-api.yaml

images:
- name: notavalidregistry.azurecr.io/api:v1.0.0
newName: <YOUR_ACR_SERVER>/web
newTag: <YOUR_IMAGE_TAG>
- name: notavalidregistry.azurecr.io/web:v1.0.0
newName: <YOUR_ACR_SERVER>/api
newTag: <YOUR_IMAGE_TAG>
```

</details>

Deploy the application to AKS:

```bash
# Deploy the database
kubectl apply -f manifests/deployment-db.yaml

# Deploy the application with kustomize
kubectl apply -k manifests/
```

Port forward the application to your local machine:

```bash
kubectl port-forward deployment/web 8080:80
```

Open a browser and navigate to `http://localhost:8080`. You should see the application running.