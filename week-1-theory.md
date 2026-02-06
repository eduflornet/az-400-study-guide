# Semana 1: Introducción y Fundamentos de DevOps

## ¿Qué es DevOps?
DevOps es un conjunto de prácticas que combina el desarrollo de software (Dev) y las operaciones de TI (Ops). Su objetivo es acortar el ciclo de vida del desarrollo y entregar software de alta calidad de forma continua.

## Principios y Beneficios
- Colaboración entre equipos de desarrollo y operaciones
- Automatización de procesos
- Integración y entrega continua
- Tiempo de comercialización más rápido
- Mejora en confiabilidad y escalabilidad

## Visión General de Azure DevOps
Azure DevOps es un conjunto de servicios para gestionar todo el ciclo de vida DevOps, incluyendo:
- **Boards:** Seguimiento de trabajo
- **Repos:** Control de código fuente
- **Pipelines:** Automatización CI/CD
- **Test Plans:** Herramientas de pruebas
- **Artifacts:** Gestión de paquetes

## Conceptos Clave para el Examen

### Cultura DevOps
- **Colaboración:** Romper silos entre Dev y Ops
- **Automatización:** Reducir tareas manuales repetitivas
- **Medición:** Métricas para mejorar continuamente
- **Compartir:** Conocimiento y responsabilidad compartida

### Prácticas Fundamentales
1. **Integración Continua (CI):** Integrar código frecuentemente
2. **Entrega Continua (CD):** Automatizar despliegues
3. **Infraestructura como Código (IaC):** Gestionar infraestructura con código
4. **Monitoreo y Logging:** Observabilidad completa
5. **Feedback Continuo:** Aprender de producción

### Azure DevOps vs GitHub

| Característica | Azure DevOps | GitHub |
|----------------|--------------|--------|
| Repositorios | Azure Repos | GitHub Repos |
| CI/CD | Azure Pipelines | GitHub Actions |
| Gestión de trabajo | Azure Boards | GitHub Issues/Projects |
| Paquetes | Azure Artifacts | GitHub Packages |
| Seguridad | Defender for DevOps | GitHub Advanced Security |

## Métricas DevOps Importantes

### Métricas de Flujo
- **Lead Time:** Tiempo desde idea hasta producción
- **Cycle Time:** Tiempo desde inicio de desarrollo hasta producción
- **Deployment Frequency:** Frecuencia de despliegues
- **Change Failure Rate:** Porcentaje de despliegues que fallan
- **Mean Time to Recovery (MTTR):** Tiempo promedio de recuperación

### Métricas de Calidad
- Cobertura de código
- Deuda técnica
- Bugs en producción
- Tiempo de respuesta de aplicación

## Estrategias de Branching

### GitFlow
- **main/master:** Código en producción
- **develop:** Rama de desarrollo
- **feature/*:** Nuevas características
- **release/*:** Preparación de releases
- **hotfix/*:** Correcciones urgentes

### Trunk-Based Development (Recomendado para CI/CD)
- Una rama principal (trunk/main)
- Commits frecuentes y pequeños
- Feature flags para funcionalidades incompletas
- Integración continua real

### Release Flow (Microsoft)
- Similar a GitFlow pero más simple
- Ramas de release de larga duración
- Ideal para productos con múltiples versiones en producción

## Azure Boards - Gestión de Trabajo

### Tipos de Work Items
- **Epic:** Iniciativa grande
- **Feature:** Funcionalidad específica
- **User Story:** Requisito desde perspectiva del usuario
- **Task:** Tarea técnica
- **Bug:** Defecto a corregir

### Procesos Disponibles
1. **Basic:** Simple, ideal para equipos pequeños
2. **Agile:** Metodología ágil estándar
3. **Scrum:** Framework Scrum completo
4. **CMMI:** Capability Maturity Model Integration

## Recursos de Estudio
- [Microsoft Learn: Ruta AZ-400](https://learn.microsoft.com/es-es/training/paths/az-400-work-git-for-enterprise-devops/)
- [Documentación Azure DevOps](https://learn.microsoft.com/es-es/azure/devops/)
- [DevOps Resource Center](https://learn.microsoft.com/es-es/devops/)

## Ejercicio Práctico

### Configuración Inicial (30 min)
1. Crear cuenta gratuita en Azure DevOps
2. Crear organización y proyecto
3. Explorar cada servicio (Boards, Repos, Pipelines)
4. Crear primer Board con work items de ejemplo

### Configuración de Repositorio (30 min)
1. Crear repositorio Git
2. Clonar localmente
3. Crear estructura básica de proyecto
4. Hacer primer commit y push
5. Crear branch policies básicas

## Preguntas de Repaso

1. ¿Cuál es la diferencia entre Integración Continua y Entrega Continua?
2. ¿Qué estrategia de branching es mejor para CI/CD verdadero?
3. ¿Cuáles son las 4 métricas clave de DevOps según DORA?
4. ¿Cuándo usarías Azure DevOps vs GitHub?
5. ¿Qué es un feature flag y por qué es importante?

---

> **Nota:** Esta semana es fundamental para entender la filosofía DevOps. No te enfoques solo en herramientas, entiende los principios.
