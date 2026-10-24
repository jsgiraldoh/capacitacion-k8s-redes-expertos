# ClusterRole y ClusterRoleBinding

**Módulo 6 · Sesión 6**

Lo mismo que Role y RoleBinding, pero sin límite de namespace.
Necesario para recursos globales y para acceso entre namespaces.

## Archivos

| Archivo | Qué demuestra |
|---|---|
| `01-clusterrole.yaml` | Acceso a recursos sin namespace (Nodes, PV) y a todos los namespaces |
| `02-clusterrole-como-plantilla.yaml` | ClusterRole reutilizado con RoleBinding para contener el acceso |

## Orden de aplicación

```bash
# Los namespaces deben existir primero
kubectl apply -f ../role/00-namespaces.yaml

kubectl apply -f 01-clusterrole.yaml
kubectl apply -f 02-clusterrole-como-plantilla.yaml
```

---

## Cuándo usar ClusterRole en lugar de Role

| Necesidad | Solución |
|---|---|
| Ver Nodes, PV, Namespaces, StorageClasses | ClusterRole (son recursos sin namespace) |
| Acceso a todos los namespaces a la vez | ClusterRole + ClusterRoleBinding |
| Mismo conjunto de permisos en varios namespaces | ClusterRole + RoleBinding en cada namespace |

---

## Probar el acceso a recursos sin namespace

```bash
# ✅ operador puede ver Nodes
kubectl get nodes --as=operador

# ✅ operador puede ver PersistentVolumes
kubectl get pv --as=operador

# ✅ operador puede ver StorageClasses
kubectl get storageclass --as=operador

# ❌ operador NO puede ver Pods (no está en el ClusterRole)
kubectl get pods --as=operador
```

## Probar el acceso a todos los namespaces

```bash
# ✅ monitor ve Pods en equipo-a
kubectl get pods -n equipo-a --as=monitor

# ✅ monitor ve Pods en equipo-b (un Role nunca podría hacer esto)
kubectl get pods -n equipo-b --as=monitor

# ✅ monitor ve Pods en kube-system
kubectl get pods -n kube-system --as=monitor

# ❌ monitor NO puede borrar nada
kubectl delete pod <nombre> -n equipo-a --as=monitor
```

Ver todos los permisos de monitor (los recursos con `*` en namespace
son los que no tienen límite):

```bash
kubectl auth can-i --list --as=monitor
```

---

## El truco de usar ClusterRole como plantilla

El archivo `02-clusterrole-como-plantilla.yaml` muestra el patrón
más importante: definir el ClusterRole una vez y asignarlo con
RoleBindings distintos en cada namespace.

```
ClusterRole desarrollador
  ├── RoleBinding en equipo-a  →  ana tiene permisos SOLO en equipo-a
  └── RoleBinding en equipo-b  →  pedro tiene permisos SOLO en equipo-b
```

```bash
# ana está contenida en equipo-a
kubectl auth can-i create deployments --as=ana -n equipo-a  # yes
kubectl auth can-i create deployments --as=ana -n equipo-b  # no

# pedro está contenido en equipo-b
kubectl auth can-i create deployments --as=pedro -n equipo-b  # yes
kubectl auth can-i create deployments --as=pedro -n equipo-a  # no
```

La diferencia con un ClusterRoleBinding es una sola palabra en el YAML,
pero cambia el alcance de todo el clúster a un namespace concreto.

---

## El contraste más instructivo de la sesión

```bash
# Con RoleBinding: ana solo ve equipo-a
kubectl auth can-i list pods --as=ana -n equipo-b   # no

# Cambiar a ClusterRoleBinding
kubectl create clusterrolebinding ana-global \
  --clusterrole=desarrollador --user=ana

# Ahora ana ve todo
kubectl auth can-i list pods --as=ana -n equipo-b       # yes
kubectl auth can-i list pods --as=ana -n kube-system    # yes

# Limpiar
kubectl delete clusterrolebinding ana-global
```

El mismo ClusterRole, el mismo usuario, pero el tipo de binding lo cambia todo.

---

## Limpieza completa

```bash
kubectl delete -f 02-clusterrole-como-plantilla.yaml
kubectl delete -f 01-clusterrole.yaml
kubectl delete -f ../role/03-role-serviceaccount.yaml
kubectl delete -f ../role/02-role-group.yaml
kubectl delete -f ../role/01-role-user.yaml
kubectl delete -f ../role/00-namespaces.yaml
```
