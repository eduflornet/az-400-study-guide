# Cheat Sheet AZ-400

Comandos y snippets esenciales para el examen.

## 🔧 Azure CLI - Comandos Esenciales

### App Service
```bash
# Crear App Service
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name myApp \
  --runtime "NODE|18-lts"

# Deployment slots
az webapp deployment slot create --name myApp --resource-group myRG --slot staging
az webapp deployment slot swap --name myApp --resource-group myRG --slot staging

# Configuración
az webapp config appsettings set --name myApp --resource-group myRG \
  --settings KEY=VALUE

# Logs en tiempo real
az webapp log tail --name myApp --resource-group myRG
```

### Azure Container Registry (ACR)
```bash
# Crear ACR
az acr create --resource-group myRG --name myacr --sku Basic

# Build y push
az acr build --registry myacr --image myapp:v1 .

# Login
az acr login --name myacr

# Listar imágenes
az acr repository list --name myacr
```

### AKS (Azure Kubernetes Service)
```bash
# Crear cluster
az aks create \
  --resource-group myRG \
  --name myAKS \
  --node-count 2 \
  --enable-addons monitoring \
  --generate-ssh-keys

# Obtener credenciales
az aks get-credentials --resource-group myRG --name myAKS

# Conectar ACR con AKS
az aks update --name myAKS --resource-group myRG --attach-acr myacr
```

### Key Vault
```bash
# Crear Key Vault
az keyvault create --name myVault --resource-group myRG --location eastus

# Añadir secreto
az keyvault secret set --vault-name myVault --name ApiKey --value "secret123"

# Obtener secreto
az keyvault secret show --vault-name myVault --name ApiKey --query value -o tsv

# Dar acceso a service principal
az keyvault set-policy --name myVault \
  --spn <service-principal-id> \
  --secret-permissions get list
```

### Azure Monitor
```bash
# Crear Application Insights
az monitor app-insights component create \
  --app myAppInsights \
  --location eastus \
  --resource-group myRG

# Obtener instrumentation key
az monitor app-insights component show \
  --app myAppInsights \
  --resource-group myRG \
  --query instrumentationKey -o tsv
```

---

## 📝 Azure Pipelines YAML - Templates

### Pipeline CI Básico
```yaml
trigger:
  branches:
    include:
    - main
  paths:
    exclude:
    - docs/*

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

steps:
- task: UseDotNet@2
  inputs:
    version: '7.x'

- task: DotNetCoreCLI@2
  displayName: 'Restore'
  inputs:
    command: 'restore'

- task: DotNetCoreCLI@2
  displayName: 'Build'
  inputs:
    command: 'build'
    arguments: '--configuration $(buildConfiguration)'

- task: DotNetCoreCLI@2
  displayName: 'Test'
  inputs:
    command: 'test'
    arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'

- task: PublishCodeCoverageResults@1
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: '$(Agent.TempDirectory)/**/*coverage.cobertura.xml'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'
```

### Pipeline CD Multi-Stage
```yaml
stages:
- stage: Build
  jobs:
  - job: BuildJob
    steps:
    - script: npm run build
    - publish: $(Build.ArtifactStagingDirectory)
      artifact: drop

- stage: Dev
  dependsOn: Build
  jobs:
  - deployment: DeployDev
    environment: development
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: drop
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'connection'
              appName: 'myapp-dev'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'

- stage: Prod
  dependsOn: Dev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployProd
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'connection'
              appName: 'myapp-prod'
              deployToSlotOrASE: true
              slotName: 'staging'
          - task: AzureAppServiceManage@0
            inputs:
              action: 'Swap Slots'
              sourceSlot: 'staging'
```

### Template Reutilizable
```yaml
# templates/build-template.yml
parameters:
- name: buildConfiguration
  type: string
  default: 'Release'

steps:
- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    arguments: '--configuration ${{ parameters.buildConfiguration }}'

# azure-pipelines.yml
steps:
- template: templates/build-template.yml
  parameters:
    buildConfiguration: 'Release'
```

---

## 🐙 GitHub Actions - Workflows

### CI Workflow
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm test
    
    - name: Build
      run: npm run build
    
    - name: Upload artifact
      uses: actions/upload-artifact@v3
      with:
        name: build-${{ matrix.node-version }}
        path: dist/
```

### CD Workflow con Environments
```yaml
name: CD

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - uses: actions/checkout@v3
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    - uses: azure/webapps-deploy@v2
      with:
        app-name: myapp-staging
        package: .

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
    - uses: actions/checkout@v3
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    - uses: azure/webapps-deploy@v2
      with:
        app-name: myapp-prod
        slot-name: staging
    - name: Swap slots
      run: |
        az webapp deployment slot swap \
          --name myapp-prod \
          --resource-group myRG \
          --slot staging
```

### Reusable Workflow
```yaml
# .github/workflows/deploy-template.yml
name: Deploy Template

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      app-name:
        required: true
        type: string
    secrets:
      azure-credentials:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - uses: actions/checkout@v3
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.azure-credentials }}
    - uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ inputs.app-name }}

# .github/workflows/main.yml
jobs:
  deploy-dev:
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: development
      app-name: myapp-dev
    secrets:
      azure-credentials: ${{ secrets.AZURE_CREDENTIALS }}
```

---

## 🏗️ Bicep - Templates

### App Service + SQL
```bicep
param location string = resourceGroup().location
param appName string
param sqlServerName string
param sqlAdminLogin string
@secure()
param sqlAdminPassword string

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${appName}-plan'
  location: location
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
}

resource appService 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'WEBSITE_NODE_DEFAULT_VERSION'
          value: '18-lts'
        }
      ]
      connectionStrings: [
        {
          name: 'DefaultConnection'
          connectionString: 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Database=${sqlDatabase.name};'
          type: 'SQLAzure'
        }
      ]
    }
  }
}

resource sqlServer 'Microsoft.Sql/servers@2022-05-01-preview' = {
  name: sqlServerName
  location: location
  properties: {
    administratorLogin: sqlAdminLogin
    administratorLoginPassword: sqlAdminPassword
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2022-05-01-preview' = {
  parent: sqlServer
  name: '${appName}-db'
  location: location
  sku: {
    name: 'Basic'
  }
}

output appServiceUrl string = 'https://${appService.properties.defaultHostName}'
```

### AKS + ACR
```bicep
param clusterName string
param acrName string
param location string = resourceGroup().location

resource acr 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: acrName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    adminUserEnabled: false
  }
}

resource aks 'Microsoft.ContainerService/managedClusters@2023-05-01' = {
  name: clusterName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    dnsPrefix: clusterName
    agentPoolProfiles: [
      {
        name: 'agentpool'
        count: 2
        vmSize: 'Standard_DS2_v2'
        mode: 'System'
      }
    ]
  }
}

// Dar permiso a AKS para pull de ACR
resource acrPullRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(acr.id, aks.id, 'AcrPull')
  scope: acr
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '7f951dda-4ed3-4680-a7ca-43fe172d538d')
    principalId: aks.properties.identityProfile.kubeletidentity.objectId
    principalType: 'ServicePrincipal'
  }
}
```

---

## 🐳 Docker - Comandos y Dockerfiles

### Dockerfile Multi-Stage (Node.js)
```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

ENV NODE_ENV=production
EXPOSE 3000

USER node
CMD ["node", "dist/server.js"]
```

### Dockerfile (.NET)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=publish /app/publish .
EXPOSE 80
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Comandos Docker
```bash
# Build
docker build -t myapp:v1 .

# Run con variables de entorno
docker run -d -p 3000:3000 \
  -e NODE_ENV=production \
  -e API_KEY=secret \
  myapp:v1

# Tag y push a ACR
docker tag myapp:v1 myacr.azurecr.io/myapp:v1
docker push myacr.azurecr.io/myapp:v1

# Logs
docker logs -f <container-id>

# Exec
docker exec -it <container-id> /bin/sh
```

---

## ☸️ Kubernetes - Manifiestos

### Deployment con Health Checks
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myacr.azurecr.io/myapp:v1
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: api-key
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Comandos kubectl
```bash
# Apply manifests
kubectl apply -f deployment.yaml

# Get resources
kubectl get pods
kubectl get services
kubectl get deployments

# Describe
kubectl describe pod <pod-name>

# Logs
kubectl logs -f <pod-name>

# Exec
kubectl exec -it <pod-name> -- /bin/sh

# Scale
kubectl scale deployment myapp --replicas=5

# Rollout
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp

# Secrets
kubectl create secret generic myapp-secrets \
  --from-literal=api-key=secret123
```

---

## 🔐 Seguridad - Snippets

### Azure Pipeline con Key Vault
```yaml
steps:
- task: AzureKeyVault@2
  inputs:
    azureSubscription: 'connection'
    KeyVaultName: 'myVault'
    SecretsFilter: '*'
    RunAsPreJob: true

- script: |
    echo "Using secret: $(ApiKey)"
  env:
    API_KEY: $(ApiKey)
```

### GitHub Actions con Secrets
```yaml
steps:
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
    DB_CONNECTION: ${{ secrets.DB_CONNECTION }}
  run: |
    echo "Deploying with secrets"
```

### Escaneo de Seguridad
```yaml
# GitHub Actions - CodeQL
- name: Initialize CodeQL
  uses: github/codeql-action/init@v2
  with:
    languages: javascript

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v2

# Trivy - Container scanning
- name: Run Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myacr.azurecr.io/myapp:v1
    format: 'sarif'
    output: 'trivy-results.sarif'
```

---

## 📊 Monitoreo - Application Insights

### Integración en código (Node.js)
```javascript
const appInsights = require('applicationinsights');
appInsights.setup(process.env.APPINSIGHTS_INSTRUMENTATIONKEY)
  .setAutoDependencyCorrelation(true)
  .setAutoCollectRequests(true)
  .setAutoCollectPerformance(true)
  .setAutoCollectExceptions(true)
  .setAutoCollectDependencies(true)
  .start();

const client = appInsights.defaultClient;

// Custom event
client.trackEvent({ name: 'UserLogin', properties: { userId: '123' } });

// Custom metric
client.trackMetric({ name: 'OrderValue', value: 99.99 });
```

### Queries KQL
```kql
// Errores en las últimas 24h
exceptions
| where timestamp > ago(24h)
| summarize count() by type, outerMessage
| order by count_ desc

// Response time percentiles
requests
| where timestamp > ago(1h)
| summarize percentiles(duration, 50, 90, 95, 99) by bin(timestamp, 5m)

// Dependency failures
dependencies
| where success == false
| summarize count() by name, resultCode
```

---

## ✅ Comandos de Validación Rápida

```bash
# Validar Bicep
az bicep build --file main.bicep

# Validar deployment
az deployment group validate \
  --resource-group myRG \
  --template-file main.bicep

# Test pipeline localmente (Azure Pipelines)
# Instalar: npm install -g azure-pipelines-cli
azure-pipelines validate --file azure-pipelines.yml

# Validar Dockerfile
docker build --no-cache -t test .

# Validar manifiestos K8s
kubectl apply --dry-run=client -f deployment.yaml
```

---

> **Tip:** Guarda este cheat sheet y tenlo abierto durante las prácticas. La velocidad es clave en el examen.
