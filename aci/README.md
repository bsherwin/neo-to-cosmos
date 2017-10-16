# Scaling out Neo2Cosmos with Azure Container Instances
Copying large volume of data from Neo4j to CosmosDB using a single instance of the app may not be entirely feasible, even with maxed out [RUs](https://docs.microsoft.com/en-us/azure/cosmos-db/request-units) and a Redis layer in between.

Hence we've created a little orchestration script to deploy the required resources (Cosmos DB, Redis and Storage Account), as well as spin up N number of `Azure Container Instances`.

## Prereqs
- Modify vars on top of `deploy.sh` script, pointing to your publicly available Neo4j server.
- Install [latest Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).
- Open `Bash` and do `az login`
- Make sure your desired subscription is selected with `az account set --subscription <SUBSCRIPTION_ID>`

## Run the script
`./deploy.sh`

> It takes ~5-10min to provision all resources for the first time.

## How it works
Here are the steps we perform;

> **Note:** For simplicity, we've chosen `$NEO2COSMOS_NAME` for the resource group as well as all resources inside. Hence it's important to follow Azure Storage Account [naming convention](https://docs.microsoft.com/en-us/azure/architecture/best-practices/naming-conventions).

- Create resource group in specified region.
- Deploy Cosmos DB, Redis and Storage account using [this ARM template](https://github.com/syedhassaanahmed/neo-to-cosmos/blob/master/aci/deploy-resources.json).
- Fetch auth keys for newly created  resources.
- Create `config.json` with above auth keys.
- Create an Azure File Share and upload config.json. This file share will be volume mounted on each container instance (specified in next template).
- Deploy N number of instances with [this ARM template](https://github.com/syedhassaanahmed/neo-to-cosmos/blob/master/aci/deploy-aci.json). The template creates containers with environment variables `TOTAL` and `INSTANCE`, which are then [passed to the app](https://github.com/syedhassaanahmed/neo-to-cosmos/blob/master/Dockerfile). The app is aware of how to interpret them and distribute the load accordingly.

## Caution
Azure container instances are [currently limited to long-running services](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-troubleshooting#container-continually-exits-and-restarts). If your app exits, the container will simply be destroyed and replaced with a new one. This is extremely useful if there is an exception during migration. However for the same reason we keep the app alive even after migration is complete for a particular instance.

**TL:DR;** You're responsible for deleting all the container groups after migration is complete!!!