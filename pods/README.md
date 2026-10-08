# Pods

**Módulo 3 · Sesión 2**

La unidad mínima de despliegue: uno o más contenedores que comparten IP y
espacio de red.

| Archivo | Qué contiene |
|---|---|
| `01-pod-basico.yaml` | Un Pod suelto con nginx, creado a mano |

```bash
kubectl apply -f 01-pod-basico.yaml
kubectl get pods -o wide
kubectl describe pod nginx
```

## El punto de la carpeta

Un Pod suelto **no se recupera solo**. Compruébelo:

```bash
kubectl delete pod nginx
kubectl get pods        # desapareció, y nadie lo recreó
```

Esa limitación es la razón de existir de `../replicasets/` y
`../deployments/`.
