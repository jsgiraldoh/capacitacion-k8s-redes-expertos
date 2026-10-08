# Secrets

**Módulo 6 · visto de forma anticipada en la Sesión 4**

Llegamos aquí por la puerta lateral: para entrar a Argo CD hubo que leer el
secreto `argocd-initial-admin-secret`.

## Aplicar en orden

| Archivo | Objeto | Papel |
|---|---|---|
| `01-secret.yaml` | Secret | Los datos sensibles |
| `02-deployment-podinfo.yaml` | Deployment | Los consume de las dos formas posibles |
| `03-service-lb.yaml` | Service | Para verlo en el navegador |

```bash
kubectl apply -f 01-secret.yaml
kubectl apply -f 02-deployment-podinfo.yaml
kubectl apply -f 03-service-lb.yaml
```

> **Cambio respecto a la versión anterior:** los tres objetos estaban en un
> solo archivo, con el Service declarado antes del Deployment. Separarlos
> hace explícito el orden y permite aplicar o borrar cada pieza por separado.

## Las dos formas de consumir un Secret

El mismo Pod las usa a la vez:

```bash
# Como ARCHIVOS: cada clave del Secret es un archivo
kubectl exec deploy/demo-secretos -- ls -la /etc/secretos
kubectl exec deploy/demo-secretos -- cat /etc/secretos/db-password

# Como VARIABLE DE ENTORNO
kubectl exec deploy/demo-secretos -- printenv PODINFO_UI_MESSAGE
```

### La diferencia que casi nadie explica

Cambie el valor de `MENSAJE` en `01-secret.yaml`, aplique, y observe:

- El **archivo montado** se actualiza solo, en aproximadamente un minuto
- La **variable de entorno** no cambia: se inyectó al crear el Pod

Para que la variable tome el valor nuevo hay que reiniciar:

```bash
kubectl rollout restart deploy/demo-secretos
```

## Leer el valor en Windows

`base64` no existe en PowerShell. Use la versión multiplataforma:

```powershell
kubectl get secret demo-secret -o go-template="{{.data.MENSAJE | base64decode}}"
```

## La advertencia importante

**base64 no es cifrado, solo codificación.** Cualquiera con permisos de
lectura sobre el namespace puede decodificar el Secret.

Argo CD muestra los valores como `***`, pero el dato sí es visible con
`kubectl` o con Lens. Ocultarlo en una interfaz no es protegerlo.

Para Secrets reales en Git: **Sealed Secrets** o **External Secrets
Operator**, que guardan en el repositorio una versión cifrada que solo el
clúster puede descifrar.
