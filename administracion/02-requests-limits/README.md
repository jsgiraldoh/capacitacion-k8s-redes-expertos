# Requests y limits

**Módulo 8 · Sesión 8 · Partes 4.1 y 4.2 de la guía**

| Campo | Qué significa | Quién lo usa |
|---|---|---|
| `requests` | Lo que el Pod **reserva** | El Scheduler, para elegir nodo |
| `limits` | El **techo** que no puede superar | El kernel, en tiempo de ejecución |

## Archivos

| Archivo | Qué demuestra |
|---|---|
| `01-pod-grande.yaml` | Requests imposibles: el Pod queda `Pending` |
| `02-pod-oom.yaml` | Límite de memoria superado: `OOMKilled` |

## Orden

```bash
kubectl apply -f 01-pod-grande.yaml
kubectl describe pod pod-grande -n lab-s8
kubectl delete pod pod-grande -n lab-s8

kubectl apply -f 02-pod-oom.yaml
kubectl get pod pod-oom -n lab-s8 -w
kubectl describe pod pod-oom -n lab-s8
kubectl delete pod pod-oom -n lab-s8
```

## CPU frente a memoria

| Recurso | Al superar el límite |
|---|---|
| CPU | Se frena (throttling). El contenedor sigue vivo |
| Memoria | Se mata (`OOMKilled`, código 137) |

## Sin requests ni limits

Un Pod puede consumir toda la CPU del nodo y dejar sin recursos a los
demás, y el Scheduler no puede planificar bien. Los siguientes ejercicios
(`03-quota-limitrange/`) muestran cómo evitarlo a nivel de namespace.
