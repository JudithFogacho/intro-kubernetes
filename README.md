# Intro Kubernetes: Deployment y ReplicaSet con Redis y Grafana

Práctica de **Procesos de Software**. Despliegue en un clúster local de
**minikube** de dos plataformas distintas a las vistas en clase:

| Objeto | Plataforma | Archivo | Réplicas | Exposición |
|---|---|---|---|---|
| ReplicaSet `redis-replicaset` | Redis 7.4 | `replicaset/replicaset_redis.yml` | 3 | Service `ClusterIP` `redis-service:6379` (interno) |
| Deployment `grafana-deployment` | Grafana 11.3.0 | `deployment/deployment_grafana.yml` | 2 | Service `NodePort` `grafana-service:30300` (externo) |

## Estructura

```
pod_gnix/first_pod_nginx.yml        Primer Pod (nginx)
configmap/redis_configmap.yml       redis.conf (maxmemory 64mb, allkeys-lru)
configmap/grafana_configmap.yml     Variables GF_* de Grafana
replicaset/replicaset_redis.yml     ReplicaSet de Redis (3 réplicas)
deployment/deployment_grafana.yml   Deployment de Grafana (2 réplicas, RollingUpdate)
service/redis_service.yml           Service ClusterIP para Redis
service/grafana_service.yml         Service NodePort para Grafana
capturas/                           Evidencias
```

## Cómo ejecutarlo

```bash
minikube start --driver=docker
kubectl apply -f configmap/
kubectl apply -f replicaset/ -f deployment/ -f service/
kubectl get deploy,rs,pods,svc
```

Abrir Grafana desde el navegador:

```bash
minikube service grafana-service --url
```

En Linux o con el driver de VM también funciona `http://$(minikube ip):30300`.
En macOS con el driver Docker hay que usar el comando anterior, que abre un túnel.

Probar Redis a través de su Service:

```bash
kubectl run redis-client --image=redis:7.4 --restart=Never --command -- sleep 3600
kubectl exec redis-client -- redis-cli -h redis-service ping    # PONG
```

## Configuración

- **Redis**: el ConfigMap `redis-config` se monta como archivo en
  `/usr/local/etc/redis/redis.conf` y el contenedor arranca con
  `redis-server /usr/local/etc/redis/redis.conf`. Se verifica con
  `redis-cli CONFIG GET maxmemory` → `67108864` (64 MB).
- **Grafana**: el ConfigMap `grafana-config` se inyecta como variables de entorno
  con `envFrom` (tema oscuro, acceso anónimo como Viewer, URL raíz).
  Incluye `readinessProbe` sobre `/api/health`, así el Service solo envía
  tráfico a pods listos.
- Ambos contenedores declaran `requests` y `limits` de CPU y memoria.

## Conceptos demostrados

| Funcionalidad | Objeto | Comando | Evidencia |
|---|---|---|---|
| Mantener N réplicas | ReplicaSet | `kubectl get rs` | `capturas/03_objetos.png` |
| Autorreparación | ReplicaSet | `kubectl delete pod <redis>` | `capturas/08_autorreparacion.png` |
| Escalado | ReplicaSet | `kubectl scale rs redis-replicaset --replicas=5` | `capturas/09_escalado.png` |
| Actualización sin caída | Deployment | `kubectl set image deployment/grafana-deployment grafana=grafana/grafana:11.4.0` | `capturas/10_rolling_update.png` |
| Historial y rollback | Deployment | `kubectl rollout undo deployment/grafana-deployment` | `capturas/11_rollback.png` |
| Exposición | Service | `minikube service grafana-service --url` | `capturas/grafana_home.png` |

## ReplicaSet vs Deployment

| | ReplicaSet | Deployment |
|---|---|---|
| Mantiene N réplicas vivas | Sí | Sí, a través de su ReplicaSet |
| Escalado | Sí | Sí |
| Actualizar versión sin caída | No | Sí (RollingUpdate) |
| Historial y rollback | No | Sí |

El Deployment no gestiona pods directamente: crea un ReplicaSet nuevo por cada
versión y conserva los anteriores con 0 réplicas, por eso puede volver atrás.

## Limpiar

```bash
kubectl delete -f service/ -f deployment/ -f replicaset/ -f configmap/
kubectl delete pod redis-client myfirstpod
```
