# Semana 3: Integración Continua (CI)

## ¿Qué es Integración Continua?
Integración Continua (CI) es la práctica de construir y probar código automáticamente cada vez que se realiza un cambio. Esto ayuda a detectar errores temprano y mejora la calidad del software.

## Conceptos Clave

### Principios de CI
1. **Integración frecuente:** Commits múltiples veces al día
2. **Build automatizado:** Cada commit dispara un build
3. **Tests automatizados:** Validación automática de calidad
4. **Feedback rápido:** Resultados en minutos, no horas
5. **Fix rápido:** Prioridad máxima si el build falla

### Componentes de un Pipeline CI
- **Source:** Repositorio de código
- **Trigger:** Evento que inicia el pipeline
- **Agent:** Máquina que ejecuta el pipeline
- **Steps/Tasks:** Acciones a ejecutar
- **Artifacts:** Salidas del build

## Azure Pipelines

### YAML vs Classic Pipelines

| Característica | YAML | Classic |
|----------------|------|---------|
| Versionado | ✅ En repo | ❌ En Azure DevOps |
| Code review | ✅ Via PR | ❌ No |
| Reutilización | ✅ Templates | ⚠️ Task groups |
| Portabilidad | ✅ Alta | ❌ Baja |
| Recomendado | ✅ Sí | ❌ Legacy |

**Conclusión:** Siempre usa YAML para nuevos pipelines.

### Estructura Básica de Pipeline YAML

```yaml
# azure-pipelines.yml

# Trigger: cuándo ejecutar
trigger:
  branches:
    include:
    - main
    - develop
  paths:
    exclude:
    - docs/**
    - README.md

# Pool: dónde ejecutar
pool:
  vmImage: 'ubuntu-latest'

# Variables
variables:
  buildConfiguration: 'Release'
  dotnetVersion: '7.x'

# Stages, Jobs, Steps
stages:
- stage: Build
  jobs:
  - job: BuildJob
    steps:
    - task: UseDotNet@2
      displayName: 'Install .NET'
      inputs:
        version: $(dotnetVersion)
    
    - task: DotNetCoreCLI@2
      displayName: 'Restore dependencies'
      inputs:
        command: 'restore'
    
    - task: DotNetCoreCLI@2
      displayName: 'Build'
      inputs:
        command: 'build'
        arguments: '--configuration $(buildConfiguration)'
    
    - task: DotNetCoreCLI@2
      displayName: 'Run tests'
      inputs:
        command: 'test'
        arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'
    
    - task: PublishCodeCoverageResults@1
      displayName: 'Publish coverage'
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '$(Agent.TempDirectory)/**/*coverage.cobertura.xml'
    
    - task: PublishBuildArtifacts@1
      displayName: 'Publish artifacts'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'
```

### Tipos de Triggers

#### Branch Triggers
```yaml
trigger:
  branches:
    include:
    - main
    - releases/*
    exclude:
    - experimental/*
```

#### Path Triggers
```yaml
trigger:
  paths:
    include:
    - src/**
    exclude:
    - docs/**
    - '*.md'
```

#### PR Triggers
```yaml
pr:
  branches:
    include:
    - main
  paths:
    exclude:
    - docs/**
```

#### Scheduled Triggers
```yaml
schedules:
- cron: "0 0 * * *"  # Diario a medianoche
  displayName: Nightly build
  branches:
    include:
    - main
  always: true  # Ejecutar aunque no haya cambios
```

### Agents

#### Microsoft-Hosted Agents
```yaml
pool:
  vmImage: 'ubuntu-latest'  # Ubuntu
  # vmImage: 'windows-latest'  # Windows
  # vmImage: 'macOS-latest'    # macOS
```

**Ventajas:**
- Sin mantenimiento
- Siempre actualizados
- Múltiples OS disponibles

**Desventajas:**
- Tiempo limitado (6 horas)
- No persisten datos
- Pueden ser lentos

#### Self-Hosted Agents
```yaml
pool:
  name: 'MyAgentPool'
```

**Ventajas:**
- Control total
- Más rápidos (cache local)
- Sin límite de tiempo
- Acceso a recursos internos

**Desventajas:**
- Requieren mantenimiento
- Costos de infraestructura

### Variables

#### Variables Simples
```yaml
variables:
  buildConfiguration: 'Release'
  projectName: 'MyApp'
```

#### Variable Groups
```yaml
variables:
- group: 'shared-variables'
- name: 'specificVar'
  value: 'value'
```

#### Variables en Runtime
```yaml
steps:
- script: echo "##vso[task.setvariable variable=myVar]myValue"
- script: echo $(myVar)
```

#### Variables Secretas
```yaml
variables:
- group: 'secrets'  # Desde Azure Key Vault

steps:
- script: echo $(secretPassword)
  env:
    SECRET: $(secretPassword)
```

### Templates Reutilizables

#### Template de Steps
```yaml
# templates/build-steps.yml
parameters:
- name: buildConfiguration
  type: string
  default: 'Release'

steps:
- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    arguments: '--configuration ${{ parameters.buildConfiguration }}'
```

#### Usar Template
```yaml
# azure-pipelines.yml
steps:
- template: templates/build-steps.yml
  parameters:
    buildConfiguration: 'Release'
```

#### Template de Job
```yaml
# templates/build-job.yml
parameters:
- name: jobName
  type: string
- name: pool
  type: string

jobs:
- job: ${{ parameters.jobName }}
  pool:
    vmImage: ${{ parameters.pool }}
  steps:
  - script: echo "Building on ${{ parameters.pool }}"
```

### Matrix Strategy

```yaml
strategy:
  matrix:
    Linux:
      imageName: 'ubuntu-latest'
    Windows:
      imageName: 'windows-latest'
    macOS:
      imageName: 'macOS-latest'
  maxParallel: 3

pool:
  vmImage: $(imageName)

steps:
- script: echo "Building on $(imageName)"
```

## GitHub Actions

### Estructura Básica

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm test
    
    - name: Build
      run: npm run build
    
    - name: Upload artifact
      uses: actions/upload-artifact@v3
      with:
        name: build
        path: dist/
```

### Matrix Strategy en GitHub Actions

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node }}
    - run: npm test
```

### Reusable Workflows

```yaml
# .github/workflows/build-template.yml
name: Build Template

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: ${{ inputs.node-version }}
    - run: npm ci
    - run: npm run build
```

```yaml
# .github/workflows/main.yml
jobs:
  build-app:
    uses: ./.github/workflows/build-template.yml
    with:
      node-version: '18'
```

## Pruebas Automatizadas

### Tipos de Pruebas

1. **Unit Tests:** Funciones individuales
2. **Integration Tests:** Componentes juntos
3. **E2E Tests:** Flujo completo de usuario
4. **Performance Tests:** Carga y rendimiento
5. **Security Tests:** Vulnerabilidades

### Pirámide de Testing

```
        /\
       /E2E\      (Pocas, lentas, costosas)
      /------\
     /Integr.\   (Algunas, medianas)
    /----------\
   /   Unit     \ (Muchas, rápidas, baratas)
  /--------------\
```

### Cobertura de Código

```yaml
# Publicar cobertura en Azure Pipelines
- task: PublishCodeCoverageResults@1
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: '$(System.DefaultWorkingDirectory)/**/coverage.xml'
    failIfCoverageEmpty: true

# Configurar umbral mínimo
- script: |
    coverage=$(grep -oP 'line-rate="\K[^"]+' coverage.xml)
    if (( $(echo "$coverage < 0.80" | bc -l) )); then
      echo "Coverage $coverage is below 80%"
      exit 1
    fi
```

### Test Reporting

```yaml
# Azure Pipelines
- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'JUnit'
    testResultsFiles: '**/test-results.xml'
    failTaskOnFailedTests: true

# GitHub Actions
- name: Publish Test Results
  uses: EnricoMi/publish-unit-test-result-action@v2
  if: always()
  with:
    files: '**/test-results.xml'
```

## Gestión de Artifacts

### Publicar Artifacts

```yaml
# Azure Pipelines
- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'

# GitHub Actions
- uses: actions/upload-artifact@v3
  with:
    name: build-artifact
    path: dist/
    retention-days: 30
```

### Consumir Artifacts

```yaml
# Azure Pipelines
- task: DownloadBuildArtifacts@1
  inputs:
    buildType: 'current'
    artifactName: 'drop'
    downloadPath: '$(System.ArtifactsDirectory)'

# GitHub Actions
- uses: actions/download-artifact@v3
  with:
    name: build-artifact
    path: dist/
```

## Optimización de Pipelines

### Cache de Dependencias

```yaml
# Azure Pipelines - npm
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    path: $(npm_config_cache)
    restoreKeys: |
      npm | "$(Agent.OS)"

# GitHub Actions - npm
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### Parallel Jobs

```yaml
jobs:
- job: UnitTests
  steps:
  - script: npm run test:unit

- job: IntegrationTests
  steps:
  - script: npm run test:integration

- job: Lint
  steps:
  - script: npm run lint
```

### Conditional Execution

```yaml
# Solo en main
- script: echo "Deploying"
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')

# Solo si tests pasaron
- script: echo "Publishing"
  condition: succeeded()

# Siempre ejecutar (cleanup)
- script: echo "Cleanup"
  condition: always()
```

## Recursos de Estudio
- [Azure Pipelines](https://learn.microsoft.com/es-es/azure/devops/pipelines/)
- [GitHub Actions](https://docs.github.com/es/actions)
- [YAML Schema](https://learn.microsoft.com/es-es/azure/devops/pipelines/yaml-schema/)

## Ejercicio Práctico

### Parte 1: Pipeline CI Básico (2h)
1. Crear aplicación simple (Node.js o .NET)
2. Agregar tests unitarios
3. Crear pipeline YAML en Azure DevOps
4. Configurar triggers
5. Publicar artifacts

### Parte 2: GitHub Actions (2h)
1. Migrar misma aplicación a GitHub
2. Crear workflow CI
3. Implementar matrix strategy
4. Configurar cache
5. Publicar artifacts

### Parte 3: Optimización (2h)
1. Implementar parallel jobs
2. Agregar cache de dependencias
3. Crear templates reutilizables
4. Configurar cobertura de código
5. Optimizar tiempos de ejecución

## Preguntas de Repaso

1. ¿Cuál es la diferencia entre Stage, Job y Step?
2. ¿Cuándo usarías self-hosted agents vs Microsoft-hosted?
3. ¿Cómo implementas cache de dependencias?
4. ¿Qué es un template y cuándo lo usarías?
5. ¿Cómo configuras un pipeline para ejecutar solo en cambios de archivos específicos?
6. ¿Cuál es la diferencia entre trigger y pr trigger?

---

> **Tip:** En el examen, conocer la sintaxis YAML y cómo estructurar pipelines eficientemente es crucial. Practica escribir pipelines desde cero.
