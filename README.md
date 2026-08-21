# ci-workflows

Workflow de CI/CD **reutilizable** para los proyectos del homelab (cuenta personal de GitHub).

## Qué aporta

Un único workflow `workflow_call` (`.github/workflows/ci-pipeline.yml`) que cubre la **cadena común** de seguridad de un proyecto:

| Job | Qué hace |
|-----|----------|
| `trivy-fs` | Escaneo de dependencias (gate CRITICAL/HIGH) |
| `semgrep` | Análisis estático (SAST + SARIF a Code Scanning) |
| `docker-build` | Build + push a DockerHub (latest + sha) + Docker Scout |
| `trivy-image` | Escaneo de la imagen publicada |
| `preview` | Preview efímero en el runner self-hosted del homelab (healthcheck → k6 → E2E → ZAP → destruye) |

El job `test` **queda en cada repo** (cada stack tiene el suyo: Python/alembic, Node, Go...).

## Cómo conectar un proyecto nuevo

### 1. Instalar el runner (una vez por repo)

En el LXC 300 (finpowerr-preview):

```bash
./install-runner.sh <repo>   # registra una instancia del runner con label homelab-preview
```

Un runner de repo solo sirve a su repo; el label compartido no roba jobs.

### 2. Crear `ci.yml` en el repo

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  # ── Test del repo (SIEMPRE local, no reutilizable) ──
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Test
        run: ./test.sh   # tu test real

  # ── Pipeline común (reutilizable) ──
  pipeline:
    uses: mtnezdilarduya/ci-workflows/.github/workflows/ci-pipeline.yml@v1
    with:
      image_name: mtnezdilarduya/tu-proyecto
      context: .
      preview_port_range: 3100     # rango exclusivo por repo (3100-3199, 3200-3299...)
      healthcheck_script: scripts/healthcheck.sh   # opcional
      e2e_script: scripts/e2e.js                   # opcional
      zap_enabled: true
      k6_enabled: false
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

El job `pipeline` depende implícitamente de tu `test` vía `needs` (si tu test falla, el pipeline no corre).

### 3. Configurar secrets en el repo

En Settings → Secrets → Actions: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`. (Los secrets viven en el repo que llama, NO aquí.)

## Inputs

| Input | Default | Descripción |
|-------|---------|-------------|
| `image_name` | — (requerido) | Imagen Docker completa |
| `context` | `.` | Contexto del Dockerfile |
| `dockerfile` | `Dockerfile` | Ruta al Dockerfile |
| `runner_label` | `homelab-preview` | Label del runner self-hosted |
| `preview_port_range` | `3100` | Base del rango de puertos del preview (por repo) |
| `preview_internal_port` | `3000` | Puerto interno del contenedor |
| `healthcheck_script` | `""` | Script de healthcheck específico del repo |
| `e2e_script` | `""` | Script E2E (Puppeteer) del repo |
| `zap_enabled` | `true` | Activa ZAP baseline (DAST) |
| `k6_enabled` | `true` | Activa k6 (carga/rendimiento) |
| `trivy_severity` | `CRITICAL,HIGH` | Severidad mínima del gate |
| `zap_fail_threshold` | `2` | Exit de ZAP que falla (2=High) |

## Concurrencia y seguridad

- **Puerto/red/nombres únicos por run**: `${repo}-e2e-net-${sha}`, `${repo}-preview-${sha}`, puerto `base + sha%100`. Dos proyectos distintos **no colisionan**.
- **Lock (`flock`)**: si dos previews se lanzan a la vez, el segundo espera (máx. 15 min). Cola sin infraestructura.
- **Prune de disco**: `docker image prune` + `system prune` al arrancar el preview (el LXC 300 se llena con imágenes sha-* y ZAP).
- **Guard de forks**: el preview solo corre en `main` — un PR de fork jamás toca el homelab.
- **`cancel-in-progress`**: un push nuevo a la misma rama cancela el run anterior.

## Versionado

- `@main` = canary (usar solo durante desarrollo).
- `@v1` = estable (congelar tras CI verde). `@v2` para breaking changes.
- Los consumidores pueden pin por SHA para congelar.

## GVM/OpenVAS (scan periódico, fuera de CI)

El escaneo OpenVAS **no vive en GitHub Actions** — es un escáner de red pesado que corre por **cron en el LXC 301** contra el despliegue. Ver `docs/gvm.md` y `scripts/openvas_scan.sh` (parametrizado por `PROJECT_NAME`).
