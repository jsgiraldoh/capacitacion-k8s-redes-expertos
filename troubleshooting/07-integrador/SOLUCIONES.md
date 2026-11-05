# Soluciones del laboratorio integrador

> **No abrir antes de intentarlo.** Cada fallo corresponde a un escenario de las carpetas anteriores.

| Aplicación | Síntoma | Causa | Escenario |
|---|---|---|---|
| `tienda` | `ImagePullBackOff` | La etiqueta `1.25-alpin` está mal escrita (falta la `e`). | 01 |
| `catalogo` | Pod `Running`, `ENDPOINTS <none>` | El Service busca `app=catalogos` y los Pods son `app=catalogo`. | 07 |
| `reportes` | `CreateContainerConfigError` | Falta el ConfigMap `reportes-config` (clave `MODE`). | 04 |
| `archivos` | PVC y Pod en `Pending` | La StorageClass `rapido` no existe, y es inmutable: hay que recrear el PVC. | 10 |
| `monitor` | Pod `Running`, logs con `HTTP 403` | El ServiceAccount `sa-monitor` no tiene permisos. | 11 |
| `busquedas` | `CrashLoopBackOff` | Falta la variable `INDEX_URL`. | 02 |

## Comandos de solución

### tienda

**Causa:** La etiqueta `1.25-alpin` está mal escrita (falta la `e`).

```text
kubectl set image deployment/tienda nginx=nginx:1.25-alpine -n lab-roto
```

### catalogo

**Causa:** El Service busca `app=catalogos` y los Pods son `app=catalogo`.

```text
kubectl set selector service catalogo app=catalogo -n lab-roto
```

### reportes

**Causa:** Falta el ConfigMap `reportes-config` (clave `MODE`).

```text
kubectl create configmap reportes-config --from-literal=MODE=diario -n lab-roto
```

### archivos

**Causa:** La StorageClass `rapido` no existe, y es inmutable: hay que recrear el PVC.

```text
kubectl delete deployment archivos -n lab-roto
kubectl delete pvc archivos-pvc -n lab-roto
(editar `storageClassName` a una existente)
kubectl apply -f 01-cluster-roto.yaml
```

### monitor

**Causa:** El ServiceAccount `sa-monitor` no tiene permisos.

```text
kubectl create role lector-pods --verb=list --resource=pods -n lab-roto
kubectl create rolebinding lector-pods --role=lector-pods --serviceaccount=lab-roto:sa-monitor -n lab-roto
```

### busquedas

**Causa:** Falta la variable `INDEX_URL`.

```text
kubectl set env deployment/busquedas INDEX_URL=http://indice.interno -n lab-roto
```

## Nota sobre `archivos`

El PVC no admite cambiar `storageClassName`. Se borran el Deployment y el PVC, se edita `storageClassName` en `01-cluster-roto.yaml` con una StorageClass que exista (`kubectl get storageclass`) y se vuelve a aplicar el archivo.
