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
resource containerAppEnv 'Microsoft.App/managedEnvironments@2022-10-01' = {
  name: containerAppEnvName
  location: location
  sku: {
    name: 'Consumption'
  }
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

Navigate to your resource group in the Azure Portal and verify that the resources have been deployed successfully. You should see something similar to the following:

![Picture of resources deployed defined in our Bicep template](../aca-networking/media/external-environment-resources.png)

Click on your virtual network resource, then under **Settings** click **Subnets**. You'll see that our subnet that we defined in our template has been successfully provisioned with the address prefix that we defined for it in our template:

![Picture of the subnet that we defined in our Bicep template](../aca-networking/media/external-environment-subnet.png)

Navigate to your Container App environment. Here we can see that our Container App environment has been successfully integrated with our defined virtual network and the subnet we created as part of that virtual network. We can also see that an external Public facing IP address has been provisioned for our environment:

![Picture of the overview blade for our Container App environment](../aca-networking/media/external-environment-ca.png)

Navigate to the Container App that was deployed as part of the template and click on the **Application Url**. The page should be accessible (since our environment is accepting traffic from the public internet) and we should see the following:

![The external webpage of our container app](../aca-networking/media/external-environment-web.png)

## Switching to an internal environment

We will now restrict all access to our container app by changing our environment to an internal one. To do this, we need to add another property to our ```vnetConfiguration``` object in our Container App environment resource block:

```bicep
resource containerAppEnv 'Microsoft.App/managedEnvironments@2022-10-01' = {
  // Emitted for brevity
    vnetConfiguration: {
      infrastructureSubnetId: vnet.properties.subnets[0].id
      internal: true
    }
}
```

Here, we are simply setting the ```internal``` property to true, indicating that this will be an internal Container App environment.

Redeploy your template by running the following command:

```bash
az deployment group create --resource-group $RG_NAME --template-file main.bicep
```

*NOTE: If you've created an external parameters file, you can pass this through to the command by using the* ```--parameters``` *flag*.

