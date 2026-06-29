# M02-01 — Adaptación de aplicación a modelo cloudnative

[← Página anterior](README.md) · [Siguiente página →](M02-02-optimizacion-imagenes.md)

> Práctica del módulo. Lee primero el [README del módulo](README.md) (teoría, contexto y demo).

## Objetivo

Externalizar la configuración de la API demo para que **una sola imagen** funcione en distintos entornos cambiando solo variables en runtime.

## Prerrequisitos

- M01 completado (`./scripts/lab-up.sh` responde OK).
- Haber leído en el README: *Configuración vs código* y *Health vs Ready*.

## En qué consiste

Recorrerás el camino de una app «acoplada al lab» a una app alineada con la [metodología 12-Factor](../../docs/12-factor-app.md): quitar secretos del código, inyectar config vía **variables de entorno** (en el lab usarás un fichero local como simulación) y exponer `/ready` para que cualquier orquestador sepa cuándo enviar tráfico.

## Mapa del ejercicio

```text
Paso 1–2   Revisar y externalizar config en api.py
Paso 3     Añadir /ready (readiness)
Paso 4     Conectar variables de entorno (fichero local del lab)
Paso 5–6   Validar y simular otro entorno
```

| Al terminar… | Deberías poder explicar… |
|--------------|---------------------------|
| Paso 2 | Por qué `DATABASE_URL` no lleva default en código |
| Paso 3 | Diferencia entre `/health` y `/ready` |
| Paso 6 | Por qué cambiar variables de entorno no exige cambiar el código |

---

### 1 — Revisar el acoplamiento actual

**Acción:** Abre `infra/app/api/api.py`. Al terminar M01 el código tenía un bloque similar a:

```python
DATABASE_URL = "postgres://lab:lab@postgres:5432/lab"
REDIS_URL = "redis://redis:6379/0"
API_PORT = 8081
```

**Por qué:** Antes de refactorizar hay que **nombrar el problema**. Cada línea fija un host, un password o un puerto dentro del artefacto que subirías a un registry.

**En profundidad:** Si mañana la base de datos pasa de un host de desarrollo a uno de producción, con config embebida necesitas editar código, commit, build y push. Con config externa solo cambia lo que la **plataforma de despliegue** inyecta — da igual que sea Compose local, Kubernetes, ECS o Azure Container Apps.

**Resultado esperado:** Tienes una lista mental (o en el editor) de al menos tres valores que deben salir del código: URL de DB, URL de Redis y puerto.

---

### 2 — Externalizar con `os.environ`

**Acción:** Sustituye las constantes por lectura de entorno. El patrón objetivo:

```python
import os

DATABASE_URL = os.environ["DATABASE_URL"]
REDIS_URL = os.environ.get("REDIS_URL", "redis://redis:6379/0")
API_PORT = int(os.environ.get("PORT", "8081"))
SERVICE_NAME = os.environ.get("SERVICE_NAME", "cloudnative-demo-api")
LAB_SLOW_SECONDS = float(os.environ.get("LAB_SLOW_SECONDS", "3"))
```

**Por qué:**

- `os.environ["DATABASE_URL"]` **sin default** → si falta la variable, la app falla al arrancar (*fail fast*). Mejor que conectar a una DB equivocada en silencio.
- `.get(..., default)` en Redis o `PORT` → valores con fallback razonable solo donde el lab lo permite.

**En profundidad:** Usa **los mismos nombres de variable** en todos los entornos (`DATABASE_URL`, `PORT`, …). Así la imagen no cambia; solo cambia el mecanismo que las rellena (task definition en ECS, variables de app en Azure, manifiestos en Kubernetes, etc.).

> [!TIP]
> Convención habitual en cloud: `PORT` (no `API_PORT`) porque muchas plataformas inyectan `PORT` automáticamente (Cloud Run, Heroku-style, varios PaaS).

**Resultado esperado:** Un `grep -n postgres:// infra/app/api/api.py` no muestra credenciales en código (solo comentarios, si acaso).

---

### 3 — Añadir `/ready`

**Acción:** Implementa `GET /ready` que:

1. Ejecuta `SELECT 1` contra Postgres.
2. Hace `PING` a Redis.
3. Devuelve **200** si ambos OK, **503** si alguno falla.

Mantén `/health` simple (solo confirma que el proceso Flask responde).

**Por qué:** Simulas el contrato que exigen los orquestadores cloud:

- **Liveness** → `/health` — «¿ reinicio el contenedor? »
- **Readiness** → `/ready` — «¿ envío tráfico a esta réplica? »

Kubernetes, ECS con health checks y otros balanceadores managed usan esta distinción.

**En profundidad:** Si `/work` se llamara mientras Postgres está caído, verías errores 500 en cascada. `/ready` en 503 evita ese tráfico **antes** de que llegue a la lógica de negocio.

**Resultado esperado:**

```bash
curl -s http://127.0.0.1:8081/health | jq .
# {"service":"...","status":"ok"}

curl -s http://127.0.0.1:8081/ready | jq .
# {"service":"...","status":"ready"}
```

---

### 4 — Inyectar config en runtime (lab local)

**Acción:** En este curso, Docker Compose simula lo que en producción haría la plataforma:

```bash
cp infra/.env.example infra/.env
cat infra/.env
```

Verifica en `infra/docker-compose.yml` que `demo-api` recibe esas variables (p. ej. vía `env_file`).

**Por qué:** El contenedor **no debe leer ficheros de config propios del orquestador** en el código — solo el entorno del proceso. Compose traduce un fichero local a variables; Kubernetes, ECS o Azure harían lo equivalente con sus APIs.

> [!WARNING]
> **`infra/.env` es solo para el laboratorio local** (gitignored). En producción los secretos los gestiona el secret store de tu cloud, no un fichero en el repo.

**Resultado esperado:** Variables `DATABASE_URL`, `REDIS_URL`, `PORT`, `SERVICE_NAME` definidas en `.env` y visibles dentro del contenedor:

```bash
docker compose -f infra/docker-compose.yml exec demo-api printenv SERVICE_NAME
```

---

### 5 — Reconstruir y validar

**Acción:**

```bash
./scripts/lab-down.sh
./scripts/lab-up.sh
curl -s http://127.0.0.1:8081/health | jq .
curl -s http://127.0.0.1:8081/ready | jq .
curl -s http://127.0.0.1:8081/work | jq .
```

**Por qué:** Confirmas el circuito completo: config externa → contenedor → endpoints → dependencias reales (Postgres/Redis en `/work`).

**Resultado esperado:** Los tres `curl` devuelven JSON válido; `/work` incluye `"hits"` creciente si repites la petición.

---

### 6 — Probar otro entorno sin tocar código

**Acción:** Edita `infra/.env`:

```bash
SERVICE_NAME=cloudnative-demo-api-dev
```

Recrea solo la API:

```bash
docker compose -f infra/docker-compose.yml up -d --force-recreate demo-api
curl -s http://127.0.0.1:8081/health | jq .service
```

**Por qué:** Demuestra la promesa cloudnative: **misma imagen, distinta config inyectada**. En AWS cambiarías la task definition; en Azure las app settings; en Kubernetes los manifiestos — sin tocar el código.

**En profundidad:** Observa que **no** recompilaste lógica Python distinta — solo reinjectaste entorno. Eso es lo que escalará a decenas de réplicas en K8s.

**Resultado esperado:** Respuesta `"cloudnative-demo-api-dev"`. Restaura `SERVICE_NAME=cloudnative-demo-api` al cerrar el lab.

---

## Recapitulación

| Concepto | Dónde lo aplicaste |
|----------|-------------------|
| Config externa | Variables de entorno (`os.environ`) |
| Fail fast | `DATABASE_URL` obligatorio |
| Liveness | `/health` |
| Readiness | `/ready` con Postgres + Redis |
| Imagen portable | Cambio de config sin editar código fuente |

## Comprueba tu entendimiento

**Sin secretos en código**

Busca `postgres://` en `api.py`.

→ No debe aparecer en constantes; la conexión se define en variables de entorno.

**Readiness bajo fallo**

Para Postgres: `docker compose -f infra/docker-compose.yml stop postgres`, luego `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8081/ready`.

→ Código **503**. Vuelve a levantar Postgres y confirma **200**.

## Reto

### 1 — Fallo de Redis

Detén Redis, consulta `/ready` y anota el HTTP. Arranca Redis de nuevo.

<details>
<summary>Ver orientación</summary>

```bash
docker compose -f infra/docker-compose.yml stop redis
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8081/ready   # 503
docker compose -f infra/docker-compose.yml start redis
# espera unos segundos
curl -s http://127.0.0.1:8081/ready | jq .status   # "ready"
```

</details>

## Errores frecuentes

| Síntoma | Causa probable | Cómo arreglarlo |
|---------|----------------|-----------------|
| `KeyError: DATABASE_URL` | Falta `.env` o `env_file` | `cp infra/.env.example infra/.env` y `./scripts/lab-up.sh` |
| `/ready` siempre 503 | Postgres/Redis parados o arrancando | `docker compose -f infra/docker-compose.yml ps` |
| Cambio en `.env` no aplica | Contenedor no recreado | `docker compose ... up -d --force-recreate demo-api` |
| `/health` OK pero `/work` falla | Readiness no usada por Compose (normal aquí) | En despliegues orquestados el balanceador respetará `/ready` |

→ Siguiente: **[M02-02 — Optimización de imágenes](M02-02-optimizacion-imagenes.md)**
