# Laboratorio integrador: «El clúster roto»

**Módulo 9 · Sesión 9** · Duración: 45 minutos · Trabajo por equipos de 2 o 3 personas

Seis aplicaciones en el namespace `lab-roto`, cada una con **un fallo distinto**. Son los mismos fallos de los escenarios 01 a 14, pero mezclados y sin etiquetas: hay que reconocerlos.

## Reglas

1. Aplicar el manifiesto y **no abrir `SOLUCIONES.md`**.
2. Usar el método: estado, `describe`, logs, `exec`, y los objetos de alrededor.
3. Corregir con `kubectl` o editando el archivo. **No** cambiar imágenes por otras distintas ni borrar el namespace.
4. Un equipo gana un punto por cada aplicación sana. Se anota el orden de resolución.

## Empezar

```text
kubectl apply -f 01-cluster-roto.yaml
kubectl get pods -n lab-roto
```

Estado inicial:

```text
NAME                        READY   STATUS                       RESTARTS   AGE
archivos-6d7f8c9b5-xk2tn    0/1     Pending                      0          90s
busquedas-7c8d9f6b4-q5wnm   0/1     CrashLoopBackOff             4          90s
catalogo-5b9c7d8f6-h4xvp    1/1     Running                      0          90s
monitor-6f8b7c9d5-t2kwz     1/1     Running                      0          90s
reportes-79d8c6b7f-m8xqn    0/1     CreateContainerConfigError   0          90s
tienda-8c7d9b6f5-r3tpk      0/1     ImagePullBackOff             0          90s
```

**Observe:** `catalogo` y `monitor` aparecen en `Running`, pero **también están rotas**. No todos los fallos se ven en `get pods`.

## Pistas escalonadas

Pida una pista solo si lleva más de 8 minutos en una aplicación.

| Nivel | Pista |
|---|---|
| 1 | Mire la columna `STATUS`: ¿qué dice de la aplicación? |
| 2 | ¿Qué comando de la tabla de estados le da el motivo? |
| 3 | ¿Qué objeto *rodea* a la aplicación? (Service, ConfigMap, PVC, ServiceAccount) |

## Cómo saber que terminó

```text
kubectl get pods -n lab-roto
kubectl get endpoints catalogo -n lab-roto
kubectl get pvc -n lab-roto
kubectl logs deploy/monitor -n lab-roto --tail=1
```

| Comprobación | Resultado esperado |
|---|---|
| `get pods` | Los seis Pods en `1/1 Running` |
| `get endpoints catalogo` | Una dirección IP, no `<none>` |
| `get pvc` | `archivos-pvc` en `Bound` |
| `logs deploy/monitor` | `HTTP 200` |

## Limpieza

```text
kubectl delete namespace lab-roto
```
