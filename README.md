# MLOps Tutorial
This is an MLOps Tutorial done following Mohammad Ghodratigohar's MLOps workshop with amendments to ensure deployment.
Visit MG's Youtube playlist to learn more on how to implement this continous integration/continous deployment setup as well as learn how to setup infrastructure as a code.

https://www.youtube.com/playlist?list=PLiQS6N-W1p3m9squzZ2cPgGdH5SBhjY6f

## Workflow

- Azure Repo

- Continuous Integration \(CI\)

- Deployment\:

    - Staging\: Azure Container Instance \& Azure Machine Learning

    - Production\: Kubernetes

Note\: Tests are conducted post staging and post deployment\.


## CI Pipeline

### Use Python3.7
### Install python requirements

```
script path: package_requirement/install_requirements.sh
```

### Data test

```
pytest training/train_test.py --doctest-modules --junitxml=junit/test-results.xml --cov=data_test --cov-report=xml --cov-report=html
```

### Publish Test Results **/test-*.xml

```
**/test-*.xml
```

### Install Azure ML CLI

```
az extension add -n azure-cli-ml
```

### Azure ML Compute Cluster

```
az ml computetarget create amlcompute -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(amlcompute.clusterName) -s $(amlcompute.vmSize) --min-nodes $(amlcompute.minNodes) --max-nodes $(amlcompute.maxNodes) --idle-seconds-before-scaledown $(amlcompute.idleSecondsBeforeScaledown)
```

### Create Azure ML workspace

```
az ml workspace create -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -l $(azureml.location) --exist-ok --yes
```

### Upload data to Datastore

```
az ml datastore upload -w $(azureml.workspaceName) -g $(azureml.resourceGroup) -n $(az ml datastore show-default -w $(azureml.workspaceName) -g $(azureml.resourceGroup) --query name -o tsv) -p data -u insurance --overwrite true
```

### Make Metadata and Models directory

```
mkdir metadata && mkdir models
```

### Training Model

```
az ml run submit-script -g $(azureml.resourceGroup) -w $(azureml.workSpaceName) -e $(experiment.name) --ct $(amlcompute.clusterName) -d conda_dependencies.yml -c train_wbgt -t ../metadata/run.json train_aml.py
```

### Registering Model

```
az ml model register -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(model.name) -f metadata/run.json --asset-path outputs/models/insurance_model.pkl -d "Classification model for filing a claim" --tag "data"="insurance" --tag "model"="classification" --model-framework ScikitLearn -t metadata/model.json
```

### Downloading Model

```
az ml model download -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -i $(jq -r .modelId metadata/model.json) -t ./models --overwrite
```

### Copy Files to: $(Build.ArtifactStagingDirectory)

```
**/metadata/*
**/models/*
**/deployment/*
**/tests/integration/*
**/package_requirement/*
```

### Publish Pipeline Artifact

```
File or path Directory: $(Build.ArtifactStagingDirectory)
Artifact Name: landing
Artifact Publish Location: Azure Pipelines
```

## Releases
### Deploy to Staging
### Use Python 3.7
### Add ML Extension

```
az extension add -n azure-cli-ml
```

### Deploy to Azure Container Instance

```
az ml model deploy -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.staging) -f ../metadata/model.json --dc aciDeploymentConfigStaging.yml --ic inferenceConfig.yml --overwrite
```

### Install Requirements

```
Script path: $(System.DefaultWorkingDirectory)/_MLOps1-CI/landing/package_requirement/install_requirements.sh
```

### Staging Test

```
pytest staging_test.py --doctest-modules --junitxml=junit/test-results.xml --cov-report=xml --cov-report=html --scoreurl $(az ml service show -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.staging) --query scoringUri -o tsv)
```

### Publish Staging Test Results

```
**/test-*.xml
```

### Deploy to Production
### Use Python 3.7
### Azure CLI ML Extension

```
az extension add -n azure-cli-ml
```

### Create Azure Kubernetes Services

```
az ml computetarget create aks -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(aks.clusterName) -s $(aks.vmSize) -a $(aks.agentCount)
```

### Deploy to AKS

```
az ml model deploy -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.prod) -f ../metadata/model.json --dc aksDeploymentConfigProd.yml --ic inferenceConfig.yml --ct $(aks.clusterName) --overwrite
```

### Install Python Requirement

```
Script Path: $(System.DefaultWorkingDirectory)/_MLOps1-CI/landing/package_requirement/install_requirements.sh
```

### Production Test

```
pytest prod_test.py --doctest-modules --junitxml=junit/test-result.xml --cov=integration_test --cov-report=xml --cov-report=html --scoreurl $(az ml service show -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.prod) --query scoringUri -o tsv) --scorekey $( az ml service get-keys -g $(azureml.resourceGroup) -w $(azureml.workspaceName) -n $(service.name.prod) --query primaryKey -o tsv)
```

### Publish Production Test Result
**/test-*.xml
Search folder: $(System.DefaultWorkingDirectory)

## Library Variables

Variable group name: mlops-wsh-vg

Variables Name/ Value

AZURE_RM_SVC_CONNECTION/azure-resource-connection (Based on what you set in service connections)

BASE_NAME/mlopstt (Use whatever base name you want)

LOCATION/centralus(Use whatever location you want. This will affect the servers later on)

RESOURCE_GROUP/mlopstt-wsh-rg-2 (use whatever resource group name you want)

WORKSPACE_NAME/mlopstt-wsh-aml-2 (Same for here)

WORKSPACE_SVC_CONNECTION/aml-workspace-connection (Based on what you set in service connections)

## CI Variables

Variable Name/Variable Value

amlcompute.clusterName/amlcluster

amlcompute.idleSecondsBeforeScaledown/300

amlcompute.maxNodes/4

amlcompute.minNodes/0

amlcompute.vmSize/Standard_D4s_v3

azureml.location/centralus

azureml.resourceGroup/mlopstt-wsh-rg-2

azureml.workspaceName/mlopstt-wsh-aml-2

system.debug/false

system.definitionId/3

system.teamProject/MLOps1

experiment.name/insurance_classification

model.name/insurance_model

## Release Variables

Variable Name/Variable Value

aks.agentCount/3

aks.clusterName/aks

aks.vmSize/Standard_E4s_v3

azureml.resourceGroup/mlopstt-wsh-rg-2

azureml.workspaceName/mlopstt-wsh-aml-2

Service.name.prod/insurance-service-aks

service.name.staging/insurance-service-aci


## Recommendations:
- The tutorial uses Kubernetes for production deployment. It would be good to explore alternatives if the deployment is not being done on enterprise production, for example mini kubernetes or even just deploy to azure container instance, because the costs are very high to keep a server running K8 going. After Kubernetes deployment and testing, turn off the server manually to ensure costs do not accumulate. 

## Amendments from MG's original tutorial:
- Python version has been amended to ensure no error during run. User Python 3.7.

- Use Ubuntu-20.04, the ubuntu version in the workshop is no longer available in Azure Devops.

- Server type has been changed to a higher specification server. Azure has changed the server requirements to run Kubernetes and you may run into a lack of core situation. As of 22/8/23 Standard_E4s_v3 works. 

- However, suitable servers as of this date is far and few and may not even be available on certain regions.

- Lightgbm version as been amended to 3.3.0 instead of using latest in conda_dependencies to ensure no error during run.

- Parallelism restrictions has been implemented by Azure to prevent abuse of service, this also means that we would run into parallelism issues when running. You can either apply for a free parallelistm grant as per the error or follow the steps here to self host windows agent to bypass this issue https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/windows-agent?view=azure-devops

- Once the Kubernetes is deployed in production during the first run, you will run into an error when the pipeline run again trying to Create Azure Kubernetes Services. To date I have not found a solution that gracefully handles this, thus the alternative is to select Continue on Error under Control Options.
