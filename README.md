# Capacitación Kubernetes para Equipo de Redes

Manifiestos y ejemplos prácticos del **Programa de Capacitación Kubernetes
para Equipo de Redes**, impartido a través de EUD Academy.

Cada carpeta corresponde a un módulo del programa y contiene su propio
`README.md` con los comandos, los experimentos y las advertencias de ese
tema. Los archivos están numerados en el orden en que se aplican.

---

## Mapa de carpetas

| Carpeta | Módulo | Sesión | Contenido |
|---|---|---|---|
| [`pods/`](pods/) | 3 | 2 | La unidad mínima: un Pod suelto y su limitación |
| [`replicasets/`](replicasets/) | 3 | 2 | El primer controlador: autoreparación y selectores |
| [`deployments/`](deployments/) | 3 y 4 | 2 y 3 | Cliente `netshoot` para diagnóstico de red |
| [`services/`](services/) | 4 | 3 | ClusterIP, NodePort y LoadBalancer |
| [`storage/`](storage/) | 5 | 5 | Volúmenes efímeros, PV, PVC y StorageClass |
| [`secrets/`](secrets/) | 6 | 6 | Datos sensibles, consumidos como variable y como archivo |
| [`rbac/`](rbac/) | 6 | 6 | Role, RoleBinding, ClusterRole y ClusterRoleBinding, con User, Group y ServiceAccount |
| [`monitoring/`](monitoring/) | 7 | 8 | Metrics Server, probes, eventos y logs, Prometheus y Grafana |
| [`administration/`](administration/) | 8 | 8 | Mantenimiento de nodos, requests y limits, ResourceQuota, LimitRange y HPA |

---

## Recorrido sugerido

El orden del curso cuenta una historia. Cada paso resuelve el problema que
dejó abierto el anterior.

```
pods/             Un Pod suelto. Si lo borra, nadie lo recrea.
  ↓
replicasets/      Alguien vigila que haya N Pods vivos.
  ↓
deployments/      Además, actualizaciones progresivas y rollbacks.
  ↓
services/         Los Pods cambian de IP; el Service da una dirección estable.
  ↓
storage/          Los Pods son efímeros; el volumen sobrevive.
  ↓
secrets/          Los datos sensibles no van en la imagen ni en el YAML.
  ↓
rbac/             No todos deben poder hacer todo: quién puede hacer qué.
  ↓
monitoring/       Ver qué pasa: métricas, probes, eventos y logs.
  ↓
administration/   Operar: mantener nodos, poner límites y escalar solo.
```

---

## Antes de empezar

```bash
# Verificar el clúster y el contexto activo
kubectl get nodes
kubectl config current-context

# Trabajar en un namespace propio (recomendado)
kubectl create namespace taller-sunombre
kubectl config set-context --current --namespace=taller-sunombre
```

Fijar el namespace en el contexto evita tener que escribir `-n` en cada
comando. Es el mismo mecanismo del archivo `~/.kube/config` que vimos en la
Sesión 1.

Las carpetas `rbac/`, `monitoring/` y `administration/` crean sus propios
namespaces de laboratorio (`equipo-a`, `equipo-b`, `lab-s8`, `lab-quota`,
`lab-hpa`) y declaran el namespace de forma explícita en cada manifiesto.

---

## Herramienta de diagnóstico

El cliente de `deployments/` es la navaja suiza de todas las pruebas de red:

```bash
kubectl apply -f deployments/01-deployment-cliente.yaml

# Probar conectividad a un Service por su nombre
kubectl exec deploy/client-deployment -- curl -s http://nginx-service

# Resolver DNS interno
kubectl exec deploy/client-deployment -- nslookup nginx-service
```

---

## Los comandos de diagnóstico

Casi cualquier problema se diagnostica con estos tres, en este orden:

```bash
kubectl get <recurso>               # ¿cuál es el estado?
kubectl describe <recurso> <nombre> # los EVENTOS al final explican por qué
kubectl logs <pod>                  # ¿qué dice la aplicación?
```

Y uno más, específico de Services, que resuelve el fallo más frecuente:

```bash
kubectl get endpoints <servicio>    # si está vacío, el selector no coincide
```

Desde la Sesión 8, dos más:

```bash
kubectl get events --sort-by=.lastTimestamp   # la historia reciente del namespace
kubectl top pods -A                           # consumo real (requiere Metrics Server)
```

Y desde la Sesión 6, para los problemas de permisos (`Forbidden`):

```bash
kubectl auth can-i <verbo> <recurso> -n <namespace> --as=<identidad>
kubectl auth can-i --list -n <namespace> --as=<identidad>
```

---

## Si trabaja desde Windows

Los comandos de este repositorio están escritos en sintaxis de shell de Linux.
Estas son las diferencias que encontrará en PowerShell:

| En Linux / macOS | En PowerShell |
|---|---|
| `comando1 && comando2` | `comando1 ; comando2` (el `&&` solo existe en PowerShell 7+) |
| `... \| base64 -d` | `kubectl get secret X -o go-template="{{.data.clave \| base64decode}}"` |
| `sh -c 'echo "texto" > archivo'` | PowerShell quita las comillas simples y rompe el comando |
| Línea que termina en `\` y continúa abajo | PowerShell no lo entiende: escriba el comando completo en **una sola línea** (o termine la línea con un acento grave ``` ` ```) |
| `kubectl get ... \| grep texto` | `kubectl get ... \| Select-String texto` |

Para el caso de las comillas, la solución más limpia es **entrar al contenedor**
en lugar de pasar el comando desde fuera:

```bash
kubectl exec -it deploy/<nombre> -- sh
# ya dentro, escriba normalmente
```

Así el comando lo interpreta la shell del contenedor y no la de su equipo, y
funciona igual en cualquier sistema operativo.

Los README de `monitoring/` y `administration/` ya están escritos en una sola
línea por comando. Algunos comentarios de `rbac/` usan `\` para continuar
líneas: en PowerShell júntelas en una.

---

## Notas por entorno

Los ejemplos están pensados para funcionar en varios entornos. Donde hay
diferencias, están marcadas dentro de cada archivo.

| | Clúster local | On-premise (kubeadm) | DigitalOcean (DOKS) |
|---|---|---|---|
| **LoadBalancer** | Solo Docker Desktop | Queda en `<pending>` | Funciona, se factura |
| **PVC dinámico** | Según el entorno | Hay que configurar el CSI | `do-block-storage` listo |
| **ReadWriteMany** | No | Requiere NFS | NFS Share de DigitalOcean |
| **Metrics Server** (`kubectl top`, HPA) | Docker Desktop y kind: instalarlo | Instalarlo | Normalmente incluido; verificar |
| **`kubectl drain`** | Con un solo nodo: solo `cordon` | Requiere 2 o más nodos | Requiere 2 o más nodos en el pool |

Metrics Server en clústeres locales y kubeadm casi siempre necesita además el
flag `--kubelet-insecure-tls`. Está explicado paso a paso en
[`monitoring/01-metrics-server/`](monitoring/01-metrics-server/).

---

## Advertencia sobre costos

En DigitalOcean, estos recursos **se facturan mientras existan**, aunque
ningún Pod los use:

- Cada Service de tipo `LoadBalancer` (aproximadamente 12 USD al mes)
- Cada volumen creado por un PVC

Al terminar cada práctica:

```bash
kubectl delete -f <archivo-del-servicio-lb>
kubectl delete pvc --all -n <su-namespace>
```

Con `reclaimPolicy: Retain`, el disco **sobrevive** al borrado del PVC y hay
que eliminarlo a mano desde el panel de DigitalOcean.

---

## Pendiente

Temas del programa que aún no tienen ejemplos en este repositorio:

- Ingress, DNS interno y Network Policies (resto del Módulo 4)
- Troubleshooting (Módulo 9)
- Laboratorio final integrador

---

## Sobre este repositorio

Material didáctico elaborado por **Johan Sebastián Giraldo Hurtado**
([@jsgiraldoh](https://github.com/jsgiraldoh)) para EUD Academy.

Está conectado a **Argo CD**, de modo que los cambios en la rama `master` se
reflejan en el clúster de forma automática. Eso lo convierte, además de en
material de estudio, en una demostración viva de GitOps.

> **Aviso sobre los laboratorios.** Varios manifiestos de `rbac/`,
> `monitoring/` y `administration/` son ejercicios que fallan **a propósito**
> (probes rotas, un Pod que pide 500 Gi, un Pod que termina en `OOMKilled`,
> un Deployment que supera su cuota) y algunos modifican el comportamiento
> del namespace (ResourceQuota, LimitRange). Si la aplicación de Argo CD
> sincroniza la raíz del repositorio de forma recursiva, esos archivos se
> aplicarían solos. Apunte Argo CD solo a las carpetas que quiera
> sincronizar, o aplique los laboratorios manualmente con `kubectl apply`.
