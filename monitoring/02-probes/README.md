# Probes, logs y eventos

**Módulo 7 · Sesión 8 · Parte 2 de la guía**

Cómo decide Kubernetes si un contenedor está vivo y si puede recibir
tráfico, y cómo leer las dos fuentes de diagnóstico: eventos y logs.

## Archivos, en el orden en que se aplican

| Archivo | Qué demuestra |
|---|---|
| `01-web-probe.yaml` | Liveness y readiness sanas: todo en `1/1 Running` |
| `02-web-probe-liveness-rota.yaml` | Liveness en `/noexiste`: reinicios y `CrashLoopBackOff` |
| `03-web-ready-rota.yaml` | Readiness rota: Pods vivos pero Service sin destinos |
| `04-web-ready-ok.yaml` | Readiness corregida: los Pods vuelven al Service |

## Orden de aplicación

```bash
kubectl apply -f 01-web-probe.yaml
kubectl apply -f 02-web-probe-liveness-rota.yaml
kubectl get pods -w
kubectl apply -f 01-web-probe.yaml
kubectl apply -f 03-web-ready-rota.yaml
kubectl get endpoints web-ready
kubectl apply -f 04-web-ready-ok.yaml
```

Cada archivo trae al final sus propios comandos de prueba y lo que debe verse.

## Liveness frente a readiness

| | Liveness | Readiness |
|---|---|---|
| Pregunta | ¿Está vivo? | ¿Puede recibir tráfico? |
| Si falla | Reinicia el contenedor | Saca el Pod del Service |
| `RESTARTS` | Aumenta | No cambia |
| Síntoma visible | `CrashLoopBackOff` | `READY 0/1` y `ENDPOINTS` vacío |
| Úsela para | Apps que se bloquean sin morir | Arranque lento o carga temporal |

## Logs y eventos

```bash
# Lo que Kubernetes hizo
kubectl get events --sort-by=.lastTimestamp

# Lo que la aplicación dijo
kubectl logs NOMBRE-DEL-POD
kubectl logs NOMBRE-DEL-POD --previous
```

`--previous` muestra los logs del contenedor **antes** del reinicio.
Cuando un Pod no arranca, empiece por los eventos; cuando arranca pero
falla, siga con los logs.

## Limpieza

```bash
kubectl delete -f 04-web-ready-ok.yaml
kubectl delete -f 01-web-probe.yaml
```
