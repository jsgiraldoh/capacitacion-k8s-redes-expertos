# ResourceQuota y LimitRange

**Módulo 8 · Sesión 8 · Parte 4.3 de la guía**

| Objeto | Alcance | Qué hace |
|---|---|---|
| `ResourceQuota` | El namespace entero | Techo total de CPU, memoria y cantidad de objetos |
| `LimitRange` | Cada contenedor | Valores por defecto y límites mínimos y máximos |

## Archivos, en el orden en que se aplican

| Archivo | Qué demuestra |
|---|---|
| `00-namespace.yaml` | Crea el namespace `lab-quota` |
| `01-resourcequota.yaml` | La cuota: 5 Pods, 1 CPU y 1 Gi de requests |
| `02-pod-sin-recursos.yaml` | Un Pod sin `resources` (primero rechazado, luego aceptado) |
| `03-limitrange.yaml` | Valores por defecto que hacen válido al Pod anterior |
| `04-deployment-grande.yaml` | Superar la cuota: el error aparece en los eventos |

## Orden

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-resourcequota.yaml
kubectl describe quota cuota-lab -n lab-quota

kubectl apply -f 02-pod-sin-recursos.yaml
# ❌ Forbidden: must specify limits.cpu ...

kubectl apply -f 03-limitrange.yaml
kubectl apply -f 02-pod-sin-recursos.yaml
# ✅ pod/prueba created

kubectl apply -f 04-deployment-grande.yaml
kubectl get pods -n lab-quota
kubectl get events -n lab-quota --sort-by=.lastTimestamp
```

## La secuencia que se debe recordar

1. Se aplica la **cuota** → todos los Pods deben declarar `resources`.
2. Un Pod sin `resources` se **rechaza**.
3. Se aplica el **LimitRange** → los Pods sin `resources` reciben valores por defecto.
4. Los Pods se aceptan hasta **agotar la cuota**.

## Limpieza

```bash
kubectl delete namespace lab-quota
```
