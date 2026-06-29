# Metodología 12-Factor App

Referencia de apoyo para el curso **Kubernetes + Docker Avanzado — CloudNative, GitOps y Observabilidad**.

La metodología [**12-Factor App**](https://12factor.net/es/) (Heroku, 2011) describe cómo construir aplicaciones **SaaS** que se despliegan en la nube de forma repeatable, portable y escalable. No es un estándar oficial, pero sigue siendo la base conceptual del diseño **cloudnative** con contenedores.

## Por qué importa en este curso

En M02 aprendes a diseñar la **capa contenedor** para que la misma imagen funcione en distintos entornos (Compose local, Kubernetes, ECS, Azure Container Apps, etc.). Los 12 factores explican **qué** debe cumplir tu app para no quedar acoplada a un solo hosting.

```text
Código versionado  →  Build  →  Imagen portable  →  Config por entorno  →  Cualquier plataforma
```

## Los doce factores (resumen)

| # | Factor | Idea en una frase | En este curso |
|---|--------|-------------------|---------------|
| I | **Codebase** | Un repo por app; ramas desplegables, no forks sueltos | Repo del curso = una codebase demo |
| II | **Dependencies** | Dependencias explícitas, no asumir del SO | `requirements.txt`, capas Docker |
| III | **Config** | Toda config en el entorno | **M02-01** — variables de entorno |
| IV | **Backing services** | DB, cache, colas como recursos adjuntables | Postgres/Redis como servicios externos |
| V | **Build, release, run** | Build y run estrictamente separados | **M02-02** imagen; **M05** CI/CD |
| VI | **Processes** | Procesos stateless; estado en backing services | API sin sesión local; hits en Redis |
| VII | **Port binding** | App se expone exportando un puerto | `PORT` / 8081 en la demo |
| VIII | **Concurrency** | Escalar horizontalmente vía procesos | **M03+** — réplicas en Kubernetes |
| IX | **Disposability** | Arranque rápido, parada limpia | **M02-01** — `/health`, `/ready` |
| X | **Dev/prod parity** | Entornos lo más parecidos posible | Codespaces + kind ≈ cloud reducido |
| XI | **Logs** | Logs como flujo de eventos (stdout) | **M08** — agregación Prometheus/ELK |
| XII | **Admin processes** | Tareas admin como procesos one-off | Migraciones, jobs (mención; no lab dedicado) |

## Factores que trabajamos en profundidad

### III — Config

> *«Guarda la config en el entorno, no en el código.»*

- URLs, credenciales y flags **no** van en el repositorio ni en la imagen.
- La app lee **variables de entorno**; la plataforma las inyecta.
- Lab: [M02-01](../labs/M02-docker-avanzado-cloudnative/M02-01-adaptacion-cloudnative.md).

### V — Build, release, run

> *«Build convierte código en artefacto; release combina artefacto + config; run ejecuta.»*

| Fase | Qué es | Ejemplo en el curso |
|------|--------|---------------------|
| **Build** | Compilar / empaquetar imagen | `docker build`, pipeline CI |
| **Release** | Artefacto + config inmutable | Tag de imagen + variables del entorno |
| **Run** | Procesos en ejecución | Contenedor en Compose, Pod en K8s |

Lab: [M02-02](../labs/M02-docker-avanzado-cloudnative/M02-02-optimizacion-imagenes.md), [M05](../labs/M05-cicd/README.md).

### IX — Disposability

> *«Los procesos son desechables: arrancan rápido y mueren limpio.»*

- **Liveness** (`/health`): ¿el proceso responde? → si no, reinicio.
- **Readiness** (`/ready`): ¿puede recibir tráfico? → si no, sacarlo del balanceo.

Orquestadores (Kubernetes, ECS, etc.) usan estos contratos. Lab: [M02-01](../labs/M02-docker-avanzado-cloudnative/M02-01-adaptacion-cloudnative.md).

### IV — Backing services

> *«Trata DB y cache como recursos conectables por URL, intercambiables sin cambio de código.»*

La demo trata Postgres y Redis como **servicios externos**: solo cambia la URL en config, no la lógica de negocio. Labs: M02 (Compose) → M03 (Services en Kubernetes).

### XI — Logs

> *«Un proceso no gestiona archivos de log; escribe a stdout/stderr.»*

La plataforma agrega y rota logs. En M08 verás cómo recogerlos (ELK, etc.).

## Relación con contenedores y orquestadores

| Concepto 12-factor | Capa contenedor | Plataformas de despliegue |
|--------------------|-----------------|---------------------------|
| Config (III) | Env vars en el proceso | Compose, K8s, ECS task def, Azure app settings |
| Disposability (IX) | HTTP probes | Liveness/readiness probes |
| Port binding (VII) | `EXPOSE` + `$PORT` | Service, target group, ingress |
| Backing services (IV) | Red entre contenedores | DNS interno, service mesh |

**Principio clave:** diseña la imagen cumpliendo los factores; el orquestador concreto es intercambiable.

## Anti-patrones frecuentes

| Anti-patrón | Factor que rompe | Qué hacer |
|-------------|------------------|-----------|
| Password en `api.py` | III — Config | Variables de entorno + secret store |
| Misma imagen distinta por entorno | V — Build/release/run | Una imagen, config distinta por release |
| Guardar sesión en memoria del Pod | VI — Processes | Estado en Redis/DB |
| Logs solo dentro del contenedor | XI — Logs | Escribir a stdout; agregador externo |
| Sin `/ready` | IX — Disposability | Probe de dependencias antes de tráfico |

## Referencias

- Sitio oficial (inglés): [12factor.net](https://12factor.net/)
- Traducción al español: [12factor.net/es](https://12factor.net/es)
- Módulo del curso donde se aplica: [M02 — Docker avanzado y diseño cloudnative](../labs/M02-docker-avanzado-cloudnative/README.md)

---

[← Volver al índice del curso](../README.md)
