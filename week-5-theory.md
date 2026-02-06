# Semana 5: Seguridad y Cumplimiento en DevOps (DevSecOps)

## ¿Qué es DevSecOps?
DevSecOps integra la seguridad en cada fase del ciclo de vida DevOps, en lugar de tratarla como una fase separada al final. "Shift Left" significa mover la seguridad hacia la izquierda (más temprano) en el proceso.

## Principios de DevSecOps

### Shift Left Security
- **Seguridad desde el diseño:** Threat modeling temprano
- **Código seguro:** SAST durante desarrollo
- **Dependencias seguras:** SCA en cada build
- **Infraestructura segura:** IaC scanning
- **Contenedores seguros:** Image scanning

### Automatización de Seguridad
- Escaneos automáticos en pipeline
- Políticas como código
- Remediación automática cuando sea posible
- Feedback inmediato a desarrolladores

## Tipos de Escaneos de Seguridad

### 1. SAST (Static Application Security Testing)

Análisis de código fuente sin ejecutarlo.

```yaml
# Azure Pipelines - SonarCloud
- task: SonarCloudPrepare@1
  inputs:
    SonarCloud: 'SonarCloud-Connection'
    organization: 'myorg'
    scannerMode: 'CLI'
    projectKey: 'myproject'

- task: DotNetCoreCLI@2
  inputs:
    command: 'build'

- task: SonarCloudAnalyze@1

- task: SonarCloudPublish@1
  inputs:
    pollingTimeoutSec: '300'

# Fallar si hay vulnerabilidades críticas
- task: sonarcloud-buildbreaker@2
  inputs:
    SonarCloud: 'SonarCloud-Connection'
```

```yaml
# GitHub Actions - CodeQL
- name: Initialize CodeQL
  uses: github/codeql-action/init@v2
  with:
    languages: javascript, python

- name: Autobuild
  uses: github/codeql-action/autobuild@v2

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v2
```

**Detecta:**
- SQL Injection
- Cross-Site Scripting (XSS)
- Buffer overflows
- Hardcoded credentials
- Insecure cryptography

### 2. SCA (Software Composition Analysis)

Análisis de dependencias de terceros.

```yaml
# GitHub - Dependabot
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
```

```yaml
# Azure Pipelines - WhiteSource Bolt
- task: WhiteSource@21
  inputs:
    cwd: '$(System.DefaultWorkingDirectory)'
    projectName: 'MyProject'
```

```yaml
# OWASP Dependency Check
- script: |
    dependency-check \
      --project "MyApp" \
      --scan . \
      --format HTML \
      --format JSON \
      --failOnCVSS 7
  displayName: 'OWASP Dependency Check'
```

**Detecta:**
- Vulnerabilidades conocidas (CVEs)
- Licencias incompatibles
- Dependencias obsoletas
- Dependencias no mantenidas

### 3. Secret Scanning

Detectar credenciales en código.

```yaml
# GitHub Advanced Security - Secret Scanning
# Habilitado automáticamente en repos públicos
# Para privados: Settings → Security → Secret scanning

# Azure DevOps - Credential Scanner
- task: CredScan@3
  inputs:
    outputFormat: 'sarif'
    debugMode: false

- task: PublishSecurityAnalysisLogs@3
  inputs:
    ArtifactName: 'CodeAnalysisLogs'
```

```yaml
# TruffleHog - Escaneo de secretos
- script: |
    docker run --rm -v $(pwd):/proj \
      trufflesecurity/trufflehog:latest \
      filesystem /proj \
      --json \
      --fail
  displayName: 'Scan for secrets'
```

**Detecta:**
- API keys
- Passwords
- Private keys
- Tokens de acceso
- Connection strings

### 4. Container Scanning

Escaneo de imágenes Docker.

```yaml
# Trivy
- script: |
    docker run --rm \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v $(pwd):/root/.cache/ \
      aquasec/trivy:latest \
      image --severity HIGH,CRITICAL \
      --exit-code 1 \
      myapp:latest
  displayName: 'Scan Docker image'
```

```yaml
# Microsoft Defender for Containers
- task: AzureCLI@2
  inputs:
    azureSubscription: 'connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az acr task create \
        --registry myacr \
        --name securityScan \
        --image myapp:{{.Run.ID}} \
        --cmd "mcr.microsoft.com/azuredocs/azure-defender-scan:latest" \
        --context /dev/null
```

**Detecta:**
- Vulnerabilidades en base images
- Malware
- Configuraciones inseguras
- Secretos en layers

### 5. DAST (Dynamic Application Security Testing)

Pruebas de seguridad en aplicación en ejecución.

```yaml
# OWASP ZAP
- script: |
    docker run --rm \
      -v $(pwd):/zap/wrk/:rw \
      -t owasp/zap2docker-stable \
      zap-baseline.py \
      -t https://myapp-staging.azurewebsites.net \
      -r zap-report.html \
      -J zap-report.json
  displayName: 'OWASP ZAP Scan'
```

**Detecta:**
- Vulnerabilidades en runtime
- Configuraciones inseguras
- Problemas de autenticación
- Inyecciones SQL/XSS en producción

## Gestión de Secretos

### Azure Key Vault

#### Crear y Usar Secretos

```bash
# Crear Key Vault
az keyvault create \
  --name myKeyVault \
  --resource-group myRG \
  --location eastus

# Agregar secreto
az keyvault secret set \
  --vault-name myKeyVault \
  --name "DatabasePassword" \
  --value "SuperSecretPassword123!"

# Dar acceso a service principal
az keyvault set-policy \
  --name myKeyVault \
  --spn <service-principal-id> \
  --secret-permissions get list
```

#### Integrar en Pipeline

```yaml
# Azure Pipelines
steps:
- task: AzureKeyVault@2
  inputs:
    azureSubscription: 'connection'
    KeyVaultName: 'myKeyVault'
    SecretsFilter: '*'  # O específicos: 'DatabasePassword,ApiKey'
    RunAsPreJob: true

- script: |
    echo "Using database password"
    # $(DatabasePassword) ahora disponible
  env:
    DB_PASSWORD: $(DatabasePassword)
```

```yaml
# GitHub Actions
- name: Get secrets from Key Vault
  uses: Azure/get-keyvault-secrets@v1
  with:
    keyvault: 'myKeyVault'
    secrets: 'DatabasePassword, ApiKey'
  id: keyvault

- name: Use secret
  env:
    DB_PASSWORD: ${{ steps.keyvault.outputs.DatabasePassword }}
  run: |
    echo "Connecting to database"
```

### GitHub Secrets

```yaml
# Configurar en: Settings → Secrets and variables → Actions

steps:
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
    DB_CONNECTION: ${{ secrets.DB_CONNECTION }}
  run: |
    echo "Deploying with secrets"
```

### Variable Groups (Azure DevOps)

```yaml
# Crear variable group vinculado a Key Vault
# Pipelines → Library → Variable groups → Link secrets from Azure Key Vault

variables:
- group: 'production-secrets'
- name: 'environment'
  value: 'prod'

steps:
- script: echo $(DatabasePassword)
```

## Azure Policy y Compliance

### Azure Policy

Políticas para asegurar compliance.

```json
// Política: Requerir tags en recursos
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "field": "tags",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

```bash
# Asignar política
az policy assignment create \
  --name 'require-tags' \
  --policy 'require-tags-policy' \
  --scope '/subscriptions/<subscription-id>/resourceGroups/myRG'
```

### Azure Blueprints

Plantillas para despliegues compliant.

```yaml
# Blueprint con políticas, roles y recursos
blueprint:
  name: "secure-webapp"
  artifacts:
    - type: policyAssignment
      name: "require-https"
    - type: roleAssignment
      name: "assign-reader"
    - type: template
      name: "webapp-template"
```

### Compliance en Pipeline

```yaml
- task: AzureCLI@2
  displayName: 'Check Policy Compliance'
  inputs:
    azureSubscription: 'connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      compliance=$(az policy state list \
        --resource-group myRG \
        --query "[?complianceState=='NonCompliant'] | length(@)")
      
      if [ $compliance -gt 0 ]; then
        echo "Found $compliance non-compliant resources"
        exit 1
      fi
```

## Microsoft Defender for DevOps

### Configuración

```yaml
# Habilitar Defender for DevOps
# Azure Portal → Microsoft Defender for Cloud → DevOps Security

# Conectar Azure DevOps o GitHub
# Settings → Connectors → Add connector

# Escaneos automáticos en cada PR
- task: MicrosoftSecurityDevOps@1
  displayName: 'Run Microsoft Security DevOps'
  inputs:
    categories: 'secrets,code,dependencies,containers'
```

### Características
- **Code scanning:** SAST integrado
- **Secret scanning:** Detectar credenciales
- **Dependency scanning:** Vulnerabilidades en dependencias
- **IaC scanning:** Bicep, Terraform, ARM
- **Container scanning:** Imágenes Docker
- **Posture management:** Configuraciones inseguras

## Seguridad en Contenedores

### Dockerfile Seguro

```dockerfile
# ❌ Inseguro
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl
COPY . /app
RUN chmod 777 /app
USER root
CMD ["node", "server.js"]

# ✅ Seguro
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

# Crear usuario no-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
USER nodejs

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Best Practices
1. **Usar imágenes oficiales y específicas:** `node:18-alpine` no `node:latest`
2. **Multi-stage builds:** Reducir tamaño y superficie de ataque
3. **Usuario no-root:** Nunca ejecutar como root
4. **Escanear imágenes:** Trivy, Aqua, Defender
5. **Firmar imágenes:** Docker Content Trust
6. **Minimal base images:** Alpine, distroless

### Image Signing

```bash
# Docker Content Trust
export DOCKER_CONTENT_TRUST=1

docker build -t myacr.azurecr.io/myapp:v1 .
docker push myacr.azurecr.io/myapp:v1
# Requiere firma con clave privada
```

## RBAC y Permisos

### Azure DevOps

```yaml
# Permisos mínimos necesarios
# Project Settings → Permissions

# Service Connection con permisos limitados
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'limited-connection'  # Solo permisos de deploy
```

### GitHub

```yaml
# Permisos granulares en workflow
permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Para OIDC
      contents: read
```

## Auditoría y Logging

### Azure DevOps Audit Logs

```bash
# Consultar audit logs
az devops invoke \
  --area audit \
  --resource log \
  --route-parameters project=MyProject \
  --query-parameters startTime=2024-01-01
```

### GitHub Audit Log

```bash
# API para audit log
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/orgs/myorg/audit-log
```

### Monitoreo de Seguridad

```yaml
# Alertas en Azure Monitor
- task: AzureCLI@2
  inputs:
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az monitor metrics alert create \
        --name 'high-failed-logins' \
        --resource-group myRG \
        --scopes /subscriptions/.../myApp \
        --condition "count failedLogins > 10" \
        --window-size 5m \
        --evaluation-frequency 1m
```

## Recursos de Estudio
- [Azure Security](https://learn.microsoft.com/es-es/azure/security/)
- [DevSecOps](https://learn.microsoft.com/es-es/azure/devops/learn/devsecops)
- [Microsoft Defender for DevOps](https://learn.microsoft.com/es-es/azure/defender-for-cloud/defender-for-devops-introduction)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

## Ejercicio Práctico

### Parte 1: Escaneos de Seguridad (2h)
1. Integrar SonarCloud o CodeQL
2. Configurar Dependabot
3. Implementar secret scanning
4. Escanear imagen Docker con Trivy
5. Configurar pipeline para fallar en vulnerabilidades críticas

### Parte 2: Key Vault (1.5h)
1. Crear Azure Key Vault
2. Agregar secretos
3. Integrar en pipeline
4. Usar secretos en aplicación
5. Rotar secretos

### Parte 3: Defender for DevOps (1.5h)
1. Habilitar Microsoft Defender for DevOps
2. Conectar repositorio
3. Revisar findings
4. Remediar vulnerabilidades
5. Configurar políticas

### Parte 4: Contenedores Seguros (1h)
1. Crear Dockerfile seguro
2. Escanear con Trivy
3. Implementar multi-stage build
4. Usuario no-root
5. Publicar en ACR con escaneo

## Preguntas de Repaso

1. ¿Cuál es la diferencia entre SAST y DAST?
2. ¿Qué es "Shift Left" en DevSecOps?
3. ¿Cómo integras Azure Key Vault en un pipeline?
4. ¿Qué herramientas usarías para escanear dependencias?
5. ¿Cómo aseguras que una imagen Docker es segura?
6. ¿Qué es Microsoft Defender for DevOps?
7. ¿Cómo previenes que secretos lleguen al repositorio?

---

> **Tip:** En el examen, la seguridad es un tema MUY importante. Conoce las herramientas de escaneo y cuándo usar cada una. Defender for DevOps es clave en 2024.
