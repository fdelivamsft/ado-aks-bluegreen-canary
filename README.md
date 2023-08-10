# Guide to "blue-green" deployments using Azure Pipelines

## What is Blue/Green deployment strategy?
| ![blue-green-deploy-process](https://raw.githubusercontent.com/fdelivamsft/ado-aks-bluegreen-canary/main/img/blue-green-deployment-process.gif) |
| :--: |
| *Blue/Green Deployment* |
>Blue/Green deployments are a form of progressive delivery where a new version of the application is deployed while the old version still exists. The two versions coexist for a brief period of time while user traffic is routed to the new version, before the old version is discarded (if all goes well).

---
## What is Canary deployment strategy?
| ![canary-deploy-process](https://raw.githubusercontent.com/fdelivamsft/ado-aks-bluegreen-canary/main/img/canary-deploy.gif) |
| :--: |
| *Canary Deployment* |
>Canary deployment strategy involves deploying new versions of an application next to stable production versions to see how the canary version compares against the baseline before promoting or rejecting the deployment. 

---
### **This is a simple tutorial on how to do [Blue/Green Deployment] in Azure Pipeline using Key Vault as blue/green repository.**

## Prerequisites
* Create a project in Azure DevOps
* Access your Azure Portal and create a [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/quick-create-portal) 
* In Azure DevOps create a Service Connection to you Azure Subscription
---
## Blue/Green Deployment Pipeline:
* For Blue/Green deployment, create `blue-green.yaml` file:
```yml
trigger:
- main

pool:
  vmImage: ubuntu-latest

variables:
  BLUE_ENDPOINT_INT_TEST: "<blue internal url>"
  GREEN_ENDPOINT_INT_TEST: "<green internal url>"
  BLUE_DEPLOYMENT_SCRIPT: "kubernetes/blue-deploy.yaml"
  GREEN_DEPLOYMENT_SCRIPT: "kubernetes/green-deploy.yaml"
  KEY_VAULT_NAME: "<your key vault name>"

stages:
- stage: Config
  jobs:
  - job: retrieve
    steps:
    - task: AzureKeyVault@2
      inputs:
        azureSubscription: '<<Service Connection Name>>'
        KeyVaultName: $(KEY_VAULT_NAME)
        SecretsFilter: '*'
        RunAsPreJob: true
      name: switchKey
    - bash: |
        if [ "$SWITCHBG" = "blue" ]; then
            echo "##vso[task.setvariable variable=targetEnvValue;isoutput=true]green"
            echo "##vso[task.setvariable variable=endpoint;isoutput=true]$(GREEN_ENDPOINT_INT_TEST)"
            echo "##vso[task.setvariable variable=script;isoutput=true]$(GREEN_DEPLOYMENT_SCRIPT)"
        fi
        if [ "$SWITCHBG" = "green" ]; then
            echo "##vso[task.setvariable variable=targetEnvValue;isoutput=true]blue"
            echo "##vso[task.setvariable variable=endpoint;isoutput=true]$(BLUE_ENDPOINT_INT_TEST)"
            echo "##vso[task.setvariable variable=script;isoutput=true]$(BLUE_DEPLOYMENT_SCRIPT)"
        fi
      env:
        SWITCHBG: $(switchbluegreen)
      name: switchRetrieve
  
- stage: Deploy
  dependsOn: Config
  jobs:
  - job: Deployment 
    variables: 
      target: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.targetEnvValue'] ]
      script: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.script'] ]
    steps:
    - script: |
        echo "Deployment to $(target) environemnt"
        echo "Using the $(script) script"
  - job: Validate 
    dependsOn: Deployment
    variables: 
      endpoint: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.endpoint'] ]
    steps:
    - task: Bash@3
      inputs:
        targetType: 'inline'
        script: |
          result=$(curl -s -o /dev/null -w "%{http_code}" $(endpoint))
          
          if [ "$result" == "200" ]
          then
          echo "OK"
          else
          echo $result
          exit 1
          fi
  - deployment: UpdateLoadBalancer 
    displayName: Update the Load Balancer
    dependsOn: Validate
    variables: 
      target: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.targetEnvValue'] ]
      endpoint: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.endpoint'] ]
    environment:
      name: bluegreen
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
             echo "Update the load balancer"
          - task: AzureCLI@2
            inputs:
              azureSubscription: '<<Service Connection Name>>'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: 'az keyvault secret set --name switchbluegreen --vault-name $(KEY_VAULT_NAME) --value $(target)'

```
| ![Blue-Green Deployment Workflow](https://raw.githubusercontent.com/fdelivamsft/ado-aks-bluegreen-canary/main/img/success.png) |
| :--: |
| *Blue/Green Deployment Workflow* |
### Customization is required:
This pipeline is intended to be a reference to achieve blue/green deployment. You will need to:
* Based on your target compute service, add your code to deploy your solution (Stage: Deploy, Job: Deployment)
* Based on your load balancer or network service to route traffic, you will need to customize the code to change the routing configuration between the blue or green clusters (Stage: Deploy, Job: UpdateLoadBalancer)
* Change the values of your azure subscription service connection, the urls of your blue and green environments and the deployment scripts
* Configure the bluegreen environment with an approver to ensure that the final swap between environment is done with a proper signoff
* Create in your Azure Key Vault a secret named "switchbluegreen"
### How does it work?
Belowe I will explain how does each part of the code works.
#### Initial Setup
In this section I defined a pipeline that it is trigger from main branch and use a linux machine to build. Then I define several variable that will be useful later. The endpoints are used to test that a deployment was successful, the scripts refers to a trivial example in K8S, so they should be changed with your script to deploy your environments.
```yml
trigger:
- main

pool:
  vmImage: ubuntu-latest

variables:
  BLUE_ENDPOINT_INT_TEST: "<blue internal url>"
  GREEN_ENDPOINT_INT_TEST: "<green internal url>"
  BLUE_DEPLOYMENT_SCRIPT: "kubernetes/blue-deploy.yaml"
  GREEN_DEPLOYMENT_SCRIPT: "kubernetes/green-deploy.yaml"
  KEY_VAULT_NAME: "<your key vault name>"

stages:
- stage: Config
  jobs:
  - job: retrieve
    steps:
    - task: AzureKeyVault@2
      inputs:
        azureSubscription: '<<Service Connection Name>>'
        KeyVaultName: $(KEY_VAULT_NAME)
        SecretsFilter: '*'
        RunAsPreJob: true
      name: switchKey
    - bash: |
        if [ "$SWITCHBG" = "blue" ]; then
            echo "##vso[task.setvariable variable=targetEnvValue;isoutput=true]green"
            echo "##vso[task.setvariable variable=endpoint;isoutput=true]$(GREEN_ENDPOINT_INT_TEST)"
            echo "##vso[task.setvariable variable=script;isoutput=true]$(GREEN_DEPLOYMENT_SCRIPT)"
        fi
        if [ "$SWITCHBG" = "green" ]; then
            echo "##vso[task.setvariable variable=targetEnvValue;isoutput=true]blue"
            echo "##vso[task.setvariable variable=endpoint;isoutput=true]$(BLUE_ENDPOINT_INT_TEST)"
            echo "##vso[task.setvariable variable=script;isoutput=true]$(BLUE_DEPLOYMENT_SCRIPT)"
        fi
      env:
        SWITCHBG: $(switchbluegreen)
      name: switchRetrieve
  
- stage: Deploy
  dependsOn: Config
  jobs:
  - job: Deployment 
    variables: 
      target: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.targetEnvValue'] ]
      script: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.script'] ]
    steps:
    - script: |
        echo "Deployment to $(target) environemnt"
        echo "Using the $(script) script"
  - job: Validate 
    dependsOn: Deployment
    variables: 
      endpoint: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.endpoint'] ]
    steps:
    - task: Bash@3
      inputs:
        targetType: 'inline'
        script: |
          result=$(curl -s -o /dev/null -w "%{http_code}" $(endpoint))
          
          if [ "$result" == "200" ]
          then
          echo "OK"
          else
          echo $result
          exit 1
          fi
  - deployment: UpdateLoadBalancer 
    displayName: Update the Load Balancer
    dependsOn: Validate
    variables: 
      target: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.targetEnvValue'] ]
      endpoint: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.endpoint'] ]
    environment:
      name: bluegreen
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
             echo "Update the load balancer"
          - task: AzureCLI@2
            inputs:
              azureSubscription: '<<Service Connection Name>>'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: 'az keyvault secret set --name switchbluegreen --vault-name $(KEY_VAULT_NAME) --value $(target)'

```
#### Initial Setup
In this section I defined a pipeline that it is trigger from main branch and use a linux machine to build. Then I define several variable that will be useful later. The endpoints are used to test that a deployment was successful, the scripts refers to a trivial example in K8S, so they should be changed with your script to deploy your environments.
```yml
trigger:
- main

pool:
  vmImage: ubuntu-latest

variables:
  BLUE_ENDPOINT_INT_TEST: "<blue internal url>"
  GREEN_ENDPOINT_INT_TEST: "<green internal url>"
  BLUE_DEPLOYMENT_SCRIPT: "kubernetes/blue-deploy.yaml"
  GREEN_DEPLOYMENT_SCRIPT: "kubernetes/green-deploy.yaml"
  KEY_VAULT_NAME: "<your key vault name>"
```
#### Retrieve Stage
In this step, I am retrieving the "switchbluegreen" secret from Azure Key Vault. Then based on the value, I set the target environment, the scripts and the testing endpoints.
```yml
stages:
- stage: Config
  jobs:
  - job: retrieve
    steps:
    - task: AzureKeyVault@2
      inputs:
        azureSubscription: '<<Service Connection Name>>'
        KeyVaultName: $(KEY_VAULT_NAME)
        SecretsFilter: '*'
        RunAsPreJob: true
      name: switchKey
    - bash: |
        if [ "$SWITCHBG" = "blue" ]; then
            echo "##vso[task.setvariable variable=targetEnvValue;isoutput=true]green"
            echo "##vso[task.setvariable variable=endpoint;isoutput=true]$(GREEN_ENDPOINT_INT_TEST)"
            echo "##vso[task.setvariable variable=script;isoutput=true]$(GREEN_DEPLOYMENT_SCRIPT)"
        fi
        if [ "$SWITCHBG" = "green" ]; then
            echo "##vso[task.setvariable variable=targetEnvValue;isoutput=true]blue"
            echo "##vso[task.setvariable variable=endpoint;isoutput=true]$(BLUE_ENDPOINT_INT_TEST)"
            echo "##vso[task.setvariable variable=script;isoutput=true]$(BLUE_DEPLOYMENT_SCRIPT)"
        fi
      env:
        SWITCHBG: $(switchbluegreen)
      name: switchRetrieve 
```
#### Deploy
The Deployment Stage will be executed only if Config is successful.
This is the part that requires your customization to plug the scripts used by your DevOps team to deploy your code in any cloud you wish.
Please note the reference "stageDependencies.Config.retrieve.outputs..." to access variable of other stages.
```yml 
- stage: Deploy
  dependsOn: Config
  jobs:
  - job: Deployment 
    variables: 
      target: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.targetEnvValue'] ]
      script: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.script'] ]
    steps:
    - script: |
        echo "Deployment to $(target) environemnt"
        echo "Using the $(script) script"
```
#### Validation Step
After deploy, you would like to check that everything is working fine. In this case I created a basic script to check if your service is returning HTTP 200 when queried. A more complex logic can be implemented here.
```yml

  - job: Validate 
    dependsOn: Deployment
    variables: 
      endpoint: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.endpoint'] ]
    steps:
    - task: Bash@3
      inputs:
        targetType: 'inline'
        script: |
          result=$(curl -s -o /dev/null -w "%{http_code}" $(endpoint))
          
          if [ "$result" == "200" ]
          then
          echo "OK"
          else
          echo $result
          exit 1
          fi

```
#### Change the load balancer configuration
This step requires some customization based on your networking configuration to write the script to change your load balancer from point an environment to another.
After performing this activity, the script updates the value of the environment from blue to green or viceversa.
```yml
  - deployment: UpdateLoadBalancer 
    displayName: Update the Load Balancer
    dependsOn: Validate
    variables: 
      target: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.targetEnvValue'] ]
      endpoint: $[ stageDependencies.Config.retrieve.outputs['switchRetrieve.endpoint'] ]
    environment:
      name: bluegreen
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
             echo "Update the load balancer"
          - task: AzureCLI@2
            inputs:
              azureSubscription: '<<Service Connection Name>>'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: 'az keyvault secret set --name switchbluegreen --vault-name $(KEY_VAULT_NAME) --value $(target)'

```
