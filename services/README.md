# Services

**Módulo 4 · Sesión 3**

Darle una dirección estable a un conjunto de Pods que cambian.

## Aplicar el Deployment PRIMERO

```bash
kubectl apply -f 00-deployment-nginx.yaml
```

Los tres ejemplos comparten este Deployment. La imagen `nginxdemos/hello`
muestra el nombre del Pod que atendió la petición, así que al refrescar se
**ve** el balanceo de carga.

> **Cambio respecto a la versión anterior:** antes cada subcarpeta traía su
> propio Deployment llamado `nginx-deployment`, con imágenes distintas. Al
> aplicar uno después de otro se sobrescribían, y el resultado dependía del
> orden. Ahora el Deployment está una sola vez y los Services apuntan a él.

## Los tres tipos

| Carpeta | Archivo | Quién puede acceder |
|---|---|---|
| `clusterip/` | `01-service-clusterip.yaml` | Solo Pods del clúster |
| `nodeport/` | `02-service-nodeport.yaml` | Cualquiera con acceso a un nodo, puerto 30080 |
| `lb/` | `03-service-loadbalancer.yaml` | Clientes externos, IP pública dedicada |

Los tres Services se llaman `nginx-service` a propósito: aplicar uno cambia
el tipo del anterior, lo que permite ver la transición sobre el mismo objeto.

## La comprobación que nunca hay que saltarse

```bash
kubectl get endpoints nginx-service
```

Si los Endpoints están **vacíos**, el selector no coincide con ningún Pod y
nada funcionará — sin mostrar ningún error. Es el síntoma número uno de "mi
Service no responde".

## Notas por entorno

- **NodePort**: el puerto debe estar abierto en el firewall de **todos** los
  nodos, no solo de uno.
- **LoadBalancer en DigitalOcean**: funciona, y se factura por hora
  (~12 USD/mes). Borre el Service al terminar.
- **LoadBalancer on-premise**: queda en `<pending>` para siempre. Haría falta
  MetalLB. Use el NodePort que se crea junto con él.
