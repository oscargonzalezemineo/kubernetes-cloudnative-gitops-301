# M01-02 — Verificar aplicación demo

[← Página anterior](M01-01-bootstrap-entorno.md) · [Siguiente página →](../M02-docker-avanzado-cloudnative/README.md)

> Práctica del módulo. La teoría y la demo están en el [README del módulo](README.md).

### Objetivo

Levantar la aplicación demo en Docker Compose y localizar la configuración embebida que adaptarás en M02.

### Prerrequisitos

- M01-01 completado (Codespace + kind operativos).

### En qué consiste

Despliegas la stack demo (web, API, Postgres, Redis, generador de carga), validas endpoints y revisas el código fuente buscando parámetros que no deberían estar fijos.

### 1 — Levantar la stack demo

**Acción:** Desde la raíz del repo ejecuta:

```bash
./scripts/lab-up.sh
```

**Por qué:** Hasta M03 trabajarás la app en Compose; después la migrarás a Kubernetes con la misma base de código.

**Resultado esperado:** `health-check.sh` muestra `OK: demo-web :8080` y `OK: demo-api :8081`.

### 2 — Probar la API

**Acción:** Ejecuta:

```bash
curl -s http://127.0.0.1:8081/health | jq .
curl -s http://127.0.0.1:8081/work | jq .
```

**Por qué:** Confirma que la API responde y genera tráfico útil para labs posteriores (métricas, logs).

**Resultado esperado:** JSON con `"status":"ok"` y respuestas `/work` con campo `hits` creciente.

### 3 — Abrir la web

**Acción:** En la pestaña **Ports** del Codespace, abre el puerto **8080** (demo-web).

**Por qué:** Verificas el frontend estático que acompañará a la API en los despliegues K8s.

**Resultado esperado:** Página «Kubernetes + Docker Avanzado — demo web».

### 4 — Explorar el código

**Acción:** Abre `infra/app/api/api.py` y localiza constantes como `DATABASE_URL`, `REDIS_URL` y `API_PORT`.

**Por qué:** En M02-01 externalizarás estos valores; hoy están acoplados al entorno de lab.

**Resultado esperado:** Identificas al menos tres valores que deberían venir de variables de entorno o ConfigMap/Secret.

### 5 — Revisar contenedores

**Acción:** Ejecuta `docker compose -f infra/docker-compose.yml ps`.

**Por qué:** Conoces los servicios que más adelante modelarás como Deployments y Services en Kubernetes.

**Resultado esperado:** Cinco servicios `running`: demo-web, demo-api, postgres, redis, loadgen.

## Comprueba tu entendimiento

**Servicios activos**
`docker compose -f infra/docker-compose.yml ps --format json | jq -r '.Service' | sort`
→ Cinco servicios listados.

**Configuración embebida**
Abre `infra/app/api/api.py` y señala dónde están las credenciales de Postgres.
→ Constante `DATABASE_URL` con usuario y contraseña en texto plano.

## Reto

### 1 — Lista de cambios cloudnative

Anota tres cambios mínimos para hacer la app «cloudnative-ready» antes de M02-01 (pista: [12-Factor](../../docs/12-factor-app.md), health checks, secretos).

<details>
<summary>Ver orientación</summary>

Ejemplos válidos: externalizar `DATABASE_URL`/`REDIS_URL`, no commitear credenciales, parametrizar puerto con `PORT`, añadir probes basados en `/health`, imagen sin root.

</details>

## Errores frecuentes

| Síntoma | Causa probable | Cómo arreglarlo |
|---------|----------------|-----------------|
| `demo-api` no responde | Postgres aún no healthy | Espera 30 s y repite `health-check.sh` |
| Puerto 8080 no visible | Forwarding del Codespace | Pestaña **Ports** → visibilidad **Public** |
| Build Docker lento | Primera descarga de capas base | Normal en el primer `lab-up`; las siguientes usan caché |
