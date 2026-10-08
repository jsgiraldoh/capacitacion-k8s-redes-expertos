# Módulo 5 — Almacenamiento y Persistencia

Ejemplos de la Sesión 5, en el mismo orden que la presentación.

La idea del recorrido es sencilla: primero se comprueba que los volúmenes
efímeros **pierden los datos**, y solo entonces se introduce PV y PVC como
la solución.

---

## 01-efimeros/ — Volúmenes que NO persisten

| Archivo | Qué demuestra |
|---|---|
| `01-emptydir-sidecar.yaml` | Dos contenedores del mismo Pod comparten un directorio. Al borrar el Pod, el contenido desaparece. |
| `02-hostpath-logs.yaml` | Un agente lee `/var/log` del nodo. Útil para monitoreo, inservible para datos de aplicación. |

```bash
kubectl apply -f 01-efimeros/01-emptydir-sidecar.yaml
kubectl exec emptydir-sidecar -c nginx -- cat /usr/share/nginx/html/index.html
kubectl delete pod emptydir-sidecar     # el dato se pierde
```

---

## 02-pv-pvc/ — Persistencia real

Aplicar **en orden**:

| Archivo | Objeto | Papel |
|---|---|---|
| `01-pv-estatico.yaml` | PersistentVolume | El recurso. Lo declara el administrador. |
| `02-pvc-estatico.yaml` | PersistentVolumeClaim | La solicitud. La hace el usuario. |
| `03-deployment-con-pvc.yaml` | Deployment + Service | El consumidor. Monta el PVC. |
| `04-pvc-dinamico.yaml` | PersistentVolumeClaim | El contraste: sin PV, lo crea la StorageClass. |

```bash
kubectl apply -f 02-pv-pvc/01-pv-estatico.yaml
kubectl apply -f 02-pv-pvc/02-pvc-estatico.yaml
kubectl get pv,pvc                      # debe decir Bound
kubectl apply -f 02-pv-pvc/03-deployment-con-pvc.yaml
```

### La prueba de persistencia

```bash
# Escribir un dato
kubectl exec deploy/web-persistente -- \
  sh -c 'echo "<h1>Dato guardado el $(date)</h1>" > /usr/share/nginx/html/index.html'

# Borrar el Pod (no el Deployment)
kubectl delete pod -l app=web-persistente

# Leer de nuevo: el dato sigue ahí
kubectl exec deploy/web-persistente -- cat /usr/share/nginx/html/index.html
```

---

## Conceptos que aparecen en los archivos

| Campo | Dónde | Qué decide |
|---|---|---|
| `accessModes` | PV y PVC | Cuántos nodos pueden montarlo a la vez. `ReadWriteOnce` es uno solo. |
| `storageClassName` | PV y PVC | Cómo se emparejan. Vacío o ausente = clase por defecto del clúster. |
| `persistentVolumeReclaimPolicy` | PV | Si al borrar el PVC se destruye el disco (`Delete`) o se conserva (`Retain`). |
| `selector` | PVC | Afina qué PV elegir, por etiquetas. |

---

## Advertencias

- Con `ReadWriteOnce`, **no escale el Deployment a más de una réplica**: los
  Pods adicionales quedarán en `Pending`. Para eso se usa un StatefulSet.
- El PV estático de este ejemplo usa `hostPath`, que ata los datos a un nodo
  concreto. Sirve para el laboratorio, no para producción.
- En DigitalOcean, cada PVC dinámico **crea un disco que se factura mientras
  exista**. Elimine los PVC al terminar la práctica.

---

## Diagnóstico rápido

```bash
kubectl get storageclass        # qué ofrece el clúster
kubectl get pv,pvc              # estado del enlace
kubectl describe pvc <nombre>   # los eventos del final explican los Pending
```

| Síntoma | Causa habitual |
|---|---|
| PVC en `Pending` para siempre | `storageClassName` no coincide, o no hay PV compatible |
| Pod en `Pending` con PVC `Bound` | `ReadWriteOnce` y el disco ya está tomado por otro nodo |
| Se borró el PVC y se perdieron los datos | La política era `Delete` |
