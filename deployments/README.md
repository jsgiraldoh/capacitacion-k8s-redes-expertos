# Deployments

**Módulos 3 y 4 · Sesiones 2 y 3**

| Archivo | Qué contiene |
|---|---|
| `01-deployment-cliente.yaml` | Pod con `netshoot`, la caja de herramientas de red |

```bash
kubectl apply -f 01-deployment-cliente.yaml
kubectl exec -it deploy/client-deployment -- bash
```

## Para qué sirve este cliente

Es la herramienta de diagnóstico que usaremos en todas las pruebas de
conectividad del Módulo 4. Trae `curl`, `dig`, `nslookup`, `tcpdump`,
`netstat`, `nmap` e `iperf`.

```bash
# Probar un Service por su nombre
kubectl exec deploy/client-deployment -- curl -s http://nginx-service

# Resolver DNS interno
kubectl exec deploy/client-deployment -- nslookup nginx-service
kubectl exec deploy/client-deployment -- \
  dig nginx-service.default.svc.cluster.local
```

## Comandos de ciclo de vida

Aquí es donde se ve la diferencia con un ReplicaSet suelto:

```bash
kubectl scale deploy/client-deployment --replicas=3   # escalar
kubectl set image deploy/nginx-deployment nginx=nginx:1.27  # actualizar
kubectl rollout status deploy/nginx-deployment        # seguir el avance
kubectl rollout history deploy/nginx-deployment       # ver revisiones
kubectl rollout undo deploy/nginx-deployment          # volver atrás
```
