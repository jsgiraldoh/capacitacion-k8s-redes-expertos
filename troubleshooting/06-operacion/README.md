# Operación y nodos

**Módulo 9 · Sesión 9** · Repasa la Sesión 8

HPA sin datos y nodos que no reciben Pods.

## Escenarios de esta carpeta

| Escenario | Archivo | Qué falla |
|---|---|---|
| 14 | `14-hpa-sin-requests-roto.yaml` | El HPA no puede calcular el porcentaje de CPU |
| 15 | `15-nodo-cordon-roto.yaml` | Todos los nodos están marcados como no programables |

## Cómo trabajar cada escenario

1. Aplicar el archivo **roto**.
2. Observar el síntoma con `kubectl get`.
3. Diagnosticar con el método: `describe`, eventos, logs, `exec`.
4. Corregir y comprobar que el recurso queda sano.

Todos los comandos están en **una sola línea** (compatibles con PowerShell).
Antes de empezar: `kubectl apply -f ../00-namespace.yaml` y `kubectl apply -f ../00-cliente.yaml`.

---

## Escenario 14 · HPA en <unknown>: el Deployment no declara requests

**Repasa:** Sesión 8 (HPA y requests)  
**Archivo:** `14-hpa-sin-requests-roto.yaml`

### Síntoma

El HPA se crea, pero la columna `TARGETS` no se resuelve nunca: se queda en `<unknown>`, aunque `kubectl top pods` funcione.

```text
$ kubectl apply -f 14-hpa-sin-requests-roto.yaml
deployment.apps/cpu-app created
horizontalpodautoscaler.autoscaling/cpu-app created
$ kubectl get hpa cpu-app -n lab-ts
NAME      REFERENCE            TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
cpu-app   Deployment/cpu-app   <unknown>/50%   1         3         1          2m
```

### Diagnóstico

`describe hpa` explica por qué no puede calcular. El mensaje `missing request for cpu` apunta al Deployment. Si el mensaje fuera `unable to get metrics`, el problema sería Metrics Server (Sesión 8, Parte 1).

```text
$ kubectl describe hpa cpu-app -n lab-ts
...
Conditions:
  Type           Status  Reason                   Message
  ----           ------  ------                   -------
  AbleToScale    True    SucceededGetScale        the HPA controller was able to get the target's current scale
  ScalingActive  False   FailedGetResourceMetric  the HPA was unable to compute the replica count: failed to get cpu utilization: missing request for cpu in container php-apache of Pod cpu-app-7d9c8b6f5-xk2tn
Events:
  Type     Reason                        Age   From                       Message
  ----     ------                        ----  ----                       -------
  Warning  FailedGetResourceMetric       30s   horizontal-pod-autoscaler  failed to get cpu utilization: missing request for cpu in container php-apache of Pod cpu-app-7d9c8b6f5-xk2tn
```

### Causas posibles

- El Deployment no declara `requests.cpu` (este escenario).
- Metrics Server no está instalado o no está disponible (`unable to get metrics for resource cpu`).
- Un Pod con varios contenedores donde solo uno declara `requests`: todos deben tenerlos.
- Recién creado: los primeros 30 a 60 segundos `<unknown>` es normal.


### Solución

Se agregan los requests con `kubectl set resources`. El Deployment se actualiza y el HPA ya puede calcular el porcentaje.

```text
$ kubectl set resources deployment cpu-app --requests=cpu=200m --limits=cpu=500m -n lab-ts
deployment.apps/cpu-app resource requirements updated
$ kubectl get hpa cpu-app -n lab-ts
NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
cpu-app   Deployment/cpu-app   0%/50%    1         3         1          4m
```

> **Lección.** El HPA mide un porcentaje: sin `requests.cpu` no hay denominador y no hay escalado.

**Pregunta para el grupo:** Metrics Server funciona y `kubectl top` devuelve datos, pero el HPA sigue en `<unknown>`. ¿Qué falta?


---

## Escenario 15 · Pods en Pending: el nodo quedó en cordon

**Repasa:** Sesión 8 (cordon, drain, uncordon)  
**Archivo:** `15-nodo-cordon-roto.yaml`

### Síntoma

Primero se deja el nodo en `cordon`, como si alguien hubiera olvidado revertir un mantenimiento. Después se aplica el manifiesto: los Pods quedan en `Pending`.

```text
$ kubectl cordon docker-desktop
node/docker-desktop cordoned
$ kubectl apply -f 15-nodo-cordon-roto.yaml
deployment.apps/app-cordon created
$ kubectl get pods -l app=app-cordon -n lab-ts
NAME                         READY   STATUS    RESTARTS   AGE
app-cordon-6c8d7f9b5-2xwkq   0/1     Pending   0          25s
app-cordon-6c8d7f9b5-9tnhm   0/1     Pending   0          25s
```

### Diagnóstico

Los eventos del Pod dicen `unschedulable`, y `get nodes` lo confirma: `SchedulingDisabled` es la marca de un nodo en cordon.

```text
$ kubectl describe pod -l app=app-cordon -n lab-ts
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  30s   default-scheduler  0/1 nodes are available: 1 node(s) were unschedulable. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
$ kubectl get nodes
NAME             STATUS                     ROLES           AGE   VERSION
docker-desktop   Ready,SchedulingDisabled   control-plane   12d   v1.34.1
```

### Causas posibles

- Nodos en `cordon` o `drain` que no se revirtieron (este escenario).
- Todos los nodos tienen un taint que el Pod no tolera.
- Nodos en `NotReady`: el Scheduler no los usa.
- Recursos insuficientes en los nodos disponibles.


### Solución

`uncordon` devuelve el nodo al pool. Los Pods en `Pending` se programan solos, sin recrearlos.

```text
$ kubectl uncordon docker-desktop
node/docker-desktop uncordoned
$ kubectl get pods -l app=app-cordon -n lab-ts
NAME                         READY   STATUS    RESTARTS   AGE
app-cordon-6c8d7f9b5-2xwkq   1/1     Running   0          1m
app-cordon-6c8d7f9b5-9tnhm   1/1     Running   0          1m
```

> **Lección.** Cuando el manifiesto está bien y el Pod no se programa, el problema está en el clúster: `get nodes` es el siguiente comando.

**Pregunta para el grupo:** ¿Cómo evitaría dejar un nodo olvidado en cordon después de un mantenimiento?


---

## Demostración del instructor: nodo `NotReady` (solo kubeadm)

> **Solo en un clúster de práctica con varios nodos.** Nunca en producción. Es reversible, pero deja un nodo sin servicio unos minutos.

El nodo `NotReady` no se reproduce con un manifiesto: se provoca deteniendo el kubelet de un worker.

```text
$ kubectl get nodes
NAME        STATUS   ROLES           AGE   VERSION
master      Ready    control-plane   20d   v1.34.1
worker-1    Ready    <none>          20d   v1.34.1
worker-2    Ready    <none>          20d   v1.34.1
```

**1. Detener el kubelet** (por SSH, en el worker, nunca en el control plane):

```text
$ sudo systemctl stop kubelet
```

**2. Observar desde el equipo cliente.** Pasados unos 40 segundos el nodo cambia de estado:

```text
$ kubectl get nodes
NAME        STATUS     ROLES           AGE   VERSION
worker-1    NotReady   <none>          20d   v1.34.1

$ kubectl describe node worker-1
...
Conditions:
  Type    Status    Reason              Message
  ----    ------    ------              -------
  Ready   Unknown   NodeStatusUnknown   Kubelet stopped posting node status.
```

Los Pods del nodo siguen apareciendo en `Running` durante un rato: el control plane espera unos 5 minutos antes de evacuarlos.

**3. Diagnosticar en el nodo** (la causa casi nunca se ve desde `kubectl`):

```text
$ systemctl status kubelet
$ journalctl -u kubelet -n 50 --no-pager
$ sudo crictl ps
$ df -h
$ sudo kubeadm certs check-expiration
```

| Dónde mirar | Qué revela |
|---|---|
| `systemctl status kubelet` | Si el kubelet está detenido o reiniciándose |
| `journalctl -u kubelet` | El motivo: certificado, runtime, configuración |
| `crictl ps` | Si el runtime de contenedores responde |
| `df -h` | Disco lleno (`DiskPressure`) |
| `kubeadm certs check-expiration` | Certificados vencidos del control plane |

**4. Revertir:**

```text
$ sudo systemctl start kubelet
```

El nodo vuelve a `Ready` en unos segundos.
