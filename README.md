# Lab: Working with Networking and Azure Container Apps

In this lab, you'll gain some experience of working with Virtual Networks and Container Apps. You'll learn:

- The difference between external and internal Container App environments
- How to integrate a virtual network with an external Azure Container Apps environment using Bicep
- How to disable ingress to the Container App environment with internal environments.

## Networking in Container Apps

Azure Container Apps run in the context of an environment. This boundary around your container apps are supported by virtual networks.

When you create a Container App environment, a virtual network is created for you automatically. This resource will be inaccessible to you, as this virtual network lives inside Microsoft's tenant. This virtual network is publicly accessible over the internet, can only reach internet accessible endpoints and supported limited networking capabilities.

In this lab, you'll create a Bicep template that integrates a virtual network with an external Container App environment. You'll also learn how we can restrict access to our environment by making it internal (inaccessible to the public internet) and apply IP restrictions to our Container Apps.

## Creating our resource group

To deploy our resources, we will need to create a resource group via the Azure CLI. We can achieve this by doing the following:

```bash
# If you need to, login
az login

# Set variables for the name and location of your resource group
RG_NAME='<name-of-your-resource-group>'
LOCATION='<azure-region-near-you>'

# Create the resource group
az group create --name $RG_NAME --location $LOCATION
```

Once your resource group has been successfully created, you can start to write out your Bicep template.

## Creating our Bicep template

Once you've cloned this repository, navigate to the **Before** folder and examine the *main.bicep* file. There are a few TODO comments here that we need to resolve. There are some parameters that have been provided to you, but if you have some experience with using parameter files in Bicep/ARM, you are welcome to do so.

Looking at the Bicep template, we'll need to create the following resources:

- Azure Virtual Network (with a subnet)
- Container App environment

### Creating our VNET and Subnet resources

We will first create the virtual network that will act as a boundary around our Container App environment as well as a subnet that needs to be available for our environment deployment. To do so, write the following Bicep

```bicep
resource vnet 'Microsoft.Network/virtualNetworks@2022-07-01' = {
  name: virtualNetworkName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        vnetAddressPrefix
      ]
    }
    subnets: [
      {
        name: containerAppEnvSubnetName
        properties: {
          addressPrefix: containerAppEnvSubnetPrefix
        }
      }
    ]
  }
}
```

This will create a virtual network resource with an address prefix of *10.0.0.0/16* and a subnet with an address prefix of *10.0.0.0/23*. We will be creating a Consumption only architecture, which requires a minimum CIDR range of */23*.

Workload Profiles Architecture (currently in preview) need a */27* or larger CIDR range

### Creating our Container App Environment

With our subnet resource defined, we can now define our Container App environment resource and deploy it with our virtual network created. To do so, write the following Bicep

```bicep
resource containerAppEnv 'Microsoft.App/managedEnvironments@2022-03-01' = {
  name: containerAppEnvName
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
    vnetConfiguration: {
      infrastructureSubnetId: vnet.properties.subnets[0].id
    }
  }
}
```

The important line to note here is the ```infrastructureSubnetId``` property. Here, we are setting it to the resource ID of the subnet that we created earlier. This will integrate our virtual network to our Container App environment, which will accept external requests from the public internet.

## Deploying our template

To deploy our template, we can run the following AZ CLI command:

```bash
az deployment group create --resource-group $RG_NAME --template-file main.bicep
```

*NOTE: If you've created an external parameters file, you can pass this through to the command by using the* ```--parameters``` *flag*.