# Pipeline CI/CD con BuildKit y Helm

Pipeline de Jenkins para construir imágenes Docker (BuildKit), empaquetar charts Helm y desplegar en Kubernetes con verificación automática de rollout y rollback.

## Parámetros

| Parámetro | Descripción |
|-----------|-------------|
| `GIT_URL` | URL del repositorio Git de la aplicación |
| `APP_NAME` | Nombre de la aplicación (ej. `front`, `back`). Con `BUILD_ROOT=false` indica el subdirectorio |
| `BUILD_ROOT` | `true`: construye desde la raíz (monorepo). `false` (defecto): construye desde el subdirectorio `APP_NAME` |

## Etapas

| # | Etapa | Contenedor | Descripción |
|---|-------|------------|-------------|
| 1 | **Checkout Source Code** | `git` | Clona el repositorio de la aplicación desde `GIT_URL` |
| 2 | **Configure Environment** | `buildkit` | Verifica la resolución DNS del registry Harbor |
| 3 | **Build Docker Image** | `buildkit` | Construye y sube la imagen con BuildKit. Usa caché de registry. Se omite si `APP_NAME=apigw` |
| 4 | **Security Scan** | `trivy` | Escanea misconfigurations en Dockerfile/Helm (IaC) y vulnerabilidades en la imagen Docker. Bloquea en CRITICAL. Se omite scan de imagen si `APP_NAME=apigw` |
| 5 | **Package Helm Chart** | `helm` | Actualiza `appVersion` en `Chart.yaml` y empaqueta el chart |
| 6 | **Upload Helm Package** | `git` | Sube el chart al repo [helm-charts](https://github.com/danielbeltejar/helm-charts) (rama `develop`). Usa lock para evitar conflictos |
| 7 | **Deploy with Helm** | `helm` | Ejecuta `helm upgrade --install` del chart empaquetado |
| 8 | **Verify Deployment** | `helm` | Verifica el rollout de Deployments, StatefulSets y DaemonSets. Timeout: 5 min. Si falla: recopila diagnósticos y ejecuta `helm rollback` automático |

## Contenedores del Pod

| Contenedor | Imagen | Uso |
|------------|--------|-----|
| `buildkit` | `moby/buildkit:v0.27.0-rootless` | Construcción de imágenes Docker |
| `helm` | `alpine/k8s:1.32.3` | Helm + kubectl (empaquetado, despliegue, verificación) |
| `git` | `alpine/git` | Operaciones Git |
| `trivy` | `aquasec/trivy:0.69.3` | Escaneo de vulnerabilidades y misconfigurations |

## Requisitos

- Jenkins con **Kubernetes plugin** y pod template `buildkit`
- **Credenciales**: `GITHUB_AUTH_TOKEN`, `HARBOR_CREDENTIALS`, `DISCORD_CREDENTIALS`
- **ConfigMap** `docker-auth-config` con autenticación para el registry
- Certificados CA en `/nfs/lab-jenkins/certs/`
- Repositorio de la aplicación con `Dockerfile` y directorio `k8s/` (chart Helm)

## Uso

```
# Monorepo (Dockerfile en raíz)
BUILD_ROOT=true  APP_NAME=front  GIT_URL=https://github.com/user/repo

# Subdirectorio (Dockerfile en APP_NAME/)
BUILD_ROOT=false APP_NAME=back   GIT_URL=https://github.com/user/repo
```

## Resolución de problemas

| Problema | Solución |
|----------|----------|
| Error en `cd` | Verificar que el subdirectorio existe y contiene `Dockerfile` |
| Push al registry falla | Comprobar credenciales de Harbor y conectividad de red |
| Verificación de rollout falla | Revisar los diagnósticos en el log (eventos de pods, estado). El rollback se ejecuta automáticamente |
| Rollback falla | Comprobar RBAC del service account del pod Jenkins |
| Trivy: descarga de DB falla | Verificar conectividad a `ghcr.io` desde el pod. Considerar mirror interno de Trivy DB |
| Trivy: scan de imagen falla con auth error | Comprobar que `docker-auth-config` ConfigMap tiene credenciales para Harbor |
| Trivy: falsos positivos en IaC | Añadir excepciones en `.trivyignore` en el directorio del proyecto |