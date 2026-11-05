# Pods

**Módulo 9 · Sesión 9** · Repasa la Sesión 2

Un Pod no arranca, no se programa o se reinicia.

## Escenarios de esta carpeta

| Escenario | Archivo | Qué falla |
|---|---|---|
| 01 | `01-imagepullbackoff-roto.yaml` | Etiqueta de imagen inexistente |
| 02 | `02-crashloopbackoff-roto.yaml` | La app sale con código 1 por falta de una variable |
| 03 | `03-pending-nodeselector-roto.yaml` | nodeSelector con una etiqueta que ningún nodo tiene |
| 04 | `04-config-error-roto.yaml` | El Pod referencia un Secret que no existe |

## Cómo trabajar cada escenario

1. Aplicar el archivo **roto**.
2. Observar el síntoma con `kubectl get`.
3. Diagnosticar con el método: `describe`, eventos, logs, `exec`.
4. Corregir y comprobar que el recurso queda sano.

Todos los comandos están en **una sola línea** (compatibles con PowerShell).
Antes de empezar: `kubectl apply -f ../00-namespace.yaml` y `kubectl apply -f ../00-cliente.yaml`.

---

## Escenario 01 · ImagePullBackOff: la imagen no existe

**Repasa:** Sesión 2 (Pods y Deployments)  
**Archivo:** `01-imagepullbackoff-roto.yaml`

### Síntoma

Se aplica el manifiesto. El Deployment se crea sin problema, pero el Pod nunca llega a `Running`: pasa por `ErrImagePull` y se queda en `ImagePullBackOff`.

```text
$ kubectl apply -f 01-imagepullbackoff-roto.yaml
deployment.apps/web-imagen created
$ kubectl get pods -l app=web-imagen
NAME                         READY   STATUS             RESTARTS   AGE
web-imagen-5c9d8b7f6-k2xqp   0/1     ImagePullBackOff   0          48s
```

### Diagnóstico

El estado dice qué pasa, pero no por qué. La respuesta está en los eventos del Pod: `kubectl describe` los muestra al final. El mensaje `not found` indica que el registro no tiene esa etiqueta.

```text
$ kubectl describe pod -l app=web-imagen
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  52s                default-scheduler  Successfully assigned lab-ts/web-imagen-5c9d8b7f6-k2xqp to docker-desktop
  Normal   Pulling    25s (x3 over 52s)  kubelet            Pulling image "nginx:1.25.99"
  Warning  Failed     24s (x3 over 51s)  kubelet            Failed to pull image "nginx:1.25.99": failed to resolve reference "docker.io/library/nginx:1.25.99": not found
  Warning  Failed     24s (x3 over 51s)  kubelet            Error: ErrImagePull
  Normal   BackOff    10s (x3 over 50s)  kubelet            Back-off pulling image "nginx:1.25.99"
  Warning  Failed     10s (x3 over 50s)  kubelet            Error: ImagePullBackOff
```

### Causas posibles

- Etiqueta o nombre de la imagen mal escrito (el caso de este escenario).
- Imagen en un registro privado y el Pod no tiene `imagePullSecrets` (el mensaje dice `unauthorized` o `pull access denied`).
- El nodo no tiene salida a internet o al registro (el mensaje dice `timeout` o `no such host`).
- Límite de descargas del registro público (`toomanyrequests`).


### Solución

Se corrige la etiqueta con `kubectl set image`. Kubernetes crea un Pod nuevo y el anterior desaparece.

```text
$ kubectl set image deployment/web-imagen nginx=nginx:1.25
deployment.apps/web-imagen image updated
$ kubectl get pods -l app=web-imagen
NAME                         READY   STATUS    RESTARTS   AGE
web-imagen-6b7c9d5f8-m4tzr   1/1     Running   0          14s
```

> **Lección.** `describe` primero: los eventos casi siempre nombran la causa exacta del `ImagePullBackOff`.

**Pregunta para el grupo:** ¿Cómo distinguiría un error de etiqueta de un error de credenciales del registro? (Pista: el texto del evento.)


---

## Escenario 02 · CrashLoopBackOff: la aplicación termina con error

**Repasa:** Sesión 2 (Pods) y Sesión 6 (variables de entorno)  
**Archivo:** `02-crashloopbackoff-roto.yaml`

### Síntoma

El Pod arranca, falla y se reinicia. El contador `RESTARTS` sube y el estado alterna entre `Error` y `CrashLoopBackOff`.

```text
$ kubectl apply -f 02-crashloopbackoff-roto.yaml
deployment.apps/app-config created
$ kubectl get pods -l app=app-config
NAME                         READY   STATUS             RESTARTS      AGE
app-config-7d8f6c5b9-zq8wn   0/1     CrashLoopBackOff   3 (28s ago)   75s
```

### Diagnóstico

Aquí `describe` solo confirma que el contenedor termina (`Exit Code: 1`). La causa la dice la aplicación: se leen sus logs. Con `--previous` se ve el contenedor anterior, porque el actual puede acabar de arrancar.

```text
$ kubectl describe pod -l app=app-config
...
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
    Restart Count:  4
...
$ kubectl logs -l app=app-config --previous
ERROR: falta la variable DB_HOST
```

### Causas posibles

- Falta una variable de entorno, un archivo o un Secret que la aplicación necesita (el caso de este escenario).
- Error de la propia aplicación al iniciar (configuración inválida, puerto ocupado).
- Comando o argumentos mal escritos en `command` y `args`.
- Una liveness probe que mata un contenedor que sí funciona (se ve en los eventos).


### Solución

Se agrega la variable con `kubectl set env`. El Deployment crea un Pod nuevo con ella.

```text
$ kubectl set env deployment/app-config DB_HOST=db.interno
deployment.apps/app-config env updated
$ kubectl logs -l app=app-config
conectado a db.interno
```

> **Lección.** `CrashLoopBackOff` significa que el contenedor sí arranca y luego falla: la respuesta está en `logs --previous`.

**Pregunta para el grupo:** ¿Por qué `kubectl logs` sin `--previous` puede mostrar una salida vacía en un Pod en CrashLoopBackOff?


---

## Escenario 03 · Pending: ningún nodo cumple el nodeSelector

**Repasa:** Sesión 1 (Scheduler) y Sesión 8 (nodos)  
**Archivo:** `03-pending-nodeselector-roto.yaml`

### Síntoma

El Pod queda en `Pending` y no cambia. `RESTARTS` es 0 y el `AGE` crece: nunca llegó a ejecutarse en ningún nodo.

```text
$ kubectl apply -f 03-pending-nodeselector-roto.yaml
deployment.apps/web-ssd created
$ kubectl get pods -l app=web-ssd
NAME                      READY   STATUS    RESTARTS   AGE
web-ssd-6f9c7d8b5-t7xkp   0/1     Pending   0          2m
```

### Diagnóstico

Un Pod `Pending` aún no tiene contenedor, así que no hay logs. El Scheduler explica en los eventos por qué no lo programa. Luego se comprueban las etiquetas reales de los nodos.

```text
$ kubectl describe pod -l app=web-ssd
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  2m    default-scheduler  0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
$ kubectl get nodes --show-labels
NAME             STATUS   ROLES           AGE   VERSION   LABELS
docker-desktop   Ready    control-plane   12d   v1.34.1   beta.kubernetes.io/arch=arm64,kubernetes.io/hostname=docker-desktop,...
```

### Causas posibles

- `nodeSelector` o afinidad con una etiqueta que ningún nodo tiene (este escenario).
- Requests de CPU o memoria mayores que lo disponible (`Insufficient cpu`).
- Nodos con un taint que el Pod no tolera (`had untolerated taint`).
- Todos los nodos en `cordon` (`were unschedulable`, escenario 15).


### Solución

Hay dos caminos: etiquetar un nodo (si el requisito es válido) o quitar el `nodeSelector` (si no lo es). Aquí se etiqueta el nodo; el Pod se programa solo.

```text
$ kubectl label node docker-desktop disktype=ssd
node/docker-desktop labeled
$ kubectl get pods -l app=web-ssd
NAME                      READY   STATUS    RESTARTS   AGE
web-ssd-6f9c7d8b5-t7xkp   1/1     Running   0          3m
```

Para dejar el nodo como estaba: `kubectl label node docker-desktop disktype-` (el guion final quita la etiqueta).

> **Lección.** Un Pod en `Pending` no tiene logs: el motivo está en los eventos de `describe`, escritos por el Scheduler.

**Pregunta para el grupo:** El evento dice `0/1 nodes are available`. ¿Qué significan esos dos números en un clúster de 3 nodos?


---

## Escenario 04 · CreateContainerConfigError: falta un Secret

**Repasa:** Sesión 6 (Secrets)  
**Archivo:** `04-config-error-roto.yaml`

### Síntoma

La imagen sí se descarga, pero el contenedor no se crea. El estado es `CreateContainerConfigError`, distinto de `ImagePullBackOff`.

```text
$ kubectl apply -f 04-config-error-roto.yaml
deployment.apps/api-secret created
$ kubectl get pods -l app=api-secret
NAME                         READY   STATUS                       RESTARTS   AGE
api-secret-5b6d8f9c7-w2mhd   0/1     CreateContainerConfigError   0          35s
```

### Diagnóstico

`describe` nombra el objeto que falta. El mismo error aparece con un ConfigMap inexistente o con una clave (`key`) mal escrita dentro de un Secret que sí existe.

```text
$ kubectl describe pod -l app=api-secret
...
Events:
  Type     Reason  Age                From     Message
  ----     ------  ----               ----     -------
  Warning  Failed  8s (x4 over 35s)   kubelet  Error: secret "credenciales-db" not found
$ kubectl get secrets -n lab-ts
No resources found in lab-ts namespace.
```

### Causas posibles

- El Secret o el ConfigMap referenciado no existe en ese namespace (este escenario).
- La `key` pedida no existe dentro del Secret (`couldn't find key`).
- El objeto existe, pero en otro namespace: los Secrets no se comparten entre namespaces.
- `runAsNonRoot` con una imagen que corre como root (escenario 12).


### Solución

Se crea el Secret que faltaba. No hace falta tocar el Deployment: el kubelet reintenta y el Pod arranca.

```text
$ kubectl create secret generic credenciales-db --from-literal=DB_PASS=S3cr3t0 -n lab-ts
secret/credenciales-db created
$ kubectl get pods -l app=api-secret
NAME                         READY   STATUS    RESTARTS   AGE
api-secret-5b6d8f9c7-w2mhd   1/1     Running   0          2m
```

> **Lección.** Los objetos que un Pod referencia (Secret, ConfigMap, PVC, ServiceAccount) deben existir antes o el Pod espera.

**Pregunta para el grupo:** ¿Qué cambiaría en el diagnóstico si el Secret existiera pero la clave se llamara `PASSWORD`?


---
