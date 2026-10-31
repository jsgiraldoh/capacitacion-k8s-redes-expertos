# Monitoreo y Observabilidad

**Módulo 7 · Sesión 8**

¿Qué está pasando en el clúster ahora mismo? Esta carpeta responde con tres
herramientas, de la más simple a la más completa.

## Orden de los ejercicios

| Orden | Carpeta | Qué se hace | Dónde está en la guía |
|---|---|---|---|
| 0 | `00-namespace.yaml` | Crear el namespace `lab-s8` | Parte 0 |
| 1 | [`01-metrics-server/`](01-metrics-server/) | Instalar Metrics Server y usar `kubectl top` | Parte 1 |
| 2 | [`02-probes/`](02-probes/) | Liveness, readiness, logs y eventos | Parte 2 |
| 3 | [`03-prometheus-grafana/`](03-prometheus-grafana/) | Historial y dashboards (demostración) | Parte 6 |

## Estructura

```
monitoreo/
├── README.md
├── 00-namespace.yaml
├── 01-metrics-server/
│   ├── README.md
│   └── metrics-patch.json
├── 02-probes/
│   ├── README.md
│   ├── 01-web-probe.yaml
│   ├── 02-web-probe-liveness-rota.yaml
│   ├── 03-web-ready-rota.yaml
│   └── 04-web-ready-ok.yaml
└── 03-prometheus-grafana/
    ├── README.md
    └── values-local.yaml
```

## Antes de empezar

```bash
kubectl apply -f 00-namespace.yaml
kubectl config set-context --current --namespace=lab-s8
```

## Si trabaja desde Windows (PowerShell)

Todos los comandos de estos README están escritos en **una sola línea**.
No use `\` para continuar líneas: PowerShell 5.1 no lo entiende.

## Limpieza

```bash
kubectl delete namespace lab-s8
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

Metrics Server se deja instalado: las sesiones siguientes lo necesitan.
