# Seguridad y permisos

**Módulo 9 · Sesión 9** · Repasa la Sesión 6

RBAC, securityContext y Pod Security Admission.

## Escenarios de esta carpeta

| Escenario | Archivo | Qué falla |
|---|---|---|
| 11 | `11-rbac-forbidden-roto.yaml` | Una aplicación recibe 403 al llamar al API Server |
| 12 | `12-runasnonroot-roto.yaml` | securityContext exige un usuario no root y la imagen usa root |
| 13 | `13-psa-restricted-roto.yaml` | Un namespace con nivel restricted rechaza un Pod sin securityContext |

## Cómo trabajar cada escenario

1. Aplicar el archivo **roto**.
2. Observar el síntoma con `kubectl get`.
3. Diagnosticar con el método: `describe`, eventos, logs, `exec`.
4. Corregir y comprobar que el recurso queda sano.

Todos los comandos están en **una sola línea** (compatibles con PowerShell).
Antes de empezar: `kubectl apply -f ../00-namespace.yaml` y `kubectl apply -f ../00-cliente.yaml`.

---

## Escenario 11 · Forbidden: el ServiceAccount no tiene permisos

**Repasa:** Sesión 6 (RBAC y ServiceAccounts)  
**Archivo:** `11-rbac-forbidden-roto.yaml`

### Síntoma

El Pod está `Running` y `1/1`. El problema solo aparece en sus logs: la aplicación recibe `403`.

```text
$ kubectl apply -f 11-rbac-forbidden-roto.yaml
serviceaccount/sa-lector created
deployment.apps/consulta created
$ kubectl logs deploy/consulta -n lab-ts --tail=2
HTTP 403
  "message": "pods is forbidden: User \"system:serviceaccount:lab-ts:sa-lector\" cannot list resource \"pods\" in API group \"\" in the namespace \"lab-ts\"",
```

### Diagnóstico

El mensaje del API Server ya dice quién (`sa-lector`), qué intentó (`list pods`) y dónde (`lab-ts`). Se confirma con `auth can-i` y se comprueba si existe algún RoleBinding para ese ServiceAccount.

```text
$ kubectl auth can-i list pods --as=system:serviceaccount:lab-ts:sa-lector -n lab-ts
no
$ kubectl get rolebindings -n lab-ts
No resources found in lab-ts namespace.
```

### Causas posibles

- El ServiceAccount no tiene Role ni RoleBinding (este escenario).
- El Role existe, pero con otro verbo o recurso (`get` en lugar de `list`).
- El RoleBinding está en otro namespace, o apunta a un ServiceAccount con otro nombre.
- Falta `apiGroups` correcto: `deployments` está en `apps`, no en el grupo core.


### Solución

Se crea un Role con el verbo y el recurso exactos, y un RoleBinding que lo asigna al ServiceAccount. No hay que reiniciar el Pod: el siguiente intento ya pasa.

```text
$ kubectl create role lector-pods --verb=list --resource=pods -n lab-ts
role.rbac.authorization.k8s.io/lector-pods created
$ kubectl create rolebinding lector-pods --role=lector-pods --serviceaccount=lab-ts:sa-lector -n lab-ts
rolebinding.rbac.authorization.k8s.io/lector-pods created
$ kubectl logs deploy/consulta -n lab-ts --tail=2
HTTP 403
HTTP 200
```

> **Lección.** Un `Forbidden` nunca es un fallo de red: es una decisión de RBAC. `auth can-i --as=...` la reproduce sin tocar la aplicación.

**Pregunta para el grupo:** ¿Por qué conviene dar `list` sobre `pods` y no `*` sobre todos los recursos?


---

## Escenario 12 · runAsNonRoot: la imagen corre como root

**Repasa:** Sesión 6 (securityContext)  
**Archivo:** `12-runasnonroot-roto.yaml`

### Síntoma

El mismo estado que en el escenario 04 (`CreateContainerConfigError`), pero la causa es distinta: no falta ningún objeto.

```text
$ kubectl apply -f 12-runasnonroot-roto.yaml
deployment.apps/web-noroot created
$ kubectl get pods -l app=web-noroot -n lab-ts
NAME                         READY   STATUS                       RESTARTS   AGE
web-noroot-6d7c8b9f5-h3xwk   0/1     CreateContainerConfigError   0          30s
```

### Diagnóstico

Un mismo estado puede tener causas distintas: por eso se lee el evento y no solo el estado. Aquí el mensaje habla de `runAsNonRoot`.

```text
$ kubectl describe pod -l app=web-noroot -n lab-ts
...
Events:
  Type     Reason  Age               From     Message
  ----     ------  ----              ----     -------
  Warning  Failed  6s (x4 over 30s)  kubelet  Error: container has runAsNonRoot and image will run as root (pod: "web-noroot-6d7c8b9f5-h3xwk_lab-ts(...)", container: nginx)
```

### Causas posibles

- Imagen que corre como root con `runAsNonRoot: true` (este escenario).
- La imagen declara un usuario por nombre (no numérico) y el kubelet no puede verificarlo: se define `runAsUser`.
- Un puerto menor a 1024 con un usuario sin privilegios (nginx no puede abrir el 80 sin root).
- Un volumen sin permisos de escritura para ese usuario (`fsGroup`).


### Solución

Se usa una imagen diseñada para no correr como root. `nginxinc/nginx-unprivileged` usa el usuario 101 y escucha en el puerto 8080 en lugar del 80.

```text
$ kubectl set image deployment/web-noroot nginx=nginxinc/nginx-unprivileged:1.25 -n lab-ts
deployment.apps/web-noroot image updated
$ kubectl get pods -l app=web-noroot -n lab-ts
NAME                         READY   STATUS    RESTARTS   AGE
web-noroot-84f6d7c9b-p5nqz   1/1     Running   0          12s
```

> **Lección.** No se baja la regla de seguridad para que el Pod arranque: se cambia la imagen para que cumpla la regla.

**Pregunta para el grupo:** ¿Por qué nginx no puede escuchar en el puerto 80 cuando corre sin privilegios?


---

## Escenario 13 · Pod Security Admission: el namespace rechaza el Pod

**Repasa:** Sesión 6 (Pod Security Admission)  
**Archivo:** `13-psa-restricted-roto.yaml`

### Síntoma

Se crea el namespace, pero el Pod es rechazado en el momento de aplicarlo. El error enumera todo lo que el nivel `restricted` exige.

```text
$ kubectl apply -f 13-psa-restricted-roto.yaml
namespace/lab-psa created
Error from server (Forbidden): error when creating "13-psa-restricted-roto.yaml": pods "pod-psa" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

### Diagnóstico

El mensaje es una lista de verificación completa: cada elemento es un campo que falta. Se confirma el nivel del namespace mirando sus etiquetas.

```text
$ kubectl get namespace lab-psa --show-labels
NAME      STATUS   AGE   LABELS
lab-psa   Active   40s   kubernetes.io/metadata.name=lab-psa,pod-security.kubernetes.io/enforce=restricted
```

### Causas posibles

- El Pod no cumple el nivel del namespace (este escenario).
- Un Deployment en un namespace restringido: el Deployment se crea, pero el ReplicaSet falla (el error está en los eventos, no en `apply`).
- `enforce` rechaza; `warn` y `audit` solo avisan o registran: se revisan las tres etiquetas.


### Solución

Se agrega el `securityContext` completo y una imagen que corra sin privilegios. Estos cuatro campos son los que pide el nivel `restricted`.

```text
$ kubectl apply -f 13-psa-restricted-corregido.yaml
pod/pod-psa created
$ kubectl get pod pod-psa -n lab-psa
NAME      READY   STATUS    RESTARTS   AGE
pod-psa   1/1     Running   0          9s
```

Cambio en el archivo `13-psa-restricted-corregido.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-psa
  namespace: lab-psa
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:1.25
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

> **Lección.** El mensaje de Pod Security Admission es la lista exacta de campos por agregar: se corrige línea por línea.

**Pregunta para el grupo:** ¿Qué diferencia hay entre los modos `enforce`, `warn` y `audit` del Pod Security Admission?


---
