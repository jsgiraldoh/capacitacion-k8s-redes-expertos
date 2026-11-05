# Almacenamiento

**Módulo 9 · Sesión 9** · Repasa la Sesión 5

Un PVC no se enlaza y el Pod espera.

## Escenarios de esta carpeta

| Escenario | Archivo | Qué falla |
|---|---|---|
| 10 | `10-pvc-storageclass-roto.yaml` | El PVC pide una StorageClass inexistente |

## Cómo trabajar cada escenario

1. Aplicar el archivo **roto**.
2. Observar el síntoma con `kubectl get`.
3. Diagnosticar con el método: `describe`, eventos, logs, `exec`.
4. Corregir y comprobar que el recurso queda sano.

Todos los comandos están en **una sola línea** (compatibles con PowerShell).
Antes de empezar: `kubectl apply -f ../00-namespace.yaml` y `kubectl apply -f ../00-cliente.yaml`.

---

## Escenario 10 · PVC en Pending: la StorageClass no existe

**Repasa:** Sesión 5 (PV, PVC y StorageClass)  
**Archivo:** `10-pvc-storageclass-roto.yaml`

### Síntoma

El Deployment existe, pero el Pod no se programa y el PVC no se enlaza. Los dos quedan en `Pending`.

```text
$ kubectl apply -f 10-pvc-storageclass-roto.yaml
persistentvolumeclaim/datos-pvc created
deployment.apps/app-datos created
$ kubectl get pvc,pods -n lab-ts -l app=app-datos
NAME                              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/datos-pvc   Pending                                      ssd-premium    1m

NAME                            READY   STATUS    RESTARTS   AGE
pod/app-datos-7b9d6c8f5-q4xlm   0/1     Pending   0          1m
```

### Diagnóstico

Se sigue la cadena de dependencias de abajo hacia arriba: el Pod espera al PVC, y el PVC espera a la StorageClass. Los eventos del PVC dicen cuál falta; `get storageclass` muestra las que existen.

```text
$ kubectl describe pvc datos-pvc -n lab-ts
...
Events:
  Type     Reason              Age                From                         Message
  ----     ------              ----               ----                         -------
  Warning  ProvisioningFailed  8s (x6 over 1m)    persistentvolume-controller  storageclass.storage.k8s.io "ssd-premium" not found
$ kubectl get storageclass
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
hostpath (default)   docker.io/hostpath   Delete          Immediate           false                  12d
```

### Causas posibles

- StorageClass inexistente o mal escrita (este escenario).
- No hay StorageClass por defecto y el PVC no declara ninguna.
- `WaitForFirstConsumer`: el PVC se queda en `Pending` hasta que un Pod lo use. Es normal, no un fallo.
- El aprovisionador (CSI) está caído, o pide un modo de acceso que no soporta (`ReadWriteMany`).


### Solución

Como `storageClassName` es inmutable, se borra todo, se corrige el archivo con una StorageClass que exista (aquí `hostpath`, la predeterminada) y se aplica de nuevo.

```text
$ kubectl delete -f 10-pvc-storageclass-roto.yaml
persistentvolumeclaim "datos-pvc" deleted
deployment.apps "app-datos" deleted
$ kubectl apply -f 10-pvc-storageclass-roto.yaml
persistentvolumeclaim/datos-pvc created
deployment.apps/app-datos created
$ kubectl get pvc -n lab-ts
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
datos-pvc   Bound    pvc-3f8e1a7c-2b4d-4c6e-9a1f-5d7b8c9e0f12   1Gi        RWO            hostpath       6s
```

Cambio en el archivo `10-pvc-storageclass-roto.yaml`:

```yaml
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: hostpath          # ← una StorageClass que existe
```

> **Lección.** Un Pod que espera un volumen se diagnostica siguiendo la cadena: Pod, PVC, StorageClass.

**Pregunta para el grupo:** ¿Cómo distinguiría un PVC en `Pending` por `WaitForFirstConsumer` de uno en `Pending` por un fallo?


---
