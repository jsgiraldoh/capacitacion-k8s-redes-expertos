# HPA: escalado automático

**Módulo 8 · Sesión 8 · Parte 5 de la guía**

El HPA agrega y quita réplicas según la carga real.

## Los tres requisitos

1. **Metrics Server funcionando.** Si `kubectl top pods` no responde, el HPA tampoco.
2. **`requests.cpu` en el Deployment.** El porcentaje se calcula sobre el request.
3. **Carga real.** Con CPU en reposo no hay nada que escalar.

## Archivos, en el orden en que se aplican

| Archivo | Qué contiene |
|---|---|
| `00-namespace.yaml` | Namespace `lab-hpa` |
| `01-php-apache.yaml` | Deployment y Service de la aplicación (sin campo `replicas`) |
| `02-hpa.yaml` | El HPA: objetivo 50%, entre 1 y 5 réplicas |
| `03-load-generator.yaml` | El Pod que genera la carga |

## Orden

```bash
kubectl top nodes
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-php-apache.yaml
kubectl top pods -n lab-hpa
kubectl apply -f 02-hpa.yaml
kubectl get hpa -n lab-hpa
```

Luego, con **dos terminales**:

```bash
# Terminal 1: observar
kubectl get hpa php-apache -n lab-hpa --watch

# Terminal 2: generar carga
kubectl apply -f 03-load-generator.yaml
```

Para quitar la carga:

```bash
kubectl delete pod load-generator -n lab-hpa
```

## Lo que debe verse

| Tiempo | TARGETS | REPLICAS |
|---|---|---|
| 0:00 | `0%/50%` | 1 |
| 0:30 a 1:00 | `200% a 250%/50%` | 1 |
| 1:00 a 1:30 | `250%/50%` | 4 o 5 |
| 2:00 a 3:00 | `50% a 70%/50%` | 5 |
| Sin carga, tras ~1 min | `0%/50%` | vuelve a 1 |

## La fórmula

```
réplicas deseadas = redondeo hacia arriba de (réplicas actuales x uso actual / objetivo)
```

Ejemplo: 1 réplica al 250% con objetivo 50% da 1 x 250 / 50 = 5 réplicas.

## Si el HPA no hace lo esperado

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `TARGETS <unknown>` | Metrics Server caído o sin `requests.cpu` | `kubectl top pods -n lab-hpa` y revisar el YAML |
| `missing request for cpu` | El contenedor no declara `requests.cpu` | Agregarlo y aplicar |
| `TARGETS` sube y `REPLICAS` no | No pasó el ciclo de 15 s o llegó a `maxReplicas` | Esperar 1 minuto y `kubectl describe hpa` |
| `TARGETS` no sube con carga | Carga insuficiente | Segundo generador (ver `03-load-generator.yaml`) |
| Réplicas en `Pending` | El clúster no tiene CPU libre | Bajar `maxReplicas` |

## Aviso para quien usa Argo CD

Si el Deployment está en Git con un `replicas` fijo y Argo CD tiene
sincronización automática, Argo CD revertirá lo que haga el HPA. Por eso
`01-php-apache.yaml` no declara `replicas`. La alternativa es configurar
`ignoreDifferences` sobre ese campo.

## Limpieza

```bash
kubectl delete namespace lab-hpa
```
