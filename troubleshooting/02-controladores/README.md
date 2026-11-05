# Controladores

**Módulo 9 · Sesión 9** · Repasa la Sesión 2

El Deployment rechaza el manifiesto o una actualización se atasca.

## Escenarios de esta carpeta

| Escenario | Archivo | Qué falla |
|---|---|---|
| 05 | `05-selector-roto.yaml` | El selector no coincide con las etiquetas del Pod |
| 06 | `06-rollout-v2-roto.yaml` | Actualización con una imagen inexistente |

## Cómo trabajar cada escenario

1. Aplicar el archivo **roto**.
2. Observar el síntoma con `kubectl get`.
3. Diagnosticar con el método: `describe`, eventos, logs, `exec`.
4. Corregir y comprobar que el recurso queda sano.

Todos los comandos están en **una sola línea** (compatibles con PowerShell).
Antes de empezar: `kubectl apply -f ../00-namespace.yaml` y `kubectl apply -f ../00-cliente.yaml`.

---

## Escenario 05 · El Deployment rechaza el manifiesto: selector y labels

**Repasa:** Sesión 2 (Deployments y Labels)  
**Archivo:** `05-selector-roto.yaml`

### Síntoma

El error llega de inmediato, en el propio `kubectl apply`. No hay Pods que revisar, porque el objeto nunca se creó.

```text
$ kubectl apply -f 05-selector-roto.yaml
The Deployment "web-selector" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"app":"webapp"}: `selector` does not match template `labels`
$ kubectl get deployment web-selector -n lab-ts
Error from server (NotFound): deployments.apps "web-selector" not found
```

### Diagnóstico

Cuando `apply` falla, el mensaje ya es el diagnóstico: hay que leerlo completo. Dice qué campo es inválido y por qué. Basta comparar `spec.selector.matchLabels` con `spec.template.metadata.labels`.

```text
$ kubectl apply -f 05-selector-roto.yaml --dry-run=server
The Deployment "web-selector" is invalid: spec.template.metadata.labels: Invalid value: ...: `selector` does not match template `labels`
```

### Causas posibles

- Etiquetas del Pod distintas del selector (este escenario).
- Un typo en una sola letra: `app: web` frente a `app: web-app`.
- Cambiar el `selector` de un Deployment existente: es inmutable y obliga a recrearlo.


### Solución

Se igualan las etiquetas (`app: web` en ambos lugares) y se vuelve a aplicar. `--dry-run=server` permite probar un manifiesto sin crear nada.

```text
$ kubectl apply -f 05-selector-roto.yaml
deployment.apps/web-selector created
```

Cambio en el archivo `05-selector-roto.yaml`:

```yaml
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web                      # ← ahora coincide con el selector
```

> **Lección.** Un error en `apply` es el más barato de diagnosticar: el API Server dice exactamente qué campo falla.

**Pregunta para el grupo:** ¿Por qué el selector de un Deployment no se puede cambiar una vez creado?


---

## Escenario 06 · Rollout atascado: la nueva versión no arranca

**Repasa:** Sesión 2 (rolling update y rollback)  
**Archivo:** `06-rollout-v2-roto.yaml`  
**Antes:** aplicar `06-rollout-v1.yaml`

### Síntoma

Primero se despliega la versión 1 (tres réplicas sanas). Después se aplica la versión 2: `rollout status` no termina y aparece un Pod nuevo en `ImagePullBackOff`, mientras los tres anteriores siguen en `Running`.

```text
$ kubectl apply -f 06-rollout-v1.yaml
deployment.apps/web-rollout created
$ kubectl apply -f 06-rollout-v2-roto.yaml
deployment.apps/web-rollout configured
$ kubectl rollout status deployment/web-rollout --timeout=30s
Waiting for deployment "web-rollout" rollout to finish: 1 out of 3 new replicas have been updated...
error: timed out waiting for the condition
$ kubectl get pods -l app=web-rollout
NAME                          READY   STATUS             RESTARTS   AGE
web-rollout-6d5f9c7b4-8kqzn   1/1     Running            0          3m
web-rollout-6d5f9c7b4-j4wxp   1/1     Running            0          3m
web-rollout-6d5f9c7b4-r9tvm   1/1     Running            0          3m
web-rollout-79c8f5d6b-l2hnc   0/1     ImagePullBackOff   0          45s
```

### Diagnóstico

Dos ReplicaSets conviven: el antiguo (3 de 3 listos) y el nuevo (1 creado, 0 listos). El Deployment avanza solo mientras el Pod nuevo esté listo, y por eso el servicio sigue funcionando. El motivo del Pod nuevo está, como siempre, en sus eventos.

```text
$ kubectl get rs -l app=web-rollout
NAME                    DESIRED   CURRENT   READY   AGE
web-rollout-6d5f9c7b4   3         3         3       3m
web-rollout-79c8f5d6b   1         1         0       45s
$ kubectl rollout history deployment/web-rollout
deployment.apps/web-rollout
REVISION   CHANGE-CAUSE
1          <none>
2          <none>
```

### Causas posibles

- Imagen inexistente en la nueva versión (este escenario).
- Una readiness probe que nunca pasa en la nueva versión (`READY 0/1` con `Running`).
- La nueva versión pide más recursos de los disponibles (`Pending`).
- Un ConfigMap o Secret nuevo que la versión 2 necesita y no existe.


### Solución

`rollout undo` vuelve a la revisión anterior. Es la salida rápida; después se corrige el manifiesto con calma.

```text
$ kubectl rollout undo deployment/web-rollout
deployment.apps/web-rollout rolled back
$ kubectl rollout status deployment/web-rollout
deployment "web-rollout" successfully rolled out
```

> **Lección.** Primero se restaura el servicio (`rollout undo`) y después se investiga la causa.

**Pregunta para el grupo:** Los usuarios no notaron nada durante el rollout fallido. ¿Qué parte del diseño del Deployment lo explica?


---
