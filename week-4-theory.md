# Semana 4: Entrega Continua e Infraestructura como Código

## ¿Qué es Entrega Continua (CD)?
Entrega Continua es la práctica de preparar automáticamente los cambios de código para un release a producción. El despliegue es el proceso de liberar esos cambios.

## Conceptos Clave

### CI vs CD vs CD

- **Continuous Integration (CI):** Build y test automático
- **Continuous Delivery (CD):** Preparar para producción (aprobación manual)
- **Continuous Deployment (CD):** Despliegue automático a producción

```
Código → CI (Build/Test) → CD (Deploy a Staging) → Aprobación → Producción
                                                      ↓
                                            Continuous Deployment
                                            (Sin aprobación manual)
```

### Componentes de CD

1. **Environments:** dev, staging, production
2. **Approvals:** Aprobaciones manuales o automáticas
3. **Gates:** Validaciones pre/post despliegue
4. **Deployment Strategies:** Blue-green, canary, rolling
5. **Rollback:** Capacidad de revertir cambios

## Multi-Stage Pipelines

### Estructura Básica

```yaml
stages:
- stage: Build
  jobs:
  - job: BuildJob
    steps:
    - script: npm run build
    - publish: $(Build.ArtifactStagingDirectory)
      artifact: drop

- stage: DeployDev
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: DeployToDev
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

- stage: DeployStaging
  dependsOn: DeployDev
  jobs:
  - deployment: DeployToStaging
    environment: staging
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              appName: 'myapp-staging'

- stage: DeployProd
  dependsOn: DeployStaging
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployToProd
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              appName: 'myapp-prod'
```

### Environments

#### Configurar Environment con Aprobaciones

```yaml
# En Azure DevOps UI:
# Environments → production → Approvals and checks
# - Add check: Approvals (mínimo 2 aprobadores)
# - Add check: Business hours
# - Add check: Azure Policy compliance

jobs:
- deployment: DeployProd
  environment: 
    name: production
    resourceType: VirtualMachine
```

#### GitHub Environments

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
    - name: Deploy
      run: echo "Deploying to production"
```

**Configurar en GitHub:**
- Settings → Environments → production
- Required reviewers: 2 personas
- Wait timer: 5 minutos
- Deployment branches: solo main

## Estrategias de Despliegue

### 1. Rolling Deployment (Actualización Gradual)

```yaml
strategy:
  rolling:
    maxParallel: 2
    preDeploy:
      steps:
      - script: echo "Pre-deployment checks"
    deploy:
      steps:
      - task: AzureWebApp@1
    postDeploy:
      steps:
      - script: echo "Post-deployment validation"
```

**Ventajas:**
- Sin downtime
- Gradual, menos riesgo

**Desventajas:**
- Versiones mixtas temporalmente
- Rollback más complejo

### 2. Blue-Green Deployment

```yaml
steps:
# Desplegar a slot "blue" (staging)
- task: AzureWebApp@1
  inputs:
    deployToSlotOrASE: true
    slotName: 'blue'
    appName: 'myapp'

# Validar en blue
- script: |
    curl https://myapp-blue.azurewebsites.net/health
    if [ $? -ne 0 ]; then exit 1; fi

# Swap slots (blue → production)
- task: AzureAppServiceManage@0
  inputs:
    action: 'Swap Slots'
    sourceSlot: 'blue'
    targetSlot: 'production'
```

**Ventajas:**
- Zero downtime
- Rollback instantáneo (swap de vuelta)
- Testing en producción antes de switch

**Desventajas:**
- Requiere doble infraestructura
- Bases de datos pueden ser complejas

### 3. Canary Deployment

```yaml
# Desplegar a 10% de usuarios
- task: Kubernetes@1
  inputs:
    command: 'apply'
    arguments: '-f canary-deployment.yaml'

# Monitorear métricas
- script: |
    error_rate=$(curl https://api/metrics/error_rate)
    if (( $(echo "$error_rate > 0.05" | bc -l) )); then
      echo "Error rate too high, rolling back"
      exit 1
    fi

# Si todo bien, aumentar a 50%
- task: Kubernetes@1
  inputs:
    command: 'scale'
    arguments: 'deployment/myapp-canary --replicas=5'

# Finalmente, 100%
```

**Ventajas:**
- Riesgo mínimo
- Feedback real de usuarios
- Rollback fácil

**Desventajas:**
- Más complejo de implementar
- Requiere monitoreo robusto

### 4. Feature Flags

```javascript
// Código con feature flag
if (featureFlags.isEnabled('new-checkout')) {
  return newCheckoutFlow();
} else {
  return oldCheckoutFlow();
}
```

```yaml
# Desplegar código con flag desactivado
- task: AzureWebApp@1
  inputs:
    appSettings: '-FEATURE_NEW_CHECKOUT false'

# Activar gradualmente
# 10% → 25% → 50% → 100%
```

## Infraestructura como Código (IaC)

### ¿Por qué IaC?

- **Reproducibilidad:** Misma infra en todos los ambientes
- **Versionado:** Cambios en Git
- **Automatización:** Despliegues consistentes
- **Documentación:** El código es la documentación
- **Testing:** Validar infra antes de desplegar

### Bicep (Recomendado para Azure)

#### App Service + SQL Database

```bicep
// main.bicep
param location string = resourceGroup().location
param appName string
param environment string

@secure()
param sqlAdminPassword string

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${appName}-plan-${environment}'
  location: location
  sku: {
    name: environment == 'prod' ? 'P1v2' : 'B1'
    tier: environment == 'prod' ? 'PremiumV2' : 'Basic'
  }
}

// App Service
resource appService 'Microsoft.Web/sites@2022-03-01' = {
  name: '${appName}-${environment}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'ENVIRONMENT'
          value: environment
        }
        {
          name: 'DATABASE_CONNECTION'
          value: 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Database=${sqlDatabase.name};'
        }
      ]
      connectionStrings: [
        {
          name: 'DefaultConnection'
          connectionString: 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Initial Catalog=${sqlDatabase.name};Persist Security Info=False;User ID=${sqlServer.properties.administratorLogin};Password=${sqlAdminPassword};'
          type: 'SQLAzure'
        }
      ]
    }
  }
}

// Deployment Slot (solo en prod)
resource stagingSlot 'Microsoft.Web/sites/slots@2022-03-01' = if (environment == 'prod') {
  parent: appService
  name: 'staging'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
  }
}

// SQL Server
resource sqlServer 'Microsoft.Sql/servers@2022-05-01-preview' = {
  name: '${appName}-sql-${environment}'
  location: location
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
    minimalTlsVersion: '1.2'
  }
}

// SQL Database
resource sqlDatabase 'Microsoft.Sql/servers/databases@2022-05-01-preview' = {
  parent: sqlServer
  name: '${appName}-db'
  location: location
  sku: {
    name: environment == 'prod' ? 'S1' : 'Basic'
  }
}

// Firewall rule para Azure services
resource firewallRule 'Microsoft.Sql/servers/firewallRules@2022-05-01-preview' = {
  parent: sqlServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

// Outputs
output appServiceUrl string = 'https://${appService.properties.defaultHostName}'
output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
```

#### Módulos Reutilizables

```bicep
// modules/app-service.bicep
param appName string
param location string
param planId string

resource appService 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: planId
  }
}

output appServiceId string = appService.id
```

```bicep
// main.bicep
module appService 'modules/app-service.bicep' = {
  name: 'appServiceDeployment'
  params: {
    appName: 'myapp'
    location: location
    planId: appServicePlan.id
  }
}
```

#### Integrar Bicep en Pipeline

```yaml
steps:
- task: AzureCLI@2
  displayName: 'Validate Bicep'
  inputs:
    azureSubscription: 'connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az deployment group validate \
        --resource-group $(resourceGroup) \
        --template-file main.bicep \
        --parameters @parameters.$(environment).json

- task: AzureCLI@2
  displayName: 'Deploy Infrastructure'
  inputs:
    azureSubscription: 'connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az deployment group create \
        --resource-group $(resourceGroup) \
        --template-file main.bicep \
        --parameters @parameters.$(environment).json \
        --parameters sqlAdminPassword=$(sqlPassword)
```

### Terraform

#### Configuración Básica

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}

# Variables
variable "environment" {
  type = string
}

variable "location" {
  type    = string
  default = "eastus"
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "myapp-${var.environment}-rg"
  location = var.location
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "myapp-plan-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = var.environment == "prod" ? "P1v2" : "B1"
}

# App Service
resource "azurerm_linux_web_app" "main" {
  name                = "myapp-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      node_version = "18-lts"
    }
  }

  app_settings = {
    "ENVIRONMENT" = var.environment
  }
}

# Outputs
output "app_url" {
  value = "https://${azurerm_linux_web_app.main.default_hostname}"
}
```

#### Workspaces para Múltiples Ambientes

```bash
# Crear workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Usar workspace
terraform workspace select dev
terraform apply -var="environment=dev"
```

#### Integrar Terraform en Pipeline

```yaml
steps:
- task: TerraformInstaller@0
  inputs:
    terraformVersion: 'latest'

- task: TerraformTaskV4@4
  displayName: 'Terraform Init'
  inputs:
    provider: 'azurerm'
    command: 'init'
    backendServiceArm: 'connection'
    backendAzureRmResourceGroupName: 'terraform-state-rg'
    backendAzureRmStorageAccountName: 'tfstate'
    backendAzureRmContainerName: 'tfstate'
    backendAzureRmKey: '$(environment).terraform.tfstate'

- task: TerraformTaskV4@4
  displayName: 'Terraform Plan'
  inputs:
    provider: 'azurerm'
    command: 'plan'
    environmentServiceNameAzureRM: 'connection'
    commandOptions: '-var="environment=$(environment)" -out=tfplan'

- task: TerraformTaskV4@4
  displayName: 'Terraform Apply'
  inputs:
    provider: 'azurerm'
    command: 'apply'
    environmentServiceNameAzureRM: 'connection'
    commandOptions: 'tfplan'
```

## Rollback Strategies

### Automático con Health Checks

```yaml
- task: AzureWebApp@1
  inputs:
    appName: 'myapp-prod'
    
- script: |
    # Esperar 30 segundos
    sleep 30
    
    # Verificar health endpoint
    response=$(curl -s -o /dev/null -w "%{http_code}" https://myapp-prod.azurewebsites.net/health)
    
    if [ $response -ne 200 ]; then
      echo "Health check failed, rolling back"
      exit 1
    fi
  displayName: 'Health Check'
  
- task: AzureAppServiceManage@0
  condition: failed()
  inputs:
    action: 'Swap Slots'
    sourceSlot: 'production'
    targetSlot: 'staging'
  displayName: 'Rollback on Failure'
```

### Manual con Aprobación

```yaml
- stage: Rollback
  dependsOn: DeployProd
  condition: failed()
  jobs:
  - deployment: RollbackProd
    environment: production-rollback
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureAppServiceManage@0
            inputs:
              action: 'Swap Slots'
```

## Recursos de Estudio
- [Azure Pipelines Release](https://learn.microsoft.com/es-es/azure/devops/pipelines/release/)
- [Bicep Documentation](https://learn.microsoft.com/es-es/azure/azure-resource-manager/bicep/)
- [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)

## Ejercicio Práctico

### Parte 1: Multi-Stage Pipeline (2h)
1. Crear pipeline con 3 stages: dev, staging, prod
2. Configurar aprobaciones en prod
3. Implementar health checks
4. Configurar rollback automático

### Parte 2: IaC con Bicep (2h)
1. Crear template Bicep para App Service + SQL
2. Parametrizar por ambiente
3. Integrar en pipeline
4. Desplegar a dev y staging

### Parte 3: Blue-Green Deployment (2h)
1. Configurar deployment slots
2. Implementar swap automático
3. Agregar validación pre-swap
4. Probar rollback

## Preguntas de Repaso

1. ¿Cuál es la diferencia entre Continuous Delivery y Continuous Deployment?
2. ¿Cuándo usarías blue-green vs canary deployment?
3. ¿Qué ventajas tiene Bicep sobre ARM templates?
4. ¿Cómo implementas rollback automático en un pipeline?
5. ¿Qué es un deployment slot y cuándo lo usarías?
6. ¿Cómo gestionas secretos en templates de IaC?

---

> **Tip:** En el examen, conocer las estrategias de despliegue y cuándo usar cada una es crucial. Blue-green es ideal para zero-downtime, canary para minimizar riesgo.
