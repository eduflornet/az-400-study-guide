# Semana 6: Monitoreo, Observabilidad y Contenedores

## Observabilidad vs Monitoreo

### Monitoreo
- **Qué:** Recolectar métricas predefinidas
- **Cuándo:** Problemas conocidos
- **Ejemplo:** CPU > 80%, Memory > 90%

### Observabilidad
- **Qué:** Entender el sistema desde sus salidas
- **Cuándo:** Problemas desconocidos
- **Componentes:** Logs, Métricas, Traces

```
Observabilidad = Logs + Métricas + Traces Distribuidas
```

## Los Tres Pilares de Observabilidad

### 1. Logs (Registros)

Eventos discretos con timestamp.

```javascript
// Structured logging
logger.info('User logged in', {
  userId: '123',
  timestamp: new Date(),
  ipAddress: '192.168.1.1',
  userAgent: 'Mozilla/5.0'
});
```

### 2. Metrics (Métricas)

Valores numéricos agregados en el tiempo.

```javascript
// Métricas personalizadas
telemetryClient.trackMetric({
  name: 'OrderValue',
  value: 99.99,
  properties: {
    currency: 'USD',
    region: 'US-East'
  }
});
```

### 3. Traces (Trazas Distribuidas)

Seguimiento de requests a través de servicios.

```javascript
// Distributed tracing
const span = tracer.startSpan('process-order');
span.setAttributes({
  'order.id': orderId,
  'user.id': userId
});
// ... procesamiento
span.end();
```

## Azure Monitor

### Componentes

```
Azure Monitor
├── Application Insights (APM)
├── Log Analytics (Logs)
├── Metrics (Time-series data)
├── Alerts (Notificaciones)
└── Dashboards (Visualización)
```

### Application Insights

#### Integración en Aplicación

**Node.js:**
```javascript
const appInsights = require('applicationinsights');
appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)
  .setAutoDependencyCorrelation(true)
  .setAutoCollectRequests(true)
  .setAutoCollectPerformance(true, true)
  .setAutoCollectExceptions(true)
  .setAutoCollectDependencies(true)
  .setAutoCollectConsole(true)
  .setUseDiskRetryCaching(true)
  .setSendLiveMetrics(true)
  .start();

const client = appInsights.defaultClient;

// Custom event
client.trackEvent({
  name: 'UserPurchase',
  properties: {
    userId: '123',
    amount: 99.99
  }
});

// Custom metric
client.trackMetric({
  name: 'QueueLength',
  value: 42
});

// Dependency
client.trackDependency({
  target: 'https://api.example.com',
  name: 'GET /users',
  data: 'https://api.example.com/users',
  duration: 120,
  resultCode: 200,
  success: true
});
```

**.NET:**
```csharp
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.Extensibility;

public class MyService
{
    private readonly TelemetryClient _telemetryClient;

    public MyService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public async Task ProcessOrder(Order order)
    {
        using var operation = _telemetryClient.StartOperation<RequestTelemetry>("ProcessOrder");
        
        try
        {
            // Procesar orden
            _telemetryClient.TrackEvent("OrderProcessed", new Dictionary<string, string>
            {
                { "OrderId", order.Id },
                { "Amount", order.Total.ToString() }
            });
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            throw;
        }
    }
}
```

#### Configurar desde Pipeline

```yaml
- task: AzureCLI@2
  displayName: 'Create Application Insights'
  inputs:
    azureSubscription: 'connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # Crear Application Insights
      az monitor app-insights component create \
        --app myapp-insights \
        --location eastus \
        --resource-group myRG \
        --application-type web
      
      # Obtener connection string
      connectionString=$(az monitor app-insights component show \
        --app myapp-insights \
        --resource-group myRG \
        --query connectionString -o tsv)
      
      # Configurar en App Service
      az webapp config appsettings set \
        --name myapp \
        --resource-group myRG \
        --settings APPLICATIONINSIGHTS_CONNECTION_STRING="$connectionString"
```

### Log Analytics (KQL)

#### Queries Útiles

```kql
// Errores en las últimas 24 horas
exceptions
| where timestamp > ago(24h)
| summarize count() by type, outerMessage
| order by count_ desc

// Response time percentiles
requests
| where timestamp > ago(1h)
| summarize 
    p50 = percentile(duration, 50),
    p90 = percentile(duration, 90),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
  by bin(timestamp, 5m)
| render timechart

// Dependency failures
dependencies
| where success == false
| summarize count() by name, resultCode, type
| order by count_ desc

// User journey (distributed trace)
requests
| where operation_Id == "specific-operation-id"
| union dependencies
| project timestamp, itemType, name, duration, success
| order by timestamp asc

// Availability
availabilityResults
| where timestamp > ago(7d)
| summarize 
    availability = 100.0 * count(success == true) / count()
  by bin(timestamp, 1h), location
| render timechart

// Custom events analysis
customEvents
| where name == "UserPurchase"
| extend amount = todouble(customDimensions.amount)
| summarize 
    totalRevenue = sum(amount),
    avgOrderValue = avg(amount),
    orderCount = count()
  by bin(timestamp, 1d)
```

### Alertas

#### Crear Alerta de Métrica

```yaml
- task: AzureCLI@2
  displayName: 'Create Alerts'
  inputs:
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # Alerta: Error rate > 5%
      az monitor metrics alert create \
        --name 'high-error-rate' \
        --resource-group myRG \
        --scopes /subscriptions/.../myApp \
        --condition "avg exceptions/request > 0.05" \
        --window-size 5m \
        --evaluation-frequency 1m \
        --action-group myActionGroup \
        --severity 2
      
      # Alerta: Response time > 2s
      az monitor metrics alert create \
        --name 'slow-response' \
        --resource-group myRG \
        --scopes /subscriptions/.../myApp \
        --condition "avg requests/duration > 2000" \
        --window-size 5m \
        --evaluation-frequency 1m \
        --action-group myActionGroup \
        --severity 3
      
      # Alerta: Availability < 99%
      az monitor metrics alert create \
        --name 'low-availability' \
        --resource-group myRG \
        --scopes /subscriptions/.../myApp \
        --condition "avg availabilityResults/availabilityPercentage < 99" \
        --window-size 15m \
        --evaluation-frequency 5m \
        --action-group myActionGroup \
        --severity 1
```

#### Alerta de Log (KQL)

```yaml
# Crear alerta basada en query KQL
az monitor scheduled-query create \
  --name 'multiple-failures' \
  --resource-group myRG \
  --scopes /subscriptions/.../myApp \
  --condition "count > 10" \
  --condition-query "exceptions | where timestamp > ago(5m) | count" \
  --evaluation-frequency 5m \
  --window-size 5m \
  --action-groups myActionGroup \
  --severity 2
```

### Dashboards

#### Dashboard como Código

```json
{
  "properties": {
    "lenses": [
      {
        "order": 0,
        "parts": [
          {
            "position": { "x": 0, "y": 0, "colSpan": 6, "rowSpan": 4 },
            "metadata": {
              "type": "Extension/AppInsightsExtension/PartType/MetricsChartPart",
              "settings": {
                "content": {
                  "metrics": [
                    {
                      "resourceId": "/subscriptions/.../myApp",
                      "name": "requests/count"
                    }
                  ]
                }
              }
            }
          }
        ]
      }
    ]
  }
}
```

## Contenedores y Docker

### Conceptos Fundamentales

```
Imagen → Template inmutable
Contenedor → Instancia en ejecución de una imagen
Registry → Repositorio de imágenes (ACR, Docker Hub)
```

### Dockerfile Best Practices

```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder
WORKDIR /app

# Copiar solo package files primero (cache layer)
COPY package*.json ./
RUN npm ci --only=production

# Copiar código fuente
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app

# Copiar solo lo necesario
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

# Security: usuario no-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node healthcheck.js || exit 1

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Azure Container Registry (ACR)

#### Crear y Configurar ACR

```bash
# Crear ACR
az acr create \
  --resource-group myRG \
  --name myacr \
  --sku Standard \
  --admin-enabled false

# Build y push en un comando
az acr build \
  --registry myacr \
  --image myapp:v1 \
  --image myapp:latest \
  --file Dockerfile \
  .

# Habilitar Content Trust (image signing)
az acr config content-trust update \
  --registry myacr \
  --status enabled

# Habilitar vulnerability scanning
az acr task create \
  --registry myacr \
  --name securityScan \
  --image myapp:{{.Run.ID}} \
  --cmd "mcr.microsoft.com/azuredocs/azure-defender-scan:latest" \
  --context /dev/null
```

#### Integrar ACR en Pipeline

```yaml
# Azure Pipelines
- task: Docker@2
  displayName: 'Build and Push'
  inputs:
    containerRegistry: 'ACR-Connection'
    repository: 'myapp'
    command: 'buildAndPush'
    Dockerfile: '**/Dockerfile'
    tags: |
      $(Build.BuildId)
      latest

# Escanear imagen
- task: AzureCLI@2
  displayName: 'Scan Image'
  inputs:
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az acr repository show-tags \
        --name myacr \
        --repository myapp \
        --orderby time_desc \
        --output table
```

## Azure Kubernetes Service (AKS)

### Crear Cluster AKS

```bash
# Crear AKS con monitoring
az aks create \
  --resource-group myRG \
  --name myAKS \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --enable-addons monitoring \
  --enable-managed-identity \
  --attach-acr myacr \
  --network-plugin azure \
  --generate-ssh-keys

# Obtener credenciales
az aks get-credentials \
  --resource-group myRG \
  --name myAKS
```

### Deployment con Health Checks

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myacr.azurecr.io/myapp:latest
        ports:
        - containerPort: 3000
          name: http
        
        # Environment variables
        env:
        - name: NODE_ENV
          value: "production"
        - name: APPLICATIONINSIGHTS_CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: appinsights-connection
        
        # Resource limits
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        
        # Liveness probe (¿está vivo?)
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Readiness probe (¿listo para tráfico?)
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Startup probe (para apps lentas)
        startupProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 30

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
    protocol: TCP

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
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Estrategias de Despliegue en AKS

#### 1. Rolling Update (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Pods extra durante update
      maxUnavailable: 0  # Pods que pueden estar down
```

#### 2. Blue-Green con Services

```yaml
# Blue deployment (actual)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue

---
# Green deployment (nuevo)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green

---
# Service apunta a blue inicialmente
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Cambiar a green para switch
  ports:
  - port: 80
    targetPort: 3000
```

#### 3. Canary con Flagger

```yaml
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
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    webhooks:
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary/"
```

### Desplegar a AKS desde Pipeline

```yaml
# Azure Pipelines
- task: KubernetesManifest@0
  displayName: 'Deploy to AKS'
  inputs:
    action: 'deploy'
    kubernetesServiceConnection: 'AKS-Connection'
    namespace: 'production'
    manifests: |
      k8s/deployment.yaml
      k8s/service.yaml
      k8s/hpa.yaml
    containers: 'myacr.azurecr.io/myapp:$(Build.BuildId)'

# GitHub Actions
- name: Deploy to AKS
  uses: Azure/k8s-deploy@v4
  with:
    manifests: |
      k8s/deployment.yaml
      k8s/service.yaml
    images: myacr.azurecr.io/myapp:${{ github.sha }}
    namespace: production
```

## Feedback Loops

### Integrar Monitoreo en Pipeline

```yaml
# Validar deployment con métricas
- task: AzureCLI@2
  displayName: 'Validate Deployment'
  inputs:
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # Esperar 2 minutos
      sleep 120
      
      # Query error rate
      errorRate=$(az monitor app-insights metrics show \
        --app myapp-insights \
        --resource-group myRG \
        --metric 'exceptions/request' \
        --aggregation avg \
        --interval PT5M \
        --query 'value.timeseries[0].data[-1].average' -o tsv)
      
      # Fallar si error rate > 5%
      if (( $(echo "$errorRate > 0.05" | bc -l) )); then
        echo "Error rate $errorRate exceeds threshold"
        exit 1
      fi
```

## Recursos de Estudio
- [Azure Monitor](https://learn.microsoft.com/es-es/azure/azure-monitor/)
- [Application Insights](https://learn.microsoft.com/es-es/azure/azure-monitor/app/app-insights-overview)
- [AKS](https://learn.microsoft.com/es-es/azure/aks/)
- [KQL Reference](https://learn.microsoft.com/es-es/azure/data-explorer/kusto/query/)

## Ejercicio Práctico

### Parte 1: Application Insights (2h)
1. Crear Application Insights
2. Integrar en aplicación
3. Generar telemetría personalizada
4. Crear queries KQL
5. Configurar alertas

### Parte 2: Contenedores (2h)
1. Crear Dockerfile optimizado
2. Build y push a ACR
3. Escanear vulnerabilidades
4. Implementar health checks
5. Multi-stage build

### Parte 3: AKS (2h)
1. Crear cluster AKS
2. Desplegar aplicación
3. Configurar HPA
4. Implementar rolling update
5. Monitorear con Container Insights

## Preguntas de Repaso

1. ¿Cuál es la diferencia entre liveness y readiness probe?
2. ¿Qué son los tres pilares de observabilidad?
3. ¿Cómo creas una alerta basada en query KQL?
4. ¿Cuándo usarías blue-green vs canary deployment?
5. ¿Qué es HPA y cómo funciona?
6. ¿Cómo integras ACR con AKS?

---

> **Tip:** En el examen, conocer KQL y cómo configurar observabilidad completa es crucial. Practica queries y entiende las estrategias de despliegue en AKS.
