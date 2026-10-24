# Role y RoleBinding

**Módulo 6 · Sesión 6**

Permisos limitados a un namespace. Un Role define qué se puede hacer;
un RoleBinding decide a quién se le da.

## Archivos

| Archivo | Sujeto | Qué demuestra |
|---|---|---|
| `00-namespaces.yaml` | — | Preparación: dos namespaces de laboratorio |
| `01-role-user.yaml` | User | El límite del namespace en acción |
| `02-role-group.yaml` | Group | Permisos para un equipo entero |
| `03-role-serviceaccount.yaml` | ServiceAccount | Identidad de una aplicación que corre en el clúster |

## Orden de aplicación

```bash
kubectl apply -f 00-namespaces.yaml
kubectl apply -f 01-role-user.yaml
kubectl apply -f 02-role-group.yaml
kubectl apply -f 03-role-serviceaccount.yaml
```

---

## Los tres tipos de sujeto

### User — persona que usa kubectl

```bash
# ✅ juan puede listar Pods en equipo-a
kubectl get pods -n equipo-a --as=juan

# ❌ juan NO puede nada en equipo-b
kubectl get pods -n equipo-b --as=juan

# ❌ juan NO puede borrar (solo tiene get, list, watch)
kubectl delete pod nginx -n equipo-a --as=juan
```

### Group — un equipo completo

```bash
# ✅ Ana pertenece al grupo → tiene acceso
kubectl get pods -n equipo-a \
  --as=ana --as-group=equipo-redes

# ❌ Marta NO pertenece → acceso denegado
kubectl get pods -n equipo-a --as=marta
```

### ServiceAccount — una aplicación dentro del clúster

```bash
# La sintaxis es: system:serviceaccount:<namespace>:<nombre>

# ✅ El SA puede listar Pods
kubectl auth can-i list pods \
  --as=system:serviceaccount:equipo-a:sa-monitor \
  -n equipo-a

# ❌ El SA NO puede ver Secrets
kubectl auth can-i get secrets \
  --as=system:serviceaccount:equipo-a:sa-monitor \
  -n equipo-a
```

---

## Las tres formas de probar permisos

### 1 — `kubectl` con `--as` (ejecuta la operación real)

```bash
kubectl get pods -n equipo-a --as=juan
# Si falla: Error from server (Forbidden)
# Si funciona: devuelve los Pods
```

### 2 — `auth can-i` (simulación rápida)

```bash
kubectl auth can-i list pods -n equipo-a --as=juan
# yes  → tiene permiso
# no   → no tiene permiso
```

### 3 — `auth can-i --list` (ver todos los permisos)

```bash
kubectl auth can-i --list -n equipo-a --as=juan
# Tabla con todos los recursos y verbos que puede usar
```

---

## Diagnóstico rápido

```bash
# ¿Qué Roles existen en el namespace?
kubectl get roles -n equipo-a

# ¿Quién tiene qué Role?
kubectl get rolebindings -n equipo-a -o wide

# ¿Por qué no tiene permiso?
kubectl auth can-i <verbo> <recurso> -n <namespace> --as=<usuario>
```

---

## Lo que no puede hacer un Role

- Ver Nodes, PersistentVolumes o Namespaces (recursos sin namespace)
- Dar acceso a más de un namespace a la vez

Para eso existe el ClusterRole. Ver `../clusterrole/`.
