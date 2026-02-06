# Prácticas Rápidas AZ-400

Ejercicios prácticos enfocados en las áreas más importantes del examen.

## 🎯 Práctica 1: Pipeline CI/CD Completo (2h)

### Objetivo
Crear un pipeline completo desde código hasta producción.

### Pasos
1. **Crear aplicación simple** (Node.js/Python/C#)
2. **Pipeline CI en GitHub Actions:**
   ```yaml
   name: CI
   on: [push, pull_request]
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Run tests
           run: npm test
         - name: Build
           run: npm run build
   ```
3. **Pipeline CD en Azure DevOps:**
   - Crear stages: dev, staging, prod
   - Configurar aprobaciones manuales
   - Desplegar a Azure App Service

### Entregable
Pipeline funcional con al menos 2 stages y aprobaciones.

---

## 🔒 Práctica 2: DevSecOps Pipeline (1.5h)

### Objetivo
Integrar seguridad en el pipeline.

### Pasos
1. **Escaneo de código estático (SAST):**
   - Integrar SonarCloud o CodeQL
2. **Escaneo de dependencias:**
   - Configurar Dependabot (GitHub)
   - O WhiteSource Bolt (Azure DevOps)
3. **Secret scanning:**
   - Habilitar GitHub Advanced Security
   - O usar Azure DevOps secret detection
4. **Escaneo de contenedores:**
   ```yaml
   - name: Scan Docker image
     uses: aquasecurity/trivy-action@master
     with:
       image-ref: myapp:latest
   ```

### Entregable
Pipeline que falla si detecta vulnerabilidades críticas.

---

## 🏗️ Práctica 3: IaC con Bicep (1.5h)

### Objetivo
Desplegar infraestructura usando código.

### Pasos
1. **Crear archivo Bicep:**
   ```bicep
   param location string = resourceGroup().location
   param appName string

   resource appService 'Microsoft.Web/sites@2022-03-01' = {
     name: appName
     location: location
     properties: {
       serverFarmId: appServicePlan.id
     }
   }
   ```
2. **Integrar en pipeline:**
   ```yaml
   - task: AzureCLI@2
     inputs:
       azureSubscription: 'connection'
       scriptType: 'bash'
       scriptLocation: 'inlineScript'
       inlineScript: |
         az deployment group create \
           --resource-group myRG \
           --template-file main.bicep
   ```
3. **Validar antes de desplegar:**
   - Usar `az deployment group validate`

### Entregable
Infraestructura desplegada automáticamente desde pipeline.

---

## 📊 Práctica 4: Observabilidad Completa (1.5h)

### Objetivo
Configurar monitoreo y alertas.

### Pasos
1. **Integrar Application Insights:**
   - Añadir SDK a la aplicación
   - Configurar connection string desde Key Vault
2. **Crear dashboard personalizado:**
   - Métricas: response time, error rate, requests/sec
   - Logs: excepciones, traces
3. **Configurar alertas:**
   - Error rate > 5%
   - Response time > 2s
   - Availability < 99%
4. **Implementar health checks:**
   ```csharp
   app.MapHealthChecks("/health");
   ```

### Entregable
Dashboard funcional con alertas configuradas.

---

## 🐳 Práctica 5: Despliegue en AKS (2h)

### Objetivo
Desplegar aplicación containerizada en Kubernetes.

### Pasos
1. **Crear Dockerfile:**
   ```dockerfile
   FROM node:18-alpine
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci --only=production
   COPY . .
   EXPOSE 3000
   CMD ["node", "server.js"]
   ```
2. **Publicar en ACR:**
   ```bash
   az acr build --registry myacr --image myapp:v1 .
   ```
3. **Crear manifiestos K8s:**
   - Deployment con readiness/liveness probes
   - Service (LoadBalancer)
   - HPA (autoscaling)
4. **Implementar estrategia canary:**
   - Usar Flagger o Argo Rollouts
5. **Integrar en pipeline:**
   ```yaml
   - task: KubernetesManifest@0
     inputs:
       action: 'deploy'
       manifests: 'k8s/*.yaml'
   ```

### Entregable
Aplicación corriendo en AKS con autoscaling y health checks.

---

## 🔄 Práctica 6: GitHub Actions Avanzado (1h)

### Objetivo
Dominar características avanzadas de GitHub Actions.

### Pasos
1. **Matrix builds:**
   ```yaml
   strategy:
     matrix:
       os: [ubuntu-latest, windows-latest]
       node: [16, 18, 20]
   ```
2. **Reusable workflows:**
   - Crear workflow reutilizable
   - Llamarlo desde otro workflow
3. **Environments con protección:**
   - Configurar environment "production"
   - Añadir required reviewers
4. **Secrets y variables:**
   - Usar GitHub Secrets
   - Variables de environment

### Entregable
Workflow modular y reutilizable con múltiples environments.

---

## 📋 Práctica 7: Terraform + Azure (1.5h)

### Objetivo
Alternativa a Bicep usando Terraform.

### Pasos
1. **Crear configuración Terraform:**
   ```hcl
   resource "azurerm_app_service" "main" {
     name                = var.app_name
     location            = var.location
     resource_group_name = azurerm_resource_group.main.name
     app_service_plan_id = azurerm_app_service_plan.main.id
   }
   ```
2. **Configurar backend remoto:**
   - Usar Azure Storage para state
3. **Integrar en pipeline:**
   - terraform init
   - terraform plan
   - terraform apply (con aprobación)
4. **Implementar workspaces:**
   - dev, staging, prod

### Entregable
Infraestructura multi-ambiente con Terraform.

---

## 🎓 Práctica Final: Proyecto Completo (4h)

### Objetivo
Integrar todo lo aprendido en un proyecto real.

### Requisitos
1. **Aplicación web** (cualquier stack)
2. **Repositorio GitHub** con:
   - Branch protection rules
   - CODEOWNERS
   - PR templates
3. **CI/CD completo:**
   - GitHub Actions para CI
   - Azure Pipelines para CD
   - Multi-stage (dev → staging → prod)
4. **IaC:**
   - Bicep o Terraform
   - Módulos reutilizables
5. **Seguridad:**
   - Escaneo SAST
   - Escaneo de dependencias
   - Secret management con Key Vault
6. **Contenedores:**
   - Dockerfile optimizado
   - Publicación en ACR
   - Despliegue en AKS
7. **Observabilidad:**
   - Application Insights
   - Dashboard personalizado
   - Alertas configuradas
8. **Documentación:**
   - README con arquitectura
   - Diagramas de pipeline
   - Instrucciones de despliegue

### Entregable
Repositorio completo con todo funcionando end-to-end.

---

## ✅ Checklist de Preparación

Antes del examen, asegúrate de poder hacer esto sin ayuda:

- [ ] Crear pipeline YAML desde cero (Azure DevOps)
- [ ] Crear workflow GitHub Actions desde cero
- [ ] Escribir template Bicep para App Service + SQL
- [ ] Configurar multi-stage pipeline con aprobaciones
- [ ] Integrar Key Vault en pipeline
- [ ] Configurar branch policies avanzadas
- [ ] Crear y publicar imagen Docker en ACR
- [ ] Desplegar aplicación en AKS
- [ ] Configurar Application Insights
- [ ] Crear dashboard en Azure Monitor
- [ ] Implementar estrategia de branching (trunk-based)
- [ ] Configurar escaneo de seguridad en pipeline
- [ ] Usar service connections y variable groups
- [ ] Implementar rollback automático
- [ ] Configurar GitHub Environments con protección

---

## 🔗 Recursos Prácticos

- [Azure DevOps Demo Generator](https://azuredevopsdemogenerator.azurewebsites.net/)
- [GitHub Skills](https://skills.github.com/)
- [Azure Bicep Playground](https://bicepdemo.z22.web.core.windows.net/)
- [Terraform Azure Examples](https://github.com/hashicorp/terraform-provider-azurerm/tree/main/examples)
- [AKS Workshop](https://docs.microsoft.com/azure/aks/tutorial-kubernetes-prepare-app)

---

> **Tip:** Practica cada ejercicio al menos 2 veces. La primera vez siguiendo la guía, la segunda sin ayuda.
