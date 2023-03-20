# Using Copilot to lower the learning curve to successfully adopt Azure Kubernetes Service

## Table of Contents
- [Dockerfiles]
- [Kubernetes Manifests]
- [Deploying to AKS]

## Creating the Dockerfile

The Dockerfile is a text file that contains all the commands a user could call on the command line to assemble an image. Using docker build users can create an automated build that executes several command-line instructions in succession.

1. Create an empty file `Dockerfile` in the `src/Web` directory.
2. Use comments to create a multi-stage Dockerfile for the Web project:

    ```Dockerfile
    # Build stage
    # Use an .NET dsk 7.0 runtime as build image

    # Set the working directory to /app

    # Copy the current directory contents into the container at /app

    # Restore packages

    # Publish a release build to the /app/out directory

    # Runtime stage
    # Use an .NET sdk 7.0 runtime as base image

    # Set the working directory to /app

    # Copy the published app from the build image

    # Run the application
    ```

    Place your cursor at the beginning of each blank line and allow GitHub Copilot to suggest the appropriate command. Your Dockerfile should look like [this](/src/Web/Dockerfile).

3. Create an empty file `Dockerfile` in `src/PublicApi`.
4. Use comments to create a multi-stage Dockerfile for the PublicApi project:

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

    Place your cursor at the beginning of each blank line and allow GitHub Copilot to suggest the appropriate command. Your Dockerfile should look like [this](/src/PublicApi/Dockerfile). 
    
    If needed, begin to type the command and allow Copilot to suggest the rest. Repeat that for each new line that there should be a Docker command.

4. Build the container images:

    ```bash
    ACR_NAME=youracrname
    docker build . -f src/Web/Dockerfile -t $ACR_NAME.azurecr.io/web:v1.0.0
    docker build . -f src/PublicApi/Dockerfile -t $ACR_NAME.azurecr.io/publicapi:v1.0.0
    ```

5. Push the container images to the Azure Container Registry:

    ```bash
    docker push $ACR_NAME.azurecr.io/web:v1.0.0
    docker push $ACR_NAME.azurecr.io/publicapi:v1.0.0
    ```

    If necessary, login to the Azure Container Registry with the `az acr login` command.

## Writing the Kubernetes Configurations

Kubernetes manifests are YAML files that describe the desired state of your application and how to deploy it. The Kubernetes manifests for this application are located in the `manifests` directory.

1. Create a `manifests` directory in the root of the project.

    ```bash
    mkd manifests
    ```

2. Under the `manifests` directory, create the `deployment-db.yaml` file.

    ```bash
    touch manifests/deployment-db.yaml
    ```

3. Use comments to create a deployment manifest for the database:

    ```yaml
    # define a deployment using the `mcr.microsoft.com/mssql/server:2019-latest` image 
    # metadata: name db label app: db

    ---
    # Create a service
    # spec type: ClusterIP selector db protocol: TCP
    # port: 1433 targetPort: 1433
    ```

    Replace `SA_PASSWORD` with `@someThingComplicated1234`. Afterwards, your manifest should look like [this](/manifests/deployment-db.yaml).

4. Under the `manifests` directory, create the `deployment-web.yaml` file.

    ```bash
    touch manifests/deployment-web.yaml
    ```
5. Use comments to create a deployment manifest for the Web project:

    ```yaml
    # Define a deployment using notavalidregistry.azurecr.io/web:v1.0.0
    # metadata: name web label app: web
    # env: ASPNETCORE_ENVIRONMENT ASPNETCORE_URLS ConnectionStrings__CatalogConnection ConnectionStrings__IdentityConnection
    # ports: containerPort: 80
    # resources: {}

    ---
    #  ClusterIP for the web app on port 80 with protocol TCP
    ```

    Afterwards, your manifest should look like [this](/manifests/deployment-web.yaml).

6. Update the environment variable with the appropriate values.

    | Environment Variable | Value |
    | --- | --- |
    | ASPNETCORE_ENVIRONMENT | Development |
    | ASPNETCORE_URLS | http://+:80 |
    | ConnectionStrings__CatalogConnection | Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.CatalogDb;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True; |
    | ConnectionStrings__IdentityConnection | Server=db,1433;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.Identity;User Id=sa;Password=@someThingComplicated1234;Trusted_Connection=false;TrustServerCertificate=True; |

7. Under the `manifests` directory, create the `deployment-publicapi.yaml` file.

    ```bash
    touch manifests/deployment-publicapi.yaml
    ```

8. Use comments to create a deployment manifest for the PublicApi project:

    ```yaml
    # Define a deployment using notavalidregistry.azurecr.io/publicapi:v1.0.0
    # metadata: name api label app: api
    # env: ASPNETCORE_ENVIRONMENT ASPNETCORE_URLS ConnectionStrings__CatalogConnection ConnectionStrings__IdentityConnection

    ---
    # ClusterIP for the api app on port 80 no labels
    ```

9. Update the environment variable with the appropriate values. See table in step 6 for the values.

## Deploying to AKS

Now, that you have the Kubernetes manifests, you can deploy the application to AKS. But first, you have to create a kutomization file that will be used to deploy the application.

1. Create a file `./manifests/kustomize.yml` with the following content:

    ```yaml
    resources:
      - deployment-web.yaml
      - deployment-publicapi.yaml

    images:
    - name: notavalidregistry.azurecr.io/api:v0.1.0
    newName: <YOUR_ACR_SERVER>/api
    newTag: <YOUR_IMAGE_TAG>
    - name: notavalidregistry.azurecr.io/web:v0.1.0
    newName: <YOUR_ACR_SERVER>/web
    newTag: <YOUR_IMAGE_TAG>

    # kubectl apply manifests with kustomize
    ```

    Type out the contents of the kustomize file as show above and accept the suggestions from Copilot. Replace the `newName` and `newTag` values with the appropriate values. Once you have done that, your file should look like [this](/manifests/kustomization.yaml).

2. Deploy the application to AKS:

    ```bash
    kubectl apply -k manifests
    ```