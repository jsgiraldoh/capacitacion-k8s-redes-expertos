# ReplicaSets

**Módulo 3 · Sesión 2**

El primer controlador: mantiene vivo un número determinado de Pods idénticos.

| Archivo | Qué contiene |
|---|---|
| `01-replicaset.yaml` | ReplicaSet `frontend` con 2 réplicas |

```bash
kubectl apply -f 01-replicaset.yaml
kubectl get rs
kubectl get pods -l tier=frontend -o wide
```

## Los dos experimentos

**Autoreparación** — con dos terminales abiertas:

```bash
# Terminal 1
kubectl get pods -l tier=frontend -w
# Terminal 2
kubectl delete pod <nombre-de-un-pod>
```

Verá morir uno y nacer otro en segundos. El ReplicaSet no "vio morir" el Pod:
simplemente comparó cuántos hay con cuántos quiere, y creó la diferencia.

**El selector** — "robarle" un Pod al controlador:

```bash
kubectl label pod <nombre> tier=huerfano --overwrite
kubectl get pods --show-labels
kubectl get rs
```

El Pod sigue corriendo, pero ya no le pertenece al ReplicaSet, así que este
crea otro. Termina con un Pod más de los que declaró. Esa es la prueba de que
la relación se establece **por etiquetas, nunca por nombre**.

Limpieza: `kubectl delete pod -l tier=huerfano`

## En la práctica

Casi nunca se crean ReplicaSets a mano. Se crea un Deployment, que los
gestiona por usted y añade actualizaciones progresivas y rollbacks.
