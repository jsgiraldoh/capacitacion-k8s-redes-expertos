# Metrics Server y kubectl top

**Módulo 7 · Sesión 8 · Parte 1 de la guía**

`kubectl top` no mide nada por sí mismo: le pregunta a la API
`metrics.k8s.io`, que publica el Metrics Server. Sin ese componente,
`kubectl top` falla, y el HPA de `administration/04-hpa/` también.

## Archivos

| Archivo | Para qué sirve |
|---|---|
| `metrics-patch.json` | Parche que agrega `--kubelet-insecure-tls` (solo laboratorio) |

## Paso a paso

### 1. Provocar el error a propósito

```bash
kubectl top nodes
# error: Metrics API not available
```

### 2. Comprobar si ya está instalado

```bash
kubectl get deployment metrics-server -n kube-system
kubectl get apiservice v1beta1.metrics.k8s.io
```

- `NotFound`: no está instalado, siga al paso 3.
- Aparece con `AVAILABLE True`: ya está listo, salte al paso 8.

| Clúster | Metrics Server | Qué hacer |
|---|---|---|
| Docker Desktop | Normalmente no viene | Instalar |
| Rancher Desktop (k3s) | Normalmente incluido | Verificar |
| Minikube | Addon | `minikube addons enable metrics-server` |
| Kind | No viene | Instalar y agregar el flag de TLS |
| kubeadm | No viene | Instalar y casi siempre agregar el flag de TLS |
| DOKS, EKS, GKE, AKS | Normalmente incluido | Verificar |

La columna del medio es lo habitual, no una garantía: verifique siempre.

### 3. Instalar

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 4. Ver el estado del Pod

```bash
kubectl get pods -n kube-system -l k8s-app=metrics-server
```

En clústeres locales lo normal es `READY 0/1` con `STATUS Running`.

### 5. Descubrir por qué no está listo

```bash
kubectl logs -n kube-system deployment/metrics-server
```

Error típico: `x509: cannot validate certificate ... doesn't contain any IP SANs`.
Los kubelets locales usan certificados autofirmados y Metrics Server no confía en ellos.

### 6. Aplicar el parche

```bash
kubectl patch deployment metrics-server -n kube-system --type=json --patch-file metrics-patch.json
```

> **Solo para laboratorio.** `--kubelet-insecure-tls` desactiva la verificación
> del certificado del kubelet. En producción se configuran certificados válidos.

### 7. Esperar el nuevo Pod

```bash
kubectl rollout status deployment/metrics-server -n kube-system
```

### 8. Verificar la API de métricas

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
# AVAILABLE debe decir True
```

### 9. Usar kubectl top

```bash
kubectl top nodes
kubectl top pods -A
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

Si aparece `metrics not available yet`, espere 30 a 60 segundos: es la primera recolección.

## Si algo falla

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `Metrics API not available` | No instalado o no listo | Pasos 2 y 8 |
| `metrics not available yet` | Primera recolección pendiente | Esperar 60 segundos |
| Pod en `0/1` | Error de TLS con el kubelet | Pasos 5 y 6 |
| `unable to parse` en el patch | JSON con comillas distintas | Revisar que sean comillas rectas |
