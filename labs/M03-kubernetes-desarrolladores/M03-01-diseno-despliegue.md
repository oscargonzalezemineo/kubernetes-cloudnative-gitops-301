# M03-01 — Diseño de despliegue en Kubernetes

[← Página anterior](README.md) · [Siguiente página →](M03-02-estrategias-despliegue.md)

> Práctica del módulo. Lee el [README](README.md) (teoría).

## Objetivo

Desplegar la **API Flask** y el **frontend nginx estático** en kind con Deployments, Services, ConfigMap y Secret.

## Prerrequisitos

- M02 completado (`./scripts/lab-verify.sh m02-01` y `m02-02` OK).
- Clúster kind operativo (`./scripts/kind-up.sh`).

## Antes de empezar

```bash
./scripts/lab-prepare.sh m03-01
./scripts/lab-up.sh    # valida imágenes locales
```

Crea la carpeta de trabajo vacía:

```bash
mkdir -p infra/k8s/base
```

Consulta la solución de referencia solo al final: `infra/k8s/solutions/m03-01/`.

## Mapa del ejercicio

```text
Paso 1   Namespace cloudnative-lab
Paso 2   ConfigMap + Secret (config Flask)
Paso 3   Deployment + Service demo-api (probes /health y /ready)
Paso 4   Deployment + Service demo-web (NodePort)
Paso 5   Redis en el clúster
Paso 6   Cargar imágenes y aplicar
```

---

### 1 — Namespace

**Acción:** Crea `infra/k8s/base/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudnative-lab
```

**Por qué:** Aísla recursos del curso y simplifica `kubectl -n cloudnative-lab`.

---

### 2 — ConfigMap y Secret para Flask

**Acción:** Crea `api-configmap.yaml` y `api-secret.yaml`.

ConfigMap (no secretos):

```yaml
data:
  REDIS_URL: redis://redis:6379/0
  SERVICE_NAME: cloudnative-demo-api
  PORT: "8081"
```

Secret (`stringData`):

```yaml
DATABASE_URL: postgres://lab:lab@postgres:5432/lab
```

**Por qué:** Misma separación que en M02 — Flask lee `${DATABASE_URL}` del entorno; en K8s lo inyectas con `envFrom`.

---

### 3 — Deployment API Flask

**Acción:** `api-deployment.yaml` con:

| Campo | Valor | Qué comprueba |
|-------|-------|---------------|
| `image` | `cloudnative-demo-api:local` | Imagen local en kind |
| `imagePullPolicy` | `IfNotPresent` | No pull si ya está cargada |
| `replicas` | `2` | Dos Pods |
| `livenessProbe` | `GET /health:8081` | ¿Proceso vivo? (no mira Postgres) |
| `readinessProbe` | `GET /ready:8081` | ¿Listo para tráfico? (Postgres + Redis) |
| `envFrom` | configMapRef + secretRef | Variables de entorno |

**Por qué importa para el rollout:** Kubernetes **solo marca el Pod Ready** si `readinessProbe` recibe **200**. Un rolling update **no termina** mientras los Pods nuevos no pasen esa sonda. Si `/ready` devuelve 503, verás `rollout status` atascado (lo practicas en M03-02).

Crea `api-service.yaml` (ClusterIP, puerto 8081).

---

### 4 — Frontend nginx estático

**Acción:** `web-deployment.yaml` + `web-service.yaml` (NodePort 30080 → 8080).

La imagen `cloudnative-demo-web:local` sirve el build nginx estático con nginx y proxy `/api/` hacia la API.

---

### 5 — Redis

**Acción:** `redis.yaml` — Deployment + Service puerto 6379.

---

### 6 — Desplegar en kind

```bash
docker compose -f infra/docker-compose.yml build demo-api demo-web
kind load docker-image cloudnative-demo-api:local --name cloudnative-lab
kind load docker-image cloudnative-demo-web:local --name cloudnative-lab
kubectl apply -f infra/k8s/base/namespace.yaml
kubectl apply -f infra/k8s/base/ -n cloudnative-lab
kubectl -n cloudnative-lab get pods -w
```

**Resultado esperado:** Pods `demo-api` en columna STATUS `Running`. La columna **READY** puede ser `0/1` hasta que despliegues Postgres (siguiente bloque).

> [!WARNING]
> **Obligatorio antes de M03-02**
>
> El readinessProbe del Deployment llama a `GET /ready`. Ese endpoint de Flask (M02-01) comprueba **Postgres y Redis**. En este lab solo despliegas Redis; el Secret ya apunta a `postgres:5432`, pero el servicio Postgres **aún no existe**.
>
> Consecuencia: verás `Running` y **`0/1 Ready`** → en M03-02 el comando `kubectl rollout status` **se quedará esperando indefinidamente**.
>
> **Antes de pasar a M03-02**, despliega Postgres:
>
> ```bash
> kubectl apply -f infra/k8s/solutions/m03-03/postgres-statefulset.yaml -n cloudnative-lab
> kubectl -n cloudnative-lab wait --for=condition=ready pod -l app=postgres --timeout=120s
> kubectl -n cloudnative-lab rollout restart deployment/demo-api
> kubectl -n cloudnative-lab get pods -l app=demo-api   # debe mostrar 2/2 READY en el Deployment
> ```

---

### 7 — Verificar

```bash
./scripts/lab-verify.sh m03-01
kubectl -n cloudnative-lab port-forward svc/demo-api 8081:8081 &
curl -s http://127.0.0.1:8081/health | jq .
curl -s http://127.0.0.1:8081/work | jq .
```

## Comprueba tu entendimiento

**¿Por qué ConfigMap y Secret separados?**

→ Rotación y permisos distintos; los secretos no van en texto plano en Git en producción.

**¿Por qué se queda atascado `kubectl rollout status`?**

→ Porque la **readinessProbe** (`GET /ready`) no recibe HTTP 200. Sin Postgres, `/ready` devuelve 503, el Pod no pasa a Ready y el rolling update no avanza.

## Errores frecuentes

| Síntoma | Causa | Arreglo |
|---------|-------|---------|
| `ImagePullBackOff` | Imagen no cargada en kind | `kind load docker-image ...` |
| CrashLoop `KeyError` JDBC | Falta Secret | Revisa `envFrom` y secret |
| Readiness 503 / `0/1 Ready` | Postgres no desplegado; `/ready` exige DB | Aplica `infra/k8s/solutions/m03-03/postgres-statefulset.yaml` antes de M03-02 |
| `rollout status` colgado | Pods nunca Ready (misma causa) | Postgres + `rollout restart`; ver M03-02 |

→ **[M03-02 — Estrategias de despliegue](M03-02-estrategias-despliegue.md)**
