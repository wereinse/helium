# Build a Web API reference application using Managed Identity, Key Vault, and Cosmos DB that is designed to be deployed to Azure App Service or Azure Kubernetes Service (AKS)

![License](https://img.shields.io/badge/license-MIT-green.svg)

This is a Web API reference application designed to "fork and code" with the following features:

- Securely build, deploy and run an Azure App Service (Web App for Containers) application
- Securely build, deploy and run an Azure Kubernetes Service (AKS) application
- Use Managed Identity to securely access resources
- Securely store secrets in Key Vault
- Securely build and deploy the Docker container from Azure Container Registry (ACR) or Azure DevOps
- Connect to and query Cosmos DB
- Automatically send telemetry and logs to Azure Monitor

![alt text](./docs/images/architecture.jpg "Architecture Diagram")

## Prerequisites

- Azure subscription with permissions to create:
  - Resource Groups, Service Principals, Key Vault, Cosmos DB, Azure Container Registry, Azure Monitor, App Service or AKS
- Bash shell (tested on Mac, Ubuntu, Windows with WSL2)
  - Will not work in Cloud Shell unless you have a remote dockerd
- Azure CLI 2.0.72+ ([download](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest))
- Docker CLI ([download](https://docs.docker.com/install/))
- Visual Studio Code (optional) ([download](https://code.visualstudio.com/download))

## Setup

- Fork this repo and clone to your local machine
  - cd to the base directory of the repo
  - all instructions assume starting from the base directory of this repo

Login to Azure and select subscription

```bash

az login

# show your Azure accounts
az account list -o table

# select the Azure account
az account set -s {subscription name or Id}

export He_Sub=$(az account show --subscription {subscription name or Id} --output tsv |awk '{print $3}')

```

Choose a unique DNS name

```bash

# this will be the prefix for all resources
# only use a-z and 0-9 - do not include punctuation or uppercase characters
# must be at least 5 characters long
# must start with a-z (only lowercase)
export He_Name="youruniquename"

### if true, change He_Name
az cosmosdb check-name-exists -n ${He_Name}

### if nslookup doesn't fail to resolve, change He_Name
nslookup ${He_Name}.azurewebsites.net
nslookup ${He_Name}.vault.azure.net
nslookup ${He_Name}.azurecr.io

```

Create Resource Groups

- When experimenting with this app, you should create new resource groups to avoid accidentally deleting resources

  - If you use an existing resource group, please make sure to apply resource locks to avoid accidentally deleting resources

- You will create 3 resource groups
  - One for ACR
  - One for App Service or AKS, Key Vault and Azure Monitor
  - One for Cosmos DB (see create and load sample data to Cosmos DB step)

```bash

# set location
export He_Location=centralus

# resource group names
export He_ACR_RG=${He_Name}-rg-acr
export He_App_RG=${He_Name}-rg-app

# create the resource groups
az group create -n $He_App_RG -l $He_Location
az group create -n $He_ACR_RG -l $He_Location

```

Save your environment variables for ease of reuse and picking up where you left off.

```bash

# run the saveenv.sh script at any time to save He_*, Imdb_*, MSI_*, and AKS_* variables to ~/${He_Name}.env
# make sure you are in the root of the repo
./saveenv.sh

# at any point if your terminal environment gets cleared, you can source the file
# you only need to remember the name of the env file
source ~/{yoursameuniquename}.env

```

Create and load sample data into Cosmos DB

- This takes several minutes to run
- This reference app is designed to use a simple dataset from IMDb of 1300 movies and their associated actors and genres
- Follow the guidance in the [IMDb Repo](https://github.com/retaildevcrews/imdb) to create a Cosmos DB server (SQL API), a database, and a collection and then load the IMDb data. The repo readme also provides an explanation of the data model design decisions.
- Recommendation is to set $Imdb_Name the same value as $He_Name

```bash

# Run saveenv.sh to save the Imdb variables
./saveenv.sh

# Verify the output of the script shows the newly created Imdb_* variables

```

Get the Cosmos DB read only key used by App Service

```bash

export Imdb_Cosmos_RO_Key=$(az cosmosdb keys list -n $Imdb_Name -g $Imdb_RG --query primaryMasterKey -o tsv)

```

Create Azure Key Vault

- All secrets are stored in Azure Key Vault for security
  - This app uses Managed Identity to access Key Vault

```bash

## create the Key Vault and add secrets
az keyvault create -g $He_App_RG -n $He_Name

# add Cosmos DB keys
az keyvault secret set -o table --vault-name $He_Name --name "CosmosUrl" --value https://${Imdb_Name}.documents.azure.com:443/
az keyvault secret set -o table --vault-name $He_Name --name "CosmosKey" --value $Imdb_Cosmos_RO_Key
az keyvault secret set -o table --vault-name $He_Name --name "CosmosDatabase" --value $Imdb_DB
az keyvault secret set -o table --vault-name $He_Name --name "CosmosCollection" --value $Imdb_Col

```

(Optional) In order to run the application locally, each developer will need access to the Key Vault. Since you created the Key Vault during setup, you will automatically have permission, so this step is only required for additional developers.

Use the following command to grant permissions to each developer that will need access.

```bash

# get the object id for each developer (optional)
export dev_Object_Id=$(az ad user show --id {developer email address} --query objectId -o tsv)

# grant Key Vault access to each developer (optional)
az keyvault set-policy -n $He_Name --secret-permissions get list --key-permissions get list --object-id $dev_Object_Id

```

Choose your app language

- Choose which language version of the Helium container you want to build
- Options are: [csharp](https://github.com/retaildevcrews/helium-csharp), [java (WIP)](https://github.com/retaildevcrews/helium-java), or [typescript (WIP)](https://github.com/retaildevcrews/helium-typescript)

```bash

# set to either csharp, java, or typescript
export He_Language=csharp

```

Setup Container Registry

- Create the Container Registry with admin access _disabled_

> Currently, App Service cannot access ACR via the Managed Identity, so we have to setup a separate Service Principal and grant access to that SP.

```bash

# create the ACR
az acr create --sku Standard --admin-enabled false -g $He_ACR_RG -n $He_Name

# Login to ACR
az acr login -n $He_Name

# if you get an error that the login server isn't available, it's a DNS issue that will resolve in a minute or two, just retry

# Build the container with az acr build
az acr build -r $He_Name -t $He_Name.azurecr.io/helium-${He_Language} https://github.com/retaildevcrews/helium-${He_Language}.git

```

Create a Service Principal for Container Registry

- App Service will use this Service Principal to access Container Registry

```bash

# create a Service Principal
export He_SP_PWD=$(az ad sp create-for-rbac -n http://${He_Name}-acr-sp --query password -o tsv)
export He_SP_ID=$(az ad sp show --id http://${He_Name}-acr-sp --query appId -o tsv)

# get the Container Registry Id
export He_ACR_Id=$(az acr show -n $He_Name -g $He_ACR_RG --query "id" -o tsv)

# assign acrpull access to Service Principal
az role assignment create --assignee $He_SP_ID --scope $He_ACR_Id --role acrpull

# add credentials to Key Vault
az keyvault secret set -o table --vault-name $He_Name --name "AcrUserId" --value $He_SP_ID
az keyvault secret set -o table --vault-name $He_Name --name "AcrPassword" --value $He_SP_PWD

# Optional: Run ./saveenv.sh to save latest variables

```

Create Azure Monitor

- The Application Insights extension is in preview and needs to be added to the CLI

```bash

# Add App Insights extension
az extension add -n application-insights

# Create App Insights
export He_AppInsights_Key=$(az monitor app-insights component create -g $He_App_RG -l $He_Location -a $He_Name --query instrumentationKey -o tsv)

# add App Insights Key to Key Vault
az keyvault secret set -o table --vault-name $He_Name --name "AppInsightsKey" --value $He_AppInsights_Key

# Optional: Run ./saveenv.sh to save latest variables

```

Deploy the container to App Service or AKS

- Instructions for [App Service](docs/AppService.md)
- Instructions for [AKS](docs/aks/README.md#L233) (currently requires csharp)

Run the Integration Test

```bash

# run the tests in the container
docker run -it --rm retaildevcrew/webvalidate --host https://${He_Name}.azurewebsites.net --files baseline.json

```
## Dashboard setup

Replace the values in the `Helium_Dashboard.json` file surrounded by `%%` with the proper environment variables
after making sure the proper environment variables are set (He_Sub, He_App_RG, Imdb_RB and Imdb_Name)

```bash

cd $REPO_ROOT/docs/dashboard
sed -i "s/%%SUBSCRIPTION_GUID%%/${He_Sub}/g" Helium_Dashboard.json && \
sed -i "s/%%He_App_RG%%/${He_App_RG}/g" Helium_Dashboard.json && \
sed -i "s/%%Imdb_RG%%/${Imdb_RG}/g" Helium_Dashboard.json && \
sed -i "s/%%Imdb_NAME%%/${Imdb_Name}/g" Helium_Dashboard.json

if [ "$He_Language" == "java" ]; then sed -i "s/%%He_Language%%/gelato/g" Helium_Dashboard.json; elif [ "$He_Language" == "csharp" ]; then sed -i "s/%%He_Language%%/bluebell/g" Helium_Dashboard.json;  elif [ "$He_Language" == "typescript" ]; then sed -i "s/%%He_Language%%/sherbert/g" Helium_Dashboard.json; fi

```

Navigate to ([Dashboard](https://portal.azure.com/#dashboard)) within your Azure portal. Click upload and select the `Helium_Dashboard.json` file with your correct subscription GUID, resource group names, and app name.

For more documentation on creating and sharing Dashboards, see ([here](https://docs.microsoft.com/en-us/azure/azure-portal/azure-portal-dashboards)).

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit [Microsoft Contributor License Agreement](https://cla.opensource.microsoft.com).

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.