# Red y Services

**Módulo 9 · Sesión 9** · Repasa la Sesión 3

El Pod está bien, pero nadie logra conectarse.

## Escenarios de esta carpeta

| Escenario | Archivo | Qué falla |
|---|---|---|
| 07 | `07-service-selector-roto.yaml` | El Service apunta a una etiqueta que ningún Pod tiene |
| 08 | `08-service-targetport-roto.yaml` | El Service apunta al puerto 8080 y la app escucha en el 80 |
| 09 | `09-dns-cliente-roto.yaml` | El cliente llama a `api` y el Service está en otro namespace |

## Cómo trabajar cada escenario

1. Aplicar el archivo **roto**.
2. Observar el síntoma con `kubectl get`.
3. Diagnosticar con el método: `describe`, eventos, logs, `exec`.
4. Corregir y comprobar que el recurso queda sano.

Todos los comandos están en **una sola línea** (compatibles con PowerShell).
Antes de empezar: `kubectl apply -f ../00-namespace.yaml` y `kubectl apply -f ../00-cliente.yaml`.

---

## Escenario 07 · Service sin endpoints: el selector no coincide

**Repasa:** Sesión 3 (Services)  
**Archivo:** `07-service-selector-roto.yaml`

### Síntoma

Los Pods están en `Running`, pero al conectarse al Service la conexión falla. (Se usa el Pod `cliente` de `00-cliente.yaml`, con `curl` y `nslookup`.)

```text
$ kubectl get pods -l app=web-red
NAME                      READY   STATUS    RESTARTS   AGE
web-red-7f8d9c6b5-4xkqm   1/1     Running   0          40s
web-red-7f8d9c6b5-pn2wz   1/1     Running   0          40s
$ kubectl exec deploy/cliente -n lab-ts -- curl -s -m 3 http://web-red
curl: (7) Failed to connect to web-red port 80 after 3 ms: Couldn't connect to server
```

### Diagnóstico

El comando clave de red es `kubectl get endpoints`: si la lista está vacía, el Service no encuentra Pods. Se compara el selector del Service con las etiquetas reales de los Pods.

```text
$ kubectl get endpoints web-red -n lab-ts
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME      ENDPOINTS   AGE
web-red   <none>      55s
$ kubectl describe service web-red -n lab-ts
Name:              web-red
Namespace:         lab-ts
Selector:          app=web-rojo
Type:              ClusterIP
IP:                10.96.142.37
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         <none>
$ kubectl get pods -n lab-ts --show-labels -l app=web-red
NAME                      READY   STATUS    RESTARTS   AGE   LABELS
web-red-7f8d9c6b5-4xkqm   1/1     Running   0          55s   app=web-red,pod-template-hash=7f8d9c6b5
```

### Causas posibles

- El selector del Service no coincide con las etiquetas de los Pods (este escenario).
- Los Pods existen, pero ninguno está listo (readiness probe fallando): los Pods no listos salen de los endpoints.
- El Service está en otro namespace que los Pods.
- `targetPort` equivocado (escenario 08).


### Solución

`kubectl set selector` corrige el selector sin editar archivos. Los endpoints aparecen de inmediato.

```text
$ kubectl set selector service web-red app=web-red -n lab-ts
service/web-red selector updated
$ kubectl get endpoints web-red -n lab-ts
NAME      ENDPOINTS                   AGE
web-red   10.1.0.61:80,10.1.0.62:80   2m
$ kubectl exec deploy/cliente -n lab-ts -- curl -s -m 3 -o /dev/null -w "%{http_code}" http://web-red
200
```

> **Lección.** Si un Service no responde, `get endpoints` es el primer comando: vacío significa que el selector no encuentra Pods.

**Pregunta para el grupo:** El Pod está `Running` y `1/1`. ¿Por qué el Service aun así no le envía tráfico?


---

## Escenario 08 · targetPort equivocado: hay endpoints, pero no hay respuesta

**Repasa:** Sesión 3 (Services)  
**Archivo:** `08-service-targetport-roto.yaml`

### Síntoma

Todo se ve bien: Pod `Running` y Service con endpoints. Pero la conexión es rechazada.

```text
$ kubectl get pods,endpoints -l app=web-puerto -n lab-ts
NAME                              READY   STATUS    RESTARTS   AGE
pod/web-puerto-6c7b8d9f5-xk4tn    1/1     Running   0          1m

NAME                   ENDPOINTS         AGE
endpoints/web-puerto   10.1.0.64:8080    1m
$ kubectl exec deploy/cliente -n lab-ts -- curl -s -m 3 http://web-puerto
curl: (7) Failed to connect to web-puerto port 80 after 4 ms: Couldn't connect to server
```

### Diagnóstico

Hay endpoints, así que el selector está bien. La pista es el puerto de cada endpoint: 8080. Se compara con el puerto en que el contenedor realmente escucha y se prueba directo contra la IP del Pod, saltándose el Service.

```text
$ kubectl describe pod -l app=web-puerto -n lab-ts
...
    Port:           80/TCP
...
$ kubectl exec deploy/cliente -n lab-ts -- curl -s -m 3 -o /dev/null -w "%{http_code}" http://10.1.0.64:80
200
$ kubectl exec deploy/cliente -n lab-ts -- curl -s -m 3 -o /dev/null -w "%{http_code}" http://10.1.0.64:8080
000
```

### Causas posibles

- `targetPort` distinto del puerto en el que escucha la aplicación (este escenario).
- La aplicación escucha solo en `127.0.0.1` y no en `0.0.0.0` dentro del contenedor.
- `containerPort` mal declarado: es informativo y no abre ni cierra puertos.
- Probar el puerto del Service en lugar del puerto de la aplicación.


### Solución

Se corrige `targetPort: 80` en el archivo y se vuelve a aplicar. Los endpoints cambian al puerto correcto.

```text
$ kubectl apply -f 08-service-targetport-roto.yaml
deployment.apps/web-puerto unchanged
service/web-puerto configured
$ kubectl exec deploy/cliente -n lab-ts -- curl -s -m 3 -o /dev/null -w "%{http_code}" http://web-puerto
200
```

Cambio en el archivo `08-service-targetport-roto.yaml`:

```yaml
  ports:
  - port: 80
    targetPort: 80                    # ← el puerto en que escucha nginx
```

> **Lección.** Probar contra la IP del Pod separa los problemas: si el Pod responde y el Service no, el fallo está en el Service.

**Pregunta para el grupo:** ¿Qué diferencia hay entre `port`, `targetPort` y `containerPort`?


---

## Escenario 09 · DNS: el nombre corto no resuelve en otro namespace

**Repasa:** Sesión 2 (Namespaces) y Sesión 3 (DNS de los Services)  
**Archivo:** `09-dns-cliente-roto.yaml`  
**Antes:** aplicar `09-dns-backend.yaml`

### Síntoma

El backend está sano y su Service tiene endpoints, pero el cliente falla en cada intento. Los logs del cliente lo muestran.

```text
$ kubectl apply -f 09-dns-backend.yaml
namespace/lab-ts-backend created
deployment.apps/api created
service/api created
$ kubectl apply -f 09-dns-cliente-roto.yaml
deployment.apps/cliente-dns created
$ kubectl logs deploy/cliente-dns -n lab-ts --tail=3
FALLO: no se pudo conectar a http://api
FALLO: no se pudo conectar a http://api
FALLO: no se pudo conectar a http://api
```

### Diagnóstico

Se separa el problema en dos preguntas: ¿resuelve el nombre? y ¿responde el servicio? `nslookup` contesta la primera. El archivo `resolv.conf` explica por qué el nombre corto no alcanza: la búsqueda empieza por el namespace del propio Pod.

```text
$ kubectl exec deploy/cliente-dns -n lab-ts -- nslookup api
Server:         10.96.0.10
Address:        10.96.0.10#53

** server can't find api: NXDOMAIN
$ kubectl exec deploy/cliente-dns -n lab-ts -- cat /etc/resolv.conf
search lab-ts.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
$ kubectl exec deploy/cliente-dns -n lab-ts -- nslookup api.lab-ts-backend
Name:   api.lab-ts-backend.svc.cluster.local
Address: 10.96.88.14
```

### Causas posibles

- Nombre corto usado desde otro namespace (este escenario).
- Service inexistente o con el nombre mal escrito (`NXDOMAIN` también).
- CoreDNS caído (`kubectl get pods -n kube-system -l k8s-app=kube-dns`): ningún nombre resuelve.
- El Pod usa `dnsPolicy` distinto del predeterminado.


### Solución

Se cambia la URL a `http://api.lab-ts-backend` en el archivo del cliente y se vuelve a aplicar. El nombre completo también funciona: `api.lab-ts-backend.svc.cluster.local`.

```text
$ kubectl apply -f 09-dns-cliente-roto.yaml
deployment.apps/cliente-dns configured
$ kubectl logs deploy/cliente-dns -n lab-ts --tail=2
OK: api responde
OK: api responde
```

Cambio en el archivo `09-dns-cliente-roto.yaml`:

```yaml
            if curl -s -m 2 -o /dev/null http://api.lab-ts-backend; then
              echo "OK: api responde"
```

> **Lección.** `<servicio>` funciona dentro del namespace; `<servicio>.<namespace>` funciona desde cualquier otro.

**Pregunta para el grupo:** ¿Qué componente del clúster contesta las consultas `nslookup`, y cómo comprobaría que está sano?


---
