# Troubleshooting

**Módulo 9 · Sesión 9**

Cuando algo falla en Kubernetes, lo difícil no es la solución: es saber **por dónde empezar**. Esta carpeta tiene 15 escenarios rotos a propósito, uno por cada tipo de fallo que el grupo ya conoce de las sesiones anteriores, más un laboratorio integrador.

> **Aviso.** Estos manifiestos fallan **a propósito**. No los sincronice con Argo CD: aplíquelos a mano con `kubectl apply`.

## El método: cinco preguntas, siempre en el mismo orden

| # | Pregunta | Comando |
|---|---|---|
| 1 | ¿Qué dice el estado? | `kubectl get pods` |
| 2 | ¿Qué dice Kubernetes? | `kubectl describe pod NOMBRE` (los **eventos**, al final) |
| 3 | ¿Qué dice la aplicación? | `kubectl logs NOMBRE` y `kubectl logs NOMBRE --previous` |
| 4 | ¿Qué ve desde dentro? | `kubectl exec` con `curl` y `nslookup` |
| 5 | ¿Está bien lo de alrededor? | Service y endpoints, PVC, ServiceAccount, nodo |

## De qué estado partir

| Estado | Primer comando | Qué buscar |
|---|---|---|
| `Pending` | `kubectl describe pod` | Eventos del Scheduler: recursos, `nodeSelector`, `cordon`, PVC |
| `ImagePullBackOff` / `ErrImagePull` | `kubectl describe pod` | Mensaje `not found`, `unauthorized` o `timeout` |
| `CrashLoopBackOff` | `kubectl logs POD --previous` | El error de la aplicación antes de morir |
| `CreateContainerConfigError` | `kubectl describe pod` | Secret, ConfigMap o `securityContext` que no se cumple |
| `Running` pero `0/1` | `kubectl describe pod` | Readiness probe que falla |
| `Running` y `1/1`, pero no responde | `kubectl get endpoints SERVICE` | Selector, `targetPort`, namespace, DNS |
| `Running`, pero la app dice `403` | `kubectl auth can-i --as=...` | Role y RoleBinding del ServiceAccount |
| `OOMKilled` | `kubectl describe pod` | `Last State` y `limits.memory` |
| Nodo `NotReady` | `kubectl describe node` | `Conditions` y, en el nodo, `journalctl -u kubelet` |

## Orden de los ejercicios

| Escenario | Archivo | Qué falla | Repasa |
|---|---|---|---|
| 01 | [01-imagepullbackoff-roto.yaml](01-pods/01-imagepullbackoff-roto.yaml) | Etiqueta de imagen inexistente | Sesión 2 |
| 02 | [02-crashloopbackoff-roto.yaml](01-pods/02-crashloopbackoff-roto.yaml) | La app sale con código 1 por falta de una variable | Sesión 2 |
| 03 | [03-pending-nodeselector-roto.yaml](01-pods/03-pending-nodeselector-roto.yaml) | nodeSelector con una etiqueta que ningún nodo tiene | Sesión 1 |
| 04 | [04-config-error-roto.yaml](01-pods/04-config-error-roto.yaml) | El Pod referencia un Secret que no existe | Sesión 6 |
| 05 | [05-selector-roto.yaml](02-controladores/05-selector-roto.yaml) | El selector no coincide con las etiquetas del Pod | Sesión 2 |
| 06 | [06-rollout-v2-roto.yaml](02-controladores/06-rollout-v2-roto.yaml) | Actualización con una imagen inexistente | Sesión 2 |
| 07 | [07-service-selector-roto.yaml](03-red/07-service-selector-roto.yaml) | El Service apunta a una etiqueta que ningún Pod tiene | Sesión 3 |
| 08 | [08-service-targetport-roto.yaml](03-red/08-service-targetport-roto.yaml) | El Service apunta al puerto 8080 y la app escucha en el 80 | Sesión 3 |
| 09 | [09-dns-cliente-roto.yaml](03-red/09-dns-cliente-roto.yaml) | El cliente llama a `api` y el Service está en otro namespace | Sesión 2 |
| 10 | [10-pvc-storageclass-roto.yaml](04-almacenamiento/10-pvc-storageclass-roto.yaml) | El PVC pide una StorageClass inexistente | Sesión 5 |
| 11 | [11-rbac-forbidden-roto.yaml](05-seguridad/11-rbac-forbidden-roto.yaml) | Una aplicación recibe 403 al llamar al API Server | Sesión 6 |
| 12 | [12-runasnonroot-roto.yaml](05-seguridad/12-runasnonroot-roto.yaml) | securityContext exige un usuario no root y la imagen usa root | Sesión 6 |
| 13 | [13-psa-restricted-roto.yaml](05-seguridad/13-psa-restricted-roto.yaml) | Un namespace con nivel restricted rechaza un Pod sin securityContext | Sesión 6 |
| 14 | [14-hpa-sin-requests-roto.yaml](06-operacion/14-hpa-sin-requests-roto.yaml) | El HPA no puede calcular el porcentaje de CPU | Sesión 8 |
| 15 | [15-nodo-cordon-roto.yaml](06-operacion/15-nodo-cordon-roto.yaml) | Todos los nodos están marcados como no programables | Sesión 8 |
| — | [07-integrador/](07-integrador/) | Seis fallos mezclados en `lab-roto` | Todo lo anterior |

## Estructura

```
troubleshooting/
├── README.md
├── 00-namespace.yaml
├── 00-cliente.yaml
├── 01-pods/                 escenarios 01 a 04
├── 02-controladores/        escenarios 05 y 06
├── 03-red/                  escenarios 07 a 09
├── 04-almacenamiento/       escenario 10
├── 05-seguridad/            escenarios 11 a 13
├── 06-operacion/            escenarios 14 y 15, y nodo NotReady
└── 07-integrador/           «El clúster roto»
```

## Antes de empezar

```text
kubectl apply -f 00-namespace.yaml
kubectl apply -f 00-cliente.yaml
kubectl config set-context --current --namespace=lab-ts
```

- El escenario 14 necesita **Metrics Server** (ver `../monitoring/01-metrics-server/`).
- La demostración de nodo `NotReady` necesita un clúster **kubeadm** con varios nodos.

## Si trabaja desde Windows (PowerShell)

Todos los comandos están en **una sola línea**. Donde hay dos comandos seguidos, escríbalos por separado (o únalos con `;`).

## Limpieza

```text
kubectl delete namespace lab-ts lab-psa lab-ts-backend lab-roto
kubectl label node docker-desktop disktype-
kubectl get nodes
```

Al final, ningún nodo debe decir `SchedulingDisabled`.
