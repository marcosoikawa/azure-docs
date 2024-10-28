---
title: Quickstart - Create a new Backend Pool in Azure API Management using Bicep
description: Use this quickstart to create a backend pool in Azure API Management instance in the Standar V2 tier by using Bicep.
services: azure-resource-manager
author: maoika
ms.service: azure-api-management
tags: azure-resource-manager, bicep
ms.custom: devx-track-bicep, subject-bicepqs, devx-track-azurecli, devx-track-azurepowershell
ms.topic: quickstart-bicep
ms.author: maoika
ms.date: 10/28/2024
---

# Quickstart: Create a new Backend Pool in Azure API Management using Bicep

[!INCLUDE [api-management-availability-all-tiers](../../includes/api-management-availability-all-tiers.md)]

This quickstart describes how to use a Bicep file to create an backend pool in Azure API Management instance.

A *backend* (or *API backend*) in API Management is an HTTP service that implements your front-end API and its operations.

[!INCLUDE [About Bicep](~/reusable-content/ce-skilling/azure/includes/resource-manager-quickstart-bicep-introduction.md)]

## Prerequisites

- If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

- For Azure CLI:

    [!INCLUDE [azure-cli-prepare-your-environment-no-header.md](~/reusable-content/azure-cli/azure-cli-prepare-your-environment-no-header.md)]

- For Azure PowerShell:


    [!INCLUDE [azure-powershell-requirements-no-header](~/reusable-content/ce-skilling/azure/includes/azure-powershell-requirements-no-header.md)]

## Review the Bicep file

The template used in this quickstart is shown below. It deploys an API Management, two Open AI Services, creates the backends for the two endpoints and then creates the backend pool for those two backends

<!-- :::code language="bicep" source="~/quickstart-templates/quickstarts/microsoft.apimanagement/azure-api-management-create/main.bicep"::: -->

```bicep

var resourceSuffix = uniqueString(subscription().id, resourceGroup().id)
@description('The name of the API Management service instance')
param apiManagementServiceName string = 'apiservice${uniqueString(subscription().id, resourceGroup().id)}'

@description('The pricing tier of this API Management service')
@allowed([
  'Consumption'
  'Developer'
  'Basic'
  'Basicv2'
  'Standard'
  'Standardv2'
  'Premium'
])
param sku string = 'Basicv2'

@description('The instance size of this API Management service.')
@allowed([
  0
  1
  2
])
param skuCount int = 1

@description('Location for all resources.')


param location string = resourceGroup().location
param openAISku string = 'S0'

resource cognitiveServices1 'Microsoft.CognitiveServices/accounts@2021-10-01' = {
  name: 'cog1-${resourceSuffix}'
  location: location
  sku: {
    name: openAISku
  }
  kind: 'OpenAI'  
  properties: {
    apiProperties: {
      statisticsEnabled: false
    }
    customSubDomainName: toLower('cog1-${resourceSuffix}')
  }
}
resource openaidepl1 'Microsoft.CognitiveServices/accounts/deployments@2023-05-01'  =  {
  name: 'openaideploy1'
  parent: cognitiveServices1
  properties: {
    model: {
      format: 'OpenAI'
      name: 'gpt-35-turbo'
      version: '0613'
    }
  }
  sku: {
      name: 'Standard'
      capacity: 10
  }
}

resource cognitiveServices2 'Microsoft.CognitiveServices/accounts@2021-10-01' = {
  name: 'cog2-${resourceSuffix}'
  location: location
  sku: {
    name: openAISku
  }
  kind: 'OpenAI'  
  properties: {
    apiProperties: {
      statisticsEnabled: false
    }
    customSubDomainName: toLower('cog2-${resourceSuffix}')
  }
}
resource openaidepl2 'Microsoft.CognitiveServices/accounts/deployments@2023-05-01'  =  {
  name: 'openaideploy2'
  parent: cognitiveServices2
  properties: {
    model: {
      format: 'OpenAI'
      name: 'gpt-35-turbo'
      version: '0613'
    }
  }
  sku: {
      name: 'Standard'
      capacity: 10
  }
}



resource apiManagementService 'Microsoft.ApiManagement/service@2023-05-01-preview' = {
  name: apiManagementServiceName
  location: location
  sku: {
    name: sku
    capacity: skuCount
  }
  properties: {
    publisherEmail: 'publisher@contoso.com'
    publisherName: 'Contos Publisher'
  }
  identity: {
    type: 'SystemAssigned'
  } 
}


var roleDefinitionID = resourceId('Microsoft.Authorization/roleDefinitions', '5e0bd9bd-7b93-4f28-af87-19fc36ad61bd')
resource roleAssignment1 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
    scope: cognitiveServices1
    name: guid(subscription().id, resourceGroup().id, 'cognitiveServices1')
    properties: {
        roleDefinitionId: roleDefinitionID
        principalId: apiManagementService.identity.principalId
        principalType: 'ServicePrincipal'
    }
}
resource roleAssignment2 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: cognitiveServices2
  name: guid(subscription().id, resourceGroup().id, 'cognitiveServices2')
  properties: {
      roleDefinitionId: roleDefinitionID
      principalId: apiManagementService.identity.principalId
      principalType: 'ServicePrincipal'
  }
}


resource api 'Microsoft.ApiManagement/service/apis@2023-05-01-preview' = {
  name: 'openai'
  parent: apiManagementService
  properties: {
    apiType: 'http'
    description: 'Azure OpenAI API from API Management'
    displayName: 'OpenAI'
    format: 'openapi-link'
    path: 'openai'
    protocols: [
      'https'
    ]
    subscriptionKeyParameterNames: {
      header: 'api-key'
      query: 'api-key'
    }
    subscriptionRequired: true
    type: 'http'
    value: 'https://raw.githubusercontent.com/Azure/azure-rest-api-specs/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference/stable/2024-02-01/inference.json'
  }
}
resource apiPolicy 'Microsoft.ApiManagement/service/apis/policies@2021-12-01-preview' = {
  name: 'policy'
  parent: api
  properties: {
    format: 'rawxml'
    value: loadTextContent('policy.xml')
  }
}


resource backendOpenAI1 'Microsoft.ApiManagement/service/backends@2023-05-01-preview' = {
  name: 'backend1'
  parent: apiManagementService
  properties: {
    description: 'backend description'
    url: '${cognitiveServices1.properties.endpoint}/openai'
    protocol: 'http'
    circuitBreaker: {
      rules: [
        {
          failureCondition: {
            count: 3
            errorReasons: [
              'Server errors'
            ]
            interval: 'PT5M'
            statusCodeRanges: [
              {
                min: 429
                max: 429
              }
            ]
          }
          name: 'openAIBreakerRule'
          tripDuration: 'PT1M'
        }
      ]
    }    
  }
}
resource backendOpenAI2 'Microsoft.ApiManagement/service/backends@2023-05-01-preview' = {
  name: 'backend2'
  parent: apiManagementService
  properties: {
    description: 'backend description'
    url: '${cognitiveServices2.properties.endpoint}/openai'
    protocol: 'http'
    circuitBreaker: {
      rules: [
        {
          failureCondition: {
            count: 3
            errorReasons: [
              'Server errors'
            ]
            interval: 'PT5M'
            statusCodeRanges: [
              {
                min: 429
                max: 429
              }
            ]
          }
          name: 'openAIBreakerRule'
          tripDuration: 'PT1M'
        }
      ]
    }    
  }
}


resource backendPoolOpenAI 'Microsoft.ApiManagement/service/backends@2023-05-01-preview' = {
  name: 'openai-backend-pool'
  parent: apiManagementService
  properties: {
    description: 'Load balancer for multiple OpenAI endpoints'
    type: 'Pool'
    pool: {
      services: [
        {
          id: '/backends/${backendOpenAI1.name}'
          priority: 1
          weight: 10
        }
        {
          id: '/backends/${backendOpenAI2.name}'
          priority: 1
          weight: 20
        }]
      
    }
  }
}

resource apimSubscription 'Microsoft.ApiManagement/service/subscriptions@2023-05-01-preview' = {
  name: 'openAISubscriptionName'
  parent: apiManagementService
  properties: {
    allowTracing: true
    displayName: 'Open AI Subscription Description'
    scope: '/apis/${api.id}'
    state: 'active'
  }
}

```

The following resource is defined in the Bicep file:

- **[Microsoft.ApiManagement/service/backends](/azure/templates/microsoft.apimanagement/service/backends)**

More Azure API Management Bicep samples can be found in [Azure Quickstart Templates](https://azure.microsoft.com/resources/templates/?resourceType=Microsoft.Apimanagement&pageNumber=1&sort=Popular).

## Deploy the Bicep file

You can use Azure CLI or Azure PowerShell to deploy the Bicep file.  For more information about deploying Bicep files, see [Deploy](../azure-resource-manager/bicep/deploy-cli.md).

1. Save the Bicep file as **main.bicep** to your local computer.
1. Deploy the Bicep file using either Azure CLI or Azure PowerShell.

    # [CLI](#tab/CLI)

    ```azurecli
    az group create --name exampleRG --location eastus

    az deployment group create --resource-group exampleRG --template-file main.bicep
    ```

    # [PowerShell](#tab/PowerShell)

    ```azurepowershell
    New-AzResourceGroup -Name exampleRG -Location eastus

    New-AzResourceGroupDeployment -ResourceGroupName exampleRG -TemplateFile ./main.bicep
    ```

    ---

    When the deployment finishes, you should see a message indicating the deployment succeeded.

    > [!TIP]
    >  It can take between 10 and 30 minutes to create and activate an API Management service. Times vary by tier.

## Review deployed resources

Use the Azure portal, Azure CLI or Azure PowerShell to list the deployed App Configuration resource in the resource group.

# [CLI](#tab/CLI)

```azurecli-interactive
az resource list --resource-group exampleRG
```

# [PowerShell](#tab/PowerShell)

```azurepowershell-interactive
Get-AzResource -ResourceGroupName exampleRG
```

---

When your API Management service instance is online, you're ready to use it. Start with the tutorial to [import and publish](import-and-publish.md) your first API.

## Clean up resources

When no longer needed, delete the resource group, which deletes the resources in the resource group.

# [CLI](#tab/CLI)

```azurecli-interactive
az group delete --name exampleRG
```

# [PowerShell](#tab/PowerShell)

```azurepowershell-interactive
Remove-AzResourceGroup -Name exampleRG
```

---

## Related content

* API Management Backends: [Backends in API Management](https://learn.microsoft.com/en-us/azure/api-management/backends?tabs=bicep)