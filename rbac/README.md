# RBAC — Control de Acceso Basado en Roles

**Módulo 6 · Sesión 6**

RBAC responde a una sola pregunta: **¿quién puede hacer qué sobre qué recurso?**

## Estructura

```
rbac/
├── role/                         # Permisos dentro de un namespace
│   ├── 00-namespaces.yaml        # Preparación del entorno
│   ├── 01-role-user.yaml         # Sujeto: usuario individual
│   ├── 02-role-group.yaml        # Sujeto: grupo de usuarios
│   └── 03-role-serviceaccount.yaml  # Sujeto: aplicación en el clúster
└── clusterrole/                  # Permisos sin límite de namespace
    ├── 01-clusterrole.yaml       # Acceso a recursos globales y todos los namespaces
    └── 02-clusterrole-como-plantilla.yaml  # ClusterRole + RoleBinding (patrón reutilizable)
```

## Los cuatro objetos

| Objeto | Ámbito | Asignado con |
|---|---|---|
| `Role` | Un namespace | `RoleBinding` |
| `RoleBinding` | Un namespace | — |
| `ClusterRole` | Todo el clúster | `RoleBinding` o `ClusterRoleBinding` |
| `ClusterRoleBinding` | Todo el clúster | — |

## Por dónde empezar

```bash
# 1. Preparar el entorno
kubectl apply -f role/00-namespaces.yaml

# 2. Role con los tres tipos de sujeto
kubectl apply -f role/01-role-user.yaml
kubectl apply -f role/02-role-group.yaml
kubectl apply -f role/03-role-serviceaccount.yaml

# 3. ClusterRole
kubectl apply -f clusterrole/01-clusterrole.yaml
kubectl apply -f clusterrole/02-clusterrole-como-plantilla.yaml
```

## Los tres comandos de diagnóstico

```bash
# Simular una operación como un usuario
kubectl auth can-i list pods -n equipo-a --as=juan

# Ver todos los permisos de un usuario
kubectl auth can-i --list -n equipo-a --as=juan

# Ejecutar la operación real (si falla, muestra el error real)
kubectl get pods -n equipo-a --as=juan
```
