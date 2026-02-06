# Escenarios de Examen AZ-400

Casos prácticos tipo examen con soluciones paso a paso.

## 🎯 Escenario 1: Pipeline Fallando por Dependencias

### Situación
Tu pipeline CI falla intermitentemente. Los logs muestran errores de dependencias no encontradas. El equipo reporta que funciona localmente.

### Pregunta
¿Qué acciones debes tomar? (Selecciona 3)

**Opciones:**
- A) Usar cache de dependencias en el pipeline
- B) Especificar versiones exactas en package.json/requirements.txt
- C) Aumentar el timeout del pipeline
- D) Usar self-hosted agent en lugar de Microsoft-hosted
- E) Implementar restore de dependencias como step separado

### Respuesta Correcta
**A, B, E**

### Solución Práctica
```yaml
# GitHub Actions
- name: Cache dependencies
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

- name: Install dependencies
  run: npm ci  # usa package-lock.json, más determinista que npm install
```

---

## 🎯 Escenario 2: Secretos Expuestos en Logs

### Situación
Un desarrollador accidentalmente hizo commit de una API key. La removiste del código, pero sigue en el historial de Git.

### Pregunta
¿Qué debes hacer? (Orden correcto)

**Opciones:**
1. Rotar inmediatamente la API key comprometida
2. Usar git filter-branch o BFG Repo-Cleaner
3. Configurar secret scanning
4. Force push al repositorio
5. Notificar al equipo de seguridad

### Respuesta Correcta
**1 → 5 → 2 → 4 → 3**

### Solución Práctica
```bash
# 1. Rotar el secreto inmediatamente en Azure Key Vault
az keyvault secret set --vault-name myVault --name ApiKey --value "new-key"

# 2. Limpiar historial
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch config/secrets.json" \
  --prune-empty --tag-name-filter cat -- --all

# 3. Configurar GitHub Advanced Security
# Settings → Security → Enable secret scanning
```

---

## 🎯 Escenario 3: Despliegue Lento a Producción

### Situación
Tu pipeline CD tarda 45 minutos en desplegar a producción. El 80% del tiempo es compilación y pruebas que ya se ejecutaron en CI.

### Pregunta
¿Cómo optimizas el proceso?

**Opciones:**
- A) Usar artifacts del pipeline CI en lugar de recompilar
- B) Ejecutar pruebas en paralelo
- C) Saltar las pruebas en CD
- D) Usar deployment slots para swap rápido
- E) Aumentar el tier del App Service

### Respuesta Correcta
**A, B, D**

### Solución Práctica
```yaml
# Pipeline CI - Publicar artifacts
- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'

# Pipeline CD - Consumir artifacts
- task: DownloadBuildArtifacts@1
  inputs:
    buildType: 'specific'
    project: 'MyProject'
    pipeline: 'CI-Pipeline'
    artifactName: 'drop'

# Deployment con slots
- task: AzureWebApp@1
  inputs:
    deployToSlotOrASE: true
    slotName: 'staging'
- task: AzureAppServiceManage@0
  inputs:
    action: 'Swap Slots'
    sourceSlot: 'staging'
```

---

## 🎯 Escenario 4: Rollback Urgente

### Situación
Desplegaste a producción y los usuarios reportan errores críticos. Necesitas hacer rollback inmediato.

### Pregunta
¿Cuál es la estrategia más rápida y segura?

**Opciones:**
- A) Revertir el commit y ejecutar pipeline completo
- B) Usar deployment slots swap
- C) Redesplegar la versión anterior manualmente
- D) Usar Azure App Service deployment history
- E) Restaurar desde backup

### Respuesta Correcta
**B (si usas slots) o D (si no)**

### Solución Práctica
```bash
# Opción 1: Swap slots (instantáneo)
az webapp deployment slot swap \
  --resource-group myRG \
  --name myApp \
  --slot staging \
  --target-slot production

# Opción 2: Rollback desde portal
# App Service → Deployment Center → Deployment History → Redeploy

# Opción 3: Pipeline con rollback automático
# Configurar health check y auto-rollback en release pipeline
```

---

## 🎯 Escenario 5: Múltiples Equipos, Un Repositorio

### Situación
Tienes 3 equipos trabajando en el mismo monorepo. Cada equipo necesita su propio pipeline que solo se ejecute cuando cambian sus archivos.

### Pregunta
¿Cómo configuras los triggers?

### Solución Práctica
```yaml
# Pipeline Team A
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - services/team-a/*
      - shared/common/*
    exclude:
      - services/team-b/*
      - services/team-c/*

# GitHub Actions - Team B
on:
  push:
    branches: [main]
    paths:
      - 'services/team-b/**'
      - 'shared/common/**'
```

---

## 🎯 Escenario 6: Compliance y Auditoría

### Situación
Tu organización requiere que todos los despliegues a producción sean auditables y aprobados por al menos 2 personas del equipo de seguridad.

### Pregunta
¿Qué configuraciones implementas?

### Solución Práctica
```yaml
# Azure Pipelines - Environments con aprobaciones
stages:
- stage: Production
  jobs:
  - deployment: DeployProd
    environment: 
      name: production
      resourceType: VirtualMachine
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo Deploying to prod

# Configurar en Azure DevOps:
# Environments → production → Approvals and checks
# - Add check: Approvals (mínimo 2 aprobadores)
# - Add check: Business hours
# - Add check: Azure Policy compliance

# GitHub Actions - Environments
# Settings → Environments → production
# - Required reviewers: 2 personas
# - Wait timer: 5 minutos
# - Deployment branches: solo main
```

---

## 🎯 Escenario 7: Drift en Infraestructura

### Situación
Alguien modificó manualmente recursos en Azure. Tu pipeline IaC ahora falla porque el estado real no coincide con el código.

### Pregunta
¿Cómo detectas y corriges el drift?

### Solución Práctica
```bash
# Con Terraform
terraform plan -detailed-exitcode
# Exit code 2 = hay cambios (drift detectado)

# Opciones:
# 1. Importar cambios manuales al state
terraform import azurerm_app_service.main /subscriptions/.../myApp

# 2. Forzar que el código sea la verdad
terraform apply -auto-approve

# 3. Prevenir cambios manuales con Azure Policy
az policy assignment create \
  --name 'deny-manual-changes' \
  --policy 'deny-modifications-outside-pipeline'

# Con Bicep - usar what-if
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep
```

---

## 🎯 Escenario 8: Performance del Pipeline

### Situación
Tu pipeline tarda 30 minutos. Tienes 50 desarrolladores haciendo push constantemente. Los pipelines se acumulan.

### Pregunta
¿Qué optimizaciones implementas? (Selecciona todas las aplicables)

**Opciones:**
- A) Usar parallel jobs
- B) Implementar cache de dependencias
- C) Usar self-hosted agents más potentes
- D) Ejecutar solo tests afectados por los cambios
- E) Dividir en múltiples pipelines
- F) Usar matrix strategy

### Respuesta Correcta
**Todas son válidas según el contexto**

### Solución Práctica
```yaml
# Parallel jobs
jobs:
  test-unit:
    runs-on: ubuntu-latest
  test-integration:
    runs-on: ubuntu-latest
  test-e2e:
    runs-on: ubuntu-latest

# Cache
- uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      ~/.cache
    key: ${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}

# Matrix strategy
strategy:
  matrix:
    node: [16, 18, 20]
    os: [ubuntu-latest, windows-latest]
  max-parallel: 6

# Tests selectivos (ejemplo con Jest)
- name: Run affected tests
  run: npx jest --onlyChanged --changedSince=origin/main
```

---

## 🎯 Escenario 9: Multi-Cloud Deployment

### Situación
Necesitas desplegar la misma aplicación en Azure y AWS. Quieres mantener un solo pipeline.

### Pregunta
¿Cómo estructuras el pipeline?

### Solución Práctica
```yaml
# Azure Pipelines con templates
stages:
- stage: BuildOnce
  jobs:
  - job: Build
    steps:
    - script: npm run build
    - publish: $(Build.ArtifactStagingDirectory)

- stage: DeployAzure
  dependsOn: BuildOnce
  jobs:
  - template: templates/deploy-azure.yml

- stage: DeployAWS
  dependsOn: BuildOnce
  jobs:
  - template: templates/deploy-aws.yml

# templates/deploy-azure.yml
steps:
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'Azure-Connection'

# templates/deploy-aws.yml
steps:
- task: AWSCLI@1
  inputs:
    awsCredentials: 'AWS-Connection'
    regionName: 'us-east-1'
    awsCommand: 's3'
    awsSubCommand: 'sync'
```

---

## 🎯 Escenario 10: Zero-Downtime Deployment

### Situación
Tu aplicación debe estar disponible 24/7. Necesitas desplegar sin downtime.

### Pregunta
¿Qué estrategias implementas?

### Solución Práctica
```yaml
# Estrategia 1: Blue-Green con App Service Slots
- task: AzureWebApp@1
  inputs:
    deployToSlotOrASE: true
    slotName: 'blue'

- task: AzureAppServiceManage@0
  inputs:
    action: 'Start Swap With Preview'
    sourceSlot: 'blue'

# Validación en staging
- script: |
    curl https://myapp-blue.azurewebsites.net/health
    if [ $? -ne 0 ]; then exit 1; fi

- task: AzureAppServiceManage@0
  inputs:
    action: 'Complete Swap'

# Estrategia 2: Canary en AKS
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  progressDeadlineSeconds: 60
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
```

---

## ✅ Checklist de Escenarios

Practica resolver estos escenarios sin ayuda:

- [ ] Pipeline fallando intermitentemente
- [ ] Secreto expuesto en repositorio
- [ ] Optimizar pipeline lento
- [ ] Rollback urgente a producción
- [ ] Configurar triggers por paths
- [ ] Implementar aprobaciones y compliance
- [ ] Detectar y corregir drift en IaC
- [ ] Optimizar performance de pipelines
- [ ] Despliegue multi-cloud
- [ ] Zero-downtime deployment

---

## 📚 Recursos para Escenarios

- [Azure DevOps Troubleshooting](https://docs.microsoft.com/azure/devops/pipelines/troubleshooting)
- [GitHub Actions Best Practices](https://docs.github.com/actions/learn-github-actions/best-practices)
- [Azure Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)

---

> **Tip:** En el examen, lee cuidadosamente los requisitos. Muchas preguntas tienen múltiples soluciones correctas, pero solo una es la "mejor" según el contexto.
