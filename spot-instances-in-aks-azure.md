# Guía Completa: AKS Spot VMs con Balance Similar Node Groups

## Problema Identificado

La respuesta del soporte de Azure fue básica y no incluyó recomendaciones específicas para optimizar el uso de Spot VMs en AKS. Esta guía complementa esa información con estrategias avanzadas.

## Estrategias para Optimizar Spot VMs

### 1. Arquitectura Híbrida

**Combinación de Node Pools:**
- Node pools regulares para cargas críticas
- Spot pools para workloads tolerantes a interrupciones
- Usar taints y tolerations para control granular
- Implementar node affinity/anti-affinity

**Ejemplo de configuración:**
```yaml
# Taint para nodos Spot
kubectl taint nodes spot-node-pool spot=true:NoSchedule

# Toleration en el pod
tolerations:
- key: "spot"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

### 2. Manejo de Interrupciones

**Configuraciones esenciales:**
- **PodDisruptionBudgets**: Controlar interrupciones simultáneas
- **Múltiples Spot pools**: Diferentes AZs y tipos de VM
- **Graceful shutdown**: Handlers para SIGTERM
- **PreStop hooks**: Preparación antes del cierre

**Ejemplo PodDisruptionBudget:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

### 3. Balance Similar Node Groups

#### ¿Qué es balance-similar-node-groups?

Característica del cluster autoscaler que mantiene balance entre node groups similares, distribuyéndolos equitativamente entre zonas de disponibilidad.

#### Configuración

**Habilitar en cluster autoscaler:**
```yaml
# En el deployment del cluster autoscaler
--balance-similar-node-groups=true
--nodes=1:10:nodepool-spot-eastus2a
--nodes=1:10:nodepool-spot-eastus2b  
--nodes=1:10:nodepool-spot-eastus2c
```

#### Beneficios con Spot VMs

- **Resiliencia**: Distribución entre múltiples AZs
- **Mejor distribución**: Evita concentración en una zona
- **Reducción del blast radius**: Interrupciones menos impactantes
- **Alta disponibilidad**: Continuidad de servicio durante evictions

### 4. Creación de Node Pools Multi-AZ

**Comando para crear node pools en diferentes AZs:**

```bash
# Node pool Zona 1
az aks nodepool add \
  --cluster-name myAKSCluster \
  --name spotnodes-a \
  --availability-zones 1 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5 \
  --node-vm-size Standard_D4s_v3

# Node pool Zona 2
az aks nodepool add \
  --cluster-name myAKSCluster \
  --name spotnodes-b \
  --availability-zones 2 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5 \
  --node-vm-size Standard_D4s_v3

# Node pool Zona 3
az aks nodepool add \
  --cluster-name myAKSCluster \
  --name spotnodes-c \
  --availability-zones 3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5 \
  --node-vm-size Standard_D4s_v3
```

### 5. Configuración del Cluster Autoscaler

**Parámetros recomendados:**
```yaml
--scale-down-unneeded-time=1m      # Reacción rápida
--scale-down-delay-after-add=1m    # Tiempo después de agregar nodos
--skip-nodes-with-local-storage=false  # Si es necesario
--balance-similar-node-groups=true # Distribución equilibrada
```

### 6. Estrategias de Deployment

**Deployments resilientes:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-app
spec:
  replicas: 6  # Múltiples réplicas
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 2
  template:
    spec:
      # Distribución entre zonas
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: resilient-app
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 30
      
      containers:
      - name: app
        image: myapp:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        
        # Health checks robustos
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Workloads Ideales para Spot VMs

### ✅ Recomendados
- **Procesamiento batch/ETL**
- **Aplicaciones stateless**
- **Jobs de CI/CD**
- **Análisis de datos no críticos**
- **Ambientes de desarrollo/testing**
- **Procesamiento de imágenes/video**
- **Machine Learning training jobs**

### ❌ No Recomendados
- **Bases de datos críticas**
- **Aplicaciones con estado crítico**
- **Servicios de alta disponibilidad sin réplicas**
- **Aplicaciones que requieren uptime 24/7**

## Consideraciones Importantes

### Limitaciones
- Los node pools deben ser "similares" (mismo instance type, labels, taints)
- Funciona mejor con al menos 3 zonas de disponibilidad
- Puede resultar en costos ligeramente mayores por la distribución
- No hay SLA para nodos Spot

### Mejores Prácticas
1. **Siempre usar múltiples AZs** para Spot VMs
2. **Configurar monitoring** para detectar evictions
3. **Implementar circuit breakers** en aplicaciones
4. **Usar taints específicos** para Spot nodes
5. **Configurar alertas** para cuando los nodos Spot bajan a cero

## Monitoreo y Alertas

**Métricas clave a monitorear:**
- Número de nodos Spot disponibles por AZ
- Tasa de evictions
- Tiempo de recuperación después de evictions
- Costo comparativo vs nodos regulares

**Ejemplo de alerta:**
```yaml
alert: SpotNodesLow
expr: kube_node_info{node_type="spot"} < 2
for: 5m
labels:
  severity: warning
annotations:
  summary: "Pocos nodos Spot disponibles"
```

## Conclusión

El uso efectivo de Spot VMs en AKS requiere una estrategia integral que combine:
- Arquitectura híbrida con múltiples node pools
- Distribución geográfica con balance-similar-node-groups
- Configuración apropiada de aplicaciones resilientes
- Monitoreo continuo y alertas proactivas

Esta aproximación permite obtener ahorros significativos (hasta 90%) manteniendo la disponibilidad de las aplicaciones.
