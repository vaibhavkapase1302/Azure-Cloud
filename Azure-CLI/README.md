# Get started with Azure CLI

https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest

## Install or run in Azure Cloud Shell

```sh
az version
```

### az account set
Set a subscription to be the current active subscription

```sh
az account set --name --subscription
```

Required Parameters

```--name --subscription -n -s```

Name or ID of subscription.

### az account show
Get the details of a subscription.

If the subscription isn't specified, shows the details of the default subscription.

```sh
az account show [--name --subscription]
```

