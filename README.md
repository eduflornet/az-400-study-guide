# Guía de Estudio AZ-400 (Actualizada 2024)

Guía práctica para la certificación AZ-400, con enfoque en las áreas más importantes según las actualizaciones recientes del examen. Plan de estudio de 7 semanas con 6 horas semanales (sábados y domingos).

## 🎯 Áreas Prioritarias del Examen (Actualización 2024)

### 1. CI/CD con Azure DevOps y GitHub Actions ⭐⭐⭐
- Diseño y configuración de pipelines avanzados
- Estrategias de branching (GitFlow, trunk-based, release flow)
- Automatización de pruebas (unitarias, integración, seguridad)
- Gestión de artefactos y versionado

### 2. Infraestructura como Código (IaC) ⭐⭐⭐
- Bicep, ARM templates y Terraform
- Modularización, reutilización y validación
- Integración de IaC en pipelines
- Control de drift y despliegues idempotentes

### 3. DevSecOps y Seguridad Integrada ⭐⭐⭐
- Microsoft Defender for DevOps
- Escaneo de código, dependencias y contenedores
- Azure Policy, Blueprints
- Secret management (Key Vault, GitHub Secrets)

### 4. Observabilidad y Monitoreo ⭐⭐
- Azure Monitor, Log Analytics, Application Insights
- Alertas, dashboards, métricas y trazas distribuidas
- Feedback loops automáticos

### 5. Contenedores y AKS ⭐⭐
- Azure Container Registry (ACR)
- Estrategias de despliegue: rolling, blue-green, canary
- Construcción, escaneo y despliegue de contenedores

### 6. GitHub como Plataforma Principal ⭐⭐⭐
- GitHub Actions
- GitHub Advanced Security
- GitHub Packages y Environments
- GitHub Codespaces

### 7. Gestión del Flujo de Trabajo
- Boards, work items, backlog grooming
- Métricas de flujo (lead time, cycle time)
- Automatización de políticas de repositorio y PRs

---

## 📅 Plan de Estudio Semanal

### Semana 1: Fundamentos DevOps + Setup
- [Teoría: Fundamentos DevOps](week-1-theory.md)
- **Práctica (6h):**
  - Crear cuenta gratuita Azure DevOps + GitHub
  - Crear proyecto demo en ambas plataformas
  - Configurar repositorio con GitFlow
  - Crear primer Board con work items
  - **Entregable:** Repositorio configurado con README, .gitignore, branch policies

### Semana 2: Git Avanzado + Estrategias de Branching
- [Teoría: Control de Versiones](week-2-theory.md)
- **Práctica (6h):**
  - Implementar trunk-based development
  - Configurar branch policies avanzadas
  - Automatizar PRs con templates
  - Configurar CODEOWNERS
  - **Entregable:** Flujo de trabajo completo con PRs automatizados

### Semana 3: CI con Azure Pipelines + GitHub Actions ⭐
- [Teoría: Integración Continua](week-3-theory.md)
- **Práctica (6h):**
  - Crear pipeline CI en Azure DevOps (YAML)
  - Crear workflow CI en GitHub Actions
  - Integrar pruebas unitarias y de integración
  - Configurar triggers automáticos y condicionales
  - Implementar matrix builds
  - **Entregable:** 2 pipelines CI funcionales (Azure + GitHub) con tests

### Semana 4: CD + IaC con Bicep/Terraform ⭐
- [Teoría: Entrega Continua e IaC](week-4-theory.md)
- **Práctica (6h):**
  - Crear infraestructura con Bicep (App Service + SQL)
  - Crear pipeline CD con stages (dev, staging, prod)
  - Implementar aprobaciones manuales
  - Desplegar aplicación usando IaC
  - Configurar rollback automático
  - **Entregable:** Pipeline CI/CD completo con IaC y multi-stage deployment

### Semana 5: DevSecOps + Seguridad Integrada ⭐
- [Teoría: Seguridad y Cumplimiento](week-5-theory.md)
- **Práctica (6h):**
  - Integrar Azure Key Vault en pipelines
  - Configurar Microsoft Defender for DevOps
  - Implementar escaneo de dependencias (Dependabot/WhiteSource)
  - Escaneo de contenedores con Trivy/Aqua
  - Configurar Azure Policy para compliance
  - Implementar secret scanning
  - **Entregable:** Pipeline con seguridad integrada (SAST, SCA, secret scanning)

### Semana 6: Observabilidad + Contenedores + AKS ⭐
- [Teoría: Monitoreo y Contenedores](week-6-theory.md)
- **Práctica (6h):**
  - Configurar Application Insights en aplicación
  - Crear dashboards en Azure Monitor
  - Configurar alertas inteligentes
  - Construir y publicar imagen Docker en ACR
  - Desplegar en AKS con estrategia canary
  - Implementar health checks y readiness probes
  - **Entregable:** Aplicación containerizada en AKS con observabilidad completa

### Semana 7: Integración Total + Simulación de Examen
- **Práctica (6h):**
  - Proyecto final: Aplicación completa con:
    - CI/CD en GitHub Actions + Azure Pipelines
    - IaC con Bicep
    - Seguridad integrada (escaneos automáticos)
    - Despliegue en AKS
    - Observabilidad completa
  - Realizar examen de práctica oficial
  - Revisar áreas débiles
  - **Entregable:** Proyecto completo end-to-end documentado

---

## 🚀 Guía Rápida de Práctica

Si tienes poco tiempo, enfócate en estos ejercicios prácticos:

1. **Pipeline CI/CD completo** (Azure DevOps + GitHub Actions)
2. **Despliegue con Bicep/Terraform** (automatizado en pipeline)
3. **Integrar escaneos de seguridad** (código, dependencias, contenedores)
4. **Configurar observabilidad** (Application Insights + alertas)
5. **Desplegar en AKS** con estrategia canary

## 📚 Recursos Adicionales

- **[Cheat Sheet](cheat-sheet.md)** - Comandos y snippets esenciales
- **[Prácticas Rápidas](practicas-rapidas.md)** - Ejercicios prácticos enfocados (1-2h cada uno)
- **[Escenarios de Examen](escenarios-examen.md)** - Casos tipo examen con soluciones
- [Microsoft Learn - AZ-400](https://learn.microsoft.com/certifications/exams/az-400)
- [Azure DevOps Labs](https://azuredevopslabs.com)
- [GitHub Skills](https://skills.github.com)

---

> **Nota:** El examen AZ-400 fue actualizado en 2024 con mayor énfasis en GitHub, DevSecOps y prácticas modernas. Esta guía refleja esos cambios.

> Para contenido teórico detallado, consulta los archivos markdown vinculados en cada semana.
