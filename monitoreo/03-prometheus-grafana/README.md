# Prometheus y Grafana

**Módulo 7 · Sesión 8 · Parte 6 de la guía (demostración del instructor)**

`kubectl top` muestra solo el instante actual. Prometheus guarda el
historial y Grafana lo dibuja.

```
Nodos y Pods → Prometheus → Grafana → Alertas
```

> **Es pesado para un clúster local.** Necesita de 2 a 3 GB de memoria
> libre. En Docker Desktop revise Settings > Resources antes de instalar.

## Archivos

| Archivo | Para qué sirve |
|---|---|
| `values-local.yaml` | Valores reducidos: sin Alertmanager y con 2 días de retención |

## Paso a paso

### 1. Agregar el repositorio de charts

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Instalar

```bash
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f values-local.yaml
kubectl get pods -n monitoring -w
```

Tarda de 2 a 5 minutos. Cuando todos los Pods estén `Running` y listos, `Ctrl+C`.

### 3. Abrir Grafana

Este comando se queda ejecutando: úselo en una terminal aparte.

```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

En el navegador: `http://localhost:3000`

- Usuario: `admin`
- Contraseña: `prom-operator`

### 4. Qué mirar

Menú **Dashboards**, luego **Kubernetes / Compute Resources / Namespace (Pods)**.
Elija un namespace y amplíe el rango de tiempo a la última hora.

Si la prueba del HPA (`administracion/04-hpa/`) fue reciente, elija `lab-hpa`:
se ve la curva de CPU subir y las réplicas aumentar.

## Las tres herramientas

| Herramienta | Qué muestra | Cuándo usarla |
|---|---|---|
| `kubectl top` | El instante actual | Diagnóstico rápido |
| Prometheus | Series de tiempo guardadas | Consultar el pasado y definir alertas |
| Grafana | Dashboards sobre Prometheus | Ver tendencias y presentarlas |

## Limpieza

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```
