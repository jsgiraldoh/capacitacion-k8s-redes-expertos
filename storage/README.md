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
| `05-consumidor-pvc-dinamico.yaml` | Pod | Desbloquea el PVC si la clase usa `WaitForFirstConsumer`. |

```bash
kubectl apply -f 02-pv-pvc/01-pv-estatico.yaml
kubectl apply -f 02-pv-pvc/02-pvc-estatico.yaml
kubectl get pv,pvc                      # debe decir Bound
kubectl apply -f 02-pv-pvc/03-deployment-con-pvc.yaml
```

### La prueba de persistencia

**Paso 1 — escribir un dato.** La forma recomendada es entrar al contenedor,
porque funciona igual desde Windows, Linux y macOS:

```bash
kubectl exec -it deploy/web-persistente -- sh
```

Ya dentro del contenedor:

```sh
echo "<h1>Dato guardado el $(date)</h1>" > /usr/share/nginx/html/index.html
cat /usr/share/nginx/html/index.html
exit
```

**Paso 2 — borrar el Pod** (no el Deployment) y esperar al reemplazo:

```bash
kubectl delete pod -l app=web-persistente
kubectl get pods -w
```

**Paso 3 — leer de nuevo.** El dato sigue ahí:

```bash
kubectl exec deploy/web-persistente -- cat /usr/share/nginx/html/index.html
```

Compare con el ejemplo de `emptyDir`: allí el dato se perdió. Esa es toda la
diferencia entre efímero y persistente.

<details>
<summary>En una sola línea, según su sistema</summary>

**Linux / macOS:**

```bash
kubectl exec deploy/web-persistente -- \
  sh -c 'echo "<h1>Dato guardado el $(date)</h1>" > /usr/share/nginx/html/index.html'
```

**Windows (PowerShell):** lo anterior **no funciona**. PowerShell elimina las
comillas simples antes de pasar los argumentos, así que `sh` recibe `<h1>`
como un archivo y falla con `cannot open h1: No such file`. Use:

```powershell
kubectl exec deploy/web-persistente -- sh -c "date > /usr/share/nginx/html/index.html"
```

O bien `--%`, que le dice a PowerShell que deje de interpretar:

```powershell
kubectl exec deploy/web-persistente --% -- sh -c 'echo "<h1>Dato</h1>" > /usr/share/nginx/html/index.html'
```

</details>

---

## Si el PVC dinámico se queda en Pending

No siempre es un error. Revise la causa:

```bash
kubectl describe pvc pvc-dinamico
```

Si el evento dice **`Normal  WaitForFirstConsumer`**, la StorageClass está
esperando a propósito a que exista un Pod que use el PVC. Fíjese en el tipo:
dice `Normal`, no `Warning`.

```bash
kubectl apply -f 02-pv-pvc/05-consumidor-pvc-dinamico.yaml
kubectl get pvc      # ahora sí: Bound
kubectl get pv       # el PV apareció solo
```

### Por qué espera

El orden es contraintuitivo:

```
Lo que uno espera:
  Creo el PVC → Kubernetes crea el disco → llega un Pod y lo usa

Lo que realmente ocurre:
  Creo el PVC → Kubernetes espera
              → llega un Pod que nombra el PVC
              → el Scheduler decide en qué nodo va
              → AHORA se crea el disco, en ESE nodo
              → el PVC se enlaza
```

El Pod no "conectó" el PVC: **fue la señal que faltaba** para saber *dónde*
crear el disco.

La razón es que el provisionador local no crea discos en la nube, sino **una
carpeta en el disco duro de un nodo**. Y una carpeta del nodo A no existe en
el nodo B. Si creara la carpeta antes, tendría que adivinar el nodo; si se
equivoca, el Pod nunca podría arrancar porque su volumen estaría en otra
máquina.

La explicación completa, con la analogía, está en los comentarios de
[`02-pv-pvc/05-consumidor-pvc-dinamico.yaml`](02-pv-pvc/05-consumidor-pvc-dinamico.yaml).

### Los dos modos

```bash
kubectl get storageclass    # mire la columna VOLUMEBINDINGMODE
```

| Modo | Crea el disco | Dónde se ve |
|---|---|---|
| `Immediate` | Al crear el PVC | DOKS, EKS, AKS |
| `WaitForFirstConsumer` | Al crear el primer Pod | Rancher Desktop, k3s, almacenamiento local |

El mismo `04-pvc-dinamico.yaml` se comporta distinto según el clúster. Es un
buen contraste para mostrar en clase.

---

## Notas por entorno

| | Docker Desktop | Rancher Desktop / k3s | DigitalOcean (DOKS) |
|---|---|---|---|
| Provisionador | `docker.io/hostpath` | `rancher.io/local-path` | `dobs.csi.digitalocean.com` |
| Modo de enlace | `Immediate` | `WaitForFirstConsumer` | `Immediate` |
| Clase con `Retain` | No | No | `do-block-storage-retain` |
| Expandir volumen | No | No | Sí |
| Error de `ReadWriteOnce` | No reproducible | No reproducible | Sí, con varios nodos |

Los dos primeros son clústeres de **un solo nodo**, así que varias réplicas
caben en la misma máquina y no se produce el conflicto de `ReadWriteOnce`.
Para ese experimento hace falta el clúster de DigitalOcean o el on-premise.

Para demostrar `Retain` en un entorno local, use el **PV estático**
(`01-pv-estatico.yaml`), que lo declara explícitamente.

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
