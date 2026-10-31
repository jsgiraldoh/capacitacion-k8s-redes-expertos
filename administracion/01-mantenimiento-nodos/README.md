# Mantenimiento de nodos

**Módulo 8 · Sesión 8 · Parte 3 de la guía**

Sacar un nodo de servicio de forma ordenada para hacerle mantenimiento
sin tumbar las aplicaciones.

## Archivos

| Archivo | Qué contiene |
|---|---|
| `01-deployment-app-mant.yaml` | Tres réplicas de nginx y los comandos de prueba |

## Los tres comandos

| Comando | Qué hace |
|---|---|
| `cordon` | El nodo deja de aceptar Pods nuevos. Los actuales siguen corriendo |
| `drain` | Hace `cordon` y expulsa los Pods existentes hacia otros nodos |
| `uncordon` | El nodo vuelve a aceptar Pods |

## Importante: nodo único

Con **un solo nodo** (típico de Docker Desktop) haga `cordon` y `uncordon`,
pero **no** `drain`: no hay otro nodo donde mover los Pods y las
aplicaciones quedan sin servicio.

## Orden

```bash
kubectl apply -f 01-deployment-app-mant.yaml
kubectl get nodes
kubectl cordon NOMBRE-DEL-NODO
kubectl scale deployment app-mant --replicas=5 -n lab-s8
kubectl get pods -o wide -n lab-s8
kubectl drain NOMBRE-DEL-NODO --ignore-daemonsets --delete-emptydir-data
kubectl uncordon NOMBRE-DEL-NODO
```

`--ignore-daemonsets` es necesario porque los Pods de DaemonSets
(como `kube-proxy`) no se pueden evacuar.

## Limpieza

```bash
kubectl delete -f 01-deployment-app-mant.yaml
kubectl get nodes
```
