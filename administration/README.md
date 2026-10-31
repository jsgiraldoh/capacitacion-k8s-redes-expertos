# Administración Operativa

**Módulo 8 · Sesión 8**

Operar el clúster día a día: sacar un nodo para mantenimiento, ponerle
límites a lo que consume cada Pod y cada namespace, y dejar que el
clúster escale solo.

## Orden de los ejercicios

| Orden | Carpeta | Qué se hace | Dónde está en la guía |
|---|---|---|---|
| 1 | [`01-mantenimiento-nodos/`](01-mantenimiento-nodos/) | `cordon`, `drain` y `uncordon` | Parte 3 |
| 2 | [`02-requests-limits/`](02-requests-limits/) | Pod que no cabe y Pod `OOMKilled` | Parte 4.1 y 4.2 |
| 3 | [`03-quota-limitrange/`](03-quota-limitrange/) | Cuota por namespace y valores por defecto | Parte 4.3 |
| 4 | [`04-hpa/`](04-hpa/) | Escalado automático con carga real | Parte 5 |

## Estructura

```
administration/
├── README.md
├── 01-mantenimiento-nodos/
│   ├── README.md
│   └── 01-deployment-app-mant.yaml
├── 02-requests-limits/
│   ├── README.md
│   ├── 01-pod-grande.yaml
│   └── 02-pod-oom.yaml
├── 03-quota-limitrange/
│   ├── README.md
│   ├── 00-namespace.yaml
│   ├── 01-resourcequota.yaml
│   ├── 02-pod-sin-recursos.yaml
│   ├── 03-limitrange.yaml
│   └── 04-deployment-grande.yaml
└── 04-hpa/
    ├── README.md
    ├── 00-namespace.yaml
    ├── 01-php-apache.yaml
    ├── 02-hpa.yaml
    └── 03-load-generator.yaml
```

## Requisitos previos

- Namespace `lab-s8`: `kubectl apply -f ../monitoring/00-namespace.yaml`
- Metrics Server funcionando (ver `../monitoring/01-metrics-server/`): obligatorio para el HPA.

## Si trabaja desde Windows (PowerShell)

Todos los comandos están en **una sola línea**. No use `\` para continuar
líneas: PowerShell 5.1 no lo entiende.

## Limpieza general

```bash
kubectl delete namespace lab-s8 lab-quota lab-hpa
kubectl get nodes
```

Al final, ningún nodo debe decir `SchedulingDisabled`.
