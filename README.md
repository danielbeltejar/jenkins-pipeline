# Jenkins Pipeline para CI/CD con Docker y Helm

Esta pipeline de Jenkins automatiza el proceso de construcción, empaquetado y despliegue de aplicaciones utilizando Docker y Helm. Está diseñada para repositorios monorepo (con múltiples aplicaciones en un solo repo) y repositorios con subdirectorios (cada uno con su propia aplicación).

## Descripción General

La pipeline realiza los siguientes pasos principales:
- Checkout del código fuente desde un repositorio Git.
- Configuración del entorno (ej. hosts para registry).
- Construcción de imágenes Docker usando Kaniko.
- Empaquetado de charts de Helm.
- Subida de charts a un repositorio Git de Helm.
- Despliegue con Helm en Kubernetes.

Soporta dos modos de operación basados en el parámetro `BUILD_ROOT`:
- **BUILD_ROOT=true**: Construye desde la raíz del repositorio (monorepo). Usa `APP_NAME` para etiquetar la imagen y el chart.
- **BUILD_ROOT=false**: Construye desde el subdirectorio especificado en `APP_NAME`.

## Parámetros

- **GIT_URL** (string): URL del repositorio Git (ej. `https://github.com/user/repo`).
- **APP_NAME** (string): Nombre de la aplicación (ej. `front`, `back`). Para `BUILD_ROOT=false`, especifica el subdirectorio a construir.
- **BUILD_ROOT** (string, default: 'false'): 'true' para monorepo (raíz), 'false' para subdirectorios.

## Stages

### 1. Declarative: Checkout SCM
- Obtiene el Jenkinsfile desde el repositorio de pipelines.

### 2. Checkout Source Code
- Clona el repositorio de la aplicación usando `GIT_URL`.

### 3. Configure Environment
- Configura el entorno, como agregar entradas a `/etc/hosts` para el registry de Harbor.

### 4. Build Docker Image
- Construye la imagen Docker usando Kaniko.
- Para monorepo: contexto en raíz, Dockerfile en raíz.
- Para subdirs: contexto en subdir, Dockerfile en subdir.
- Push a Harbor registry.

### 5. Package Helm Chart
- Actualiza la versión en `Chart.yaml`.
- Empaqueta el chart de Helm.

### 6. Upload Helm Package to GitHub
- Sube el chart empaquetado a un repositorio Git de charts (ej. `helm-charts`).
- Organiza por `IMAGE_REPO` y `APP_NAME`.

### 7. Deploy with Helm
- Despliega el chart usando Helm upgrade/install.

## Requisitos

- **Jenkins con Kubernetes plugin**: Para ejecutar en pods con contenedores Kaniko, Helm y Git.
- **Credenciales**:
  - `GITHUB_AUTH_TOKEN`: Para acceso a GitHub.
  - `DISCORD_CREDENTIALS`: Para notificaciones (no usado en el código actual).
  - `HARBOR_CREDENTIALS`: Para push a Harbor.
- **ConfigMaps/Secrets**:
  - `docker-auth-config`: Para autenticación en Docker registry.
  - Certificados en `/nfs/lab-jenkins/certs/`.
- **Repositorio de aplicación**: Debe tener `Dockerfile` y `k8s/` (para Helm).
- **Repositorio de Helm charts**: `https://github.com/danielbeltejar/helm-charts` en rama `develop`.

## Uso

1. Configura el job en Jenkins como Pipeline desde SCM, apuntando a este repositorio.
2. Ejecuta el job con los parámetros deseados.
3. Para monorepo: `BUILD_ROOT=true`, `APP_NAME=front`.
4. Para subdirs: `BUILD_ROOT=false`, `APP_NAME=front` (construye solo el subdirectorio `front`).

## Mejores Prácticas Implementadas

- **Paralelización**: Builds múltiples en paralelo para eficiencia (si se implementa en futuras versiones).
- **Detección automática**: No aplica; usa `APP_NAME` explícitamente.
- **Seguridad**: Uso de credenciales y certificados.
- **Escalabilidad**: Soporte para monorepo y subdirs específicos.
- **Logging**: Salida detallada en consola.

## Troubleshooting

- **Error en cd**: Asegúrate de que los subdirs existan y tengan `Dockerfile` para `BUILD_ROOT=false`.
- **Push fallido**: Verifica credenciales de Harbor y conectividad.
- **Helm deploy**: Asegúrate de que el cluster Kubernetes esté accesible.

Para más detalles, revisa el `Jenkinsfile`.