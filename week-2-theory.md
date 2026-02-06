# Semana 2: Control de Versiones con Git

## Fundamentos de Git
Git es un sistema de control de versiones distribuido que permite a múltiples desarrolladores trabajar en código simultáneamente. Conceptos clave incluyen:
- **Repository (Repositorio):** Almacenamiento para tu código e historial
- **Commit:** Instantánea de cambios
- **Branch (Rama):** Línea independiente de desarrollo
- **Merge:** Combinar cambios de diferentes ramas

## Comandos Git Esenciales

### Básicos
```bash
git init                    # Inicializar repositorio
git clone <url>            # Clonar repositorio
git status                 # Ver estado actual
git add <file>             # Agregar archivos al staging
git commit -m "mensaje"    # Crear commit
git push                   # Enviar cambios al remoto
git pull                   # Obtener cambios del remoto
```

### Branching
```bash
git branch                 # Listar ramas
git branch <nombre>        # Crear rama
git checkout <rama>        # Cambiar de rama
git checkout -b <rama>     # Crear y cambiar a rama
git merge <rama>           # Fusionar rama
git branch -d <rama>       # Eliminar rama
```

### Avanzados
```bash
git rebase <rama>          # Rebase interactivo
git cherry-pick <commit>   # Aplicar commit específico
git stash                  # Guardar cambios temporalmente
git reset --hard <commit>  # Resetear a commit específico
git revert <commit>        # Revertir commit
```

## Estrategias de Branching

### GitFlow
Estrategia tradicional con múltiples ramas:

```
main (producción)
  └── develop (desarrollo)
       ├── feature/nueva-funcionalidad
       ├── feature/otra-funcionalidad
       └── release/v1.0
            └── hotfix/bug-critico
```

**Ventajas:**
- Estructura clara
- Ideal para releases programados
- Separación entre desarrollo y producción

**Desventajas:**
- Complejo para equipos pequeños
- Integración menos frecuente
- No ideal para CI/CD verdadero

### Trunk-Based Development ⭐ (Recomendado)
Una sola rama principal con integraciones frecuentes:

```
main
  ├── commit 1
  ├── commit 2 (feature A)
  ├── commit 3 (feature B)
  └── commit 4 (fix)
```

**Ventajas:**
- Integración continua real
- Menos conflictos de merge
- Despliegues más frecuentes
- Ideal para CI/CD

**Prácticas:**
- Commits pequeños y frecuentes
- Feature flags para código incompleto
- Revisión de código rápida
- Tests automatizados obligatorios

### Release Flow (Microsoft)
Híbrido entre GitFlow y Trunk-Based:

```
main
  ├── release/2024.01
  ├── release/2024.02
  └── topic/feature-x
```

**Características:**
- Ramas de release de larga duración
- Ideal para productos con múltiples versiones
- Usado por equipos de Microsoft

## Branch Policies en Azure DevOps

### Políticas Esenciales
1. **Require Pull Request:**
   - Mínimo de revisores (1-2)
   - Permitir auto-aprobación (desactivar)
   - Reset votes on push

2. **Build Validation:**
   - Pipeline CI debe pasar
   - Bloquear merge si falla

3. **Status Checks:**
   - Verificaciones externas
   - Escaneos de seguridad

4. **Comment Resolution:**
   - Resolver todos los comentarios antes de merge

### Configuración Recomendada
```yaml
# Ejemplo de política para rama main
branches:
  main:
    policies:
      - require_pull_request:
          min_reviewers: 2
          dismiss_stale_reviews: true
      - require_status_checks:
          strict: true
          contexts:
            - "CI Build"
            - "Security Scan"
      - require_linear_history: true
      - block_force_push: true
```

## Pull Requests (PRs)

### Anatomía de un Buen PR
1. **Título descriptivo:** "feat: Agregar autenticación OAuth"
2. **Descripción clara:**
   - Qué cambia
   - Por qué cambia
   - Cómo probarlo
3. **Tamaño manejable:** < 400 líneas
4. **Tests incluidos**
5. **Screenshots si aplica**

### PR Templates
```markdown
## Descripción
Breve descripción de los cambios

## Tipo de cambio
- [ ] Bug fix
- [ ] Nueva funcionalidad
- [ ] Breaking change
- [ ] Documentación

## Checklist
- [ ] Tests agregados/actualizados
- [ ] Documentación actualizada
- [ ] Sin warnings de linter
- [ ] Build pasa localmente

## Screenshots (si aplica)
```

### Code Review Best Practices
**Como Autor:**
- Commits atómicos y descriptivos
- Auto-revisar antes de publicar
- Responder a comentarios rápidamente

**Como Revisor:**
- Ser constructivo, no crítico
- Enfocarse en lógica, no estilo
- Aprobar solo si entiendes el código
- Usar sugerencias de código

## CODEOWNERS

Archivo para asignar automáticamente revisores:

```
# .github/CODEOWNERS o .azuredevops/CODEOWNERS

# Global owners
* @team-leads

# Frontend
/src/frontend/** @frontend-team

# Backend
/src/backend/** @backend-team

# Infrastructure
*.bicep @devops-team
*.tf @devops-team

# Security
/security/** @security-team
```

## Git Hooks

### Pre-commit Hook
```bash
#!/bin/sh
# .git/hooks/pre-commit

# Ejecutar linter
npm run lint
if [ $? -ne 0 ]; then
  echo "Linter failed. Fix errors before committing."
  exit 1
fi

# Ejecutar tests
npm test
if [ $? -ne 0 ]; then
  echo "Tests failed. Fix tests before committing."
  exit 1
fi
```

### Pre-push Hook
```bash
#!/bin/sh
# .git/hooks/pre-push

# Verificar que no hay secretos
git diff --cached --name-only | xargs grep -i "password\|secret\|api_key"
if [ $? -eq 0 ]; then
  echo "Possible secrets detected!"
  exit 1
fi
```

## Resolución de Conflictos

### Estrategias
1. **Merge:** Preserva historial completo
2. **Rebase:** Historial lineal y limpio
3. **Squash:** Combina commits en uno

### Herramientas
- VS Code (integrado)
- Git Kraken
- Beyond Compare
- Meld

### Comandos
```bash
# Durante un conflicto
git status                 # Ver archivos en conflicto
# Editar archivos manualmente
git add <archivo>          # Marcar como resuelto
git commit                 # Completar merge

# Abortar merge
git merge --abort

# Usar versión específica
git checkout --ours <file>    # Usar nuestra versión
git checkout --theirs <file>  # Usar su versión
```

## Git en Azure DevOps

### Características Especiales
- **Work Item Linking:** Vincular commits a work items
- **Branch Policies:** Políticas avanzadas de protección
- **Pull Request Workflows:** Aprobaciones y gates
- **Forks:** Soporte para fork workflow

### Integración con Pipelines
```yaml
# Trigger en cambios de archivos específicos
trigger:
  branches:
    include:
    - main
  paths:
    include:
    - src/**
    exclude:
    - docs/**
```

## Monorepo vs Multirepo

### Monorepo
**Ventajas:**
- Código compartido fácilmente
- Refactoring atómico
- Una sola fuente de verdad

**Desventajas:**
- Repositorio grande
- CI/CD más complejo
- Permisos granulares difíciles

### Multirepo
**Ventajas:**
- Repositorios pequeños
- Ownership claro
- CI/CD independiente

**Desventajas:**
- Código duplicado
- Versionado complejo
- Dependencias entre repos

## Recursos de Estudio
- [Documentación Git](https://git-scm.com/doc)
- [Azure Repos](https://learn.microsoft.com/es-es/azure/devops/repos/)
- [Pro Git Book](https://git-scm.com/book/es/v2)
- [GitHub Flow](https://docs.github.com/es/get-started/quickstart/github-flow)

## Ejercicio Práctico

### Parte 1: Configurar Repositorio (1h)
1. Crear repositorio en Azure DevOps
2. Configurar .gitignore apropiado
3. Crear README.md
4. Configurar branch policies en main
5. Crear CODEOWNERS

### Parte 2: Workflow Completo (2h)
1. Crear rama feature/nueva-funcionalidad
2. Hacer varios commits
3. Crear Pull Request
4. Simular code review
5. Resolver conflictos (si hay)
6. Merge a main

### Parte 3: Trunk-Based (1h)
1. Implementar trunk-based development
2. Usar feature flags
3. Commits directos a main (con PR)
4. Configurar CI para validar cada commit

## Preguntas de Repaso

1. ¿Cuál es la diferencia entre merge y rebase?
2. ¿Cuándo usarías GitFlow vs Trunk-Based Development?
3. ¿Qué branch policies son esenciales para un proyecto empresarial?
4. ¿Cómo vinculas un commit a un work item en Azure DevOps?
5. ¿Qué es CODEOWNERS y por qué es importante?
6. ¿Cómo previenes que secretos lleguen al repositorio?

---

> **Tip:** En el examen, conocer las estrategias de branching y cuándo usar cada una es crucial. Trunk-Based es la respuesta correcta para CI/CD moderno.
