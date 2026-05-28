# GitHub Actions — Guía Completa de Referencia
> Infraestructuras de Soporte · UMA · 2026

---

## Índice

1. [Conceptos fundamentales](#1-conceptos-fundamentales)
2. [Estructura base de un workflow](#2-estructura-base-de-un-workflow)
3. [Triggers — cuándo se ejecuta](#3-triggers--cuándo-se-ejecuta)
4. [Permissions — qué puede hacer el token](#4-permissions--qué-puede-hacer-el-token)
5. [Jobs — patrones y configuración](#5-jobs--patrones-y-configuración)
6. [Steps — tipos y sintaxis](#6-steps--tipos-y-sintaxis)
7. [Actions esenciales del Marketplace](#7-actions-esenciales-del-marketplace)
8. [Secrets y variables de entorno](#8-secrets-y-variables-de-entorno)
9. [Condicionales — if:](#9-condicionales--if)
10. [Artefactos — subir y descargar ficheros entre jobs](#10-artefactos--subir-y-descargar-ficheros-entre-jobs)
11. [Docker en el pipeline](#11-docker-en-el-pipeline)
12. [Kubernetes y self-hosted runner](#12-kubernetes-y-self-hosted-runner)
13. [Badges — estado del workflow y cobertura](#13-badges--estado-del-workflow-y-cobertura)
14. [Errores comunes y soluciones](#14-errores-comunes-y-soluciones)
15. [Pipelines completos de referencia](#15-pipelines-completos-de-referencia)

---

## 1. Conceptos fundamentales

GitHub Actions es la plataforma CI/CD nativa de GitHub. Permite automatizar compilación, tests, publicación de imágenes Docker y despliegues directamente desde el repositorio.

### Jerarquía de componentes

```
Repositorio
└── Workflow (fichero .yml en .github/workflows/)
    └── Job (se ejecuta en un Runner independiente)
        └── Step (unidad mínima de trabajo)
            ├── uses: → invoca una Action del Marketplace
            └── run:  → ejecuta un comando shell
```

**Reglas clave:**
- Un repositorio puede tener N workflows.
- Los jobs de un workflow se ejecutan **en paralelo por defecto** en Runners aislados (máquinas limpias).
- Para forzar secuencialidad entre jobs se usa `needs:`.
- Cada job arranca en una **máquina nueva y limpia**: no comparte ficheros ni estado con otros jobs.
- Para pasar ficheros entre jobs se usan **Artifacts** (`upload-artifact` / `download-artifact`).

### Tipos de Runners

| Tipo | Descripción | Cuándo usarlo |
|---|---|---|
| `ubuntu-latest` | Máquina virtual en la nube de GitHub | La gran mayoría de casos |
| `windows-latest` | VM Windows en la nube de GitHub | Builds de .NET, Windows-specific |
| `macos-latest` | VM macOS en la nube de GitHub | Builds de iOS/macOS |
| `self-hosted` | Tu propia máquina registrada | Despliegues en clústers locales (K8s), acceso a red privada |

---

## 2. Estructura base de un workflow

```yaml
name: Nombre visible en la pestaña Actions   # Opcional pero recomendado

on: [push, pull_request]                     # Triggers

permissions:                                  # Permisos del token GITHUB_TOKEN
  contents: write
  checks: write

env:                                          # Variables globales del workflow
  JAVA_VERSION: '21'

jobs:

  nombre-del-job:                             # Identificador único del job (snake_case)
    runs-on: ubuntu-latest                    # Runner
    needs: [otro-job]                         # Dependencias (opcional)
    if: github.event_name == 'push'           # Condición de ejecución (opcional)

    env:                                      # Variables locales al job
      MI_VAR: valor

    steps:

      - name: Descripción del step           # Nombre visible en la UI
        uses: owner/action@version            # Invocar una Action
        with:                                 # Parámetros de la Action
          param1: valor1
          param2: valor2

      - name: Comando shell
        run: echo "Hola mundo"               # Comando directo

      - name: Comandos múltiples
        run: |
          echo "Línea 1"
          echo "Línea 2"
        env:                                  # Variables locales al step
          MI_TOKEN: ${{ secrets.MI_TOKEN }}
```

---

## 3. Triggers — cuándo se ejecuta

### Disparadores más comunes

```yaml
# Cualquier push o PR en cualquier rama
on: [push, pull_request]

# Solo en push a main y PRs contra main
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

# Solo cuando cambian ciertos ficheros
on:
  push:
    paths:
      - 'src/**'
      - 'pom.xml'

# Programado (sintaxis cron)
on:
  schedule:
    - cron: '0 6 * * 1-5'   # Lunes a viernes a las 6:00 UTC

# Manual desde la UI o API
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Entorno de despliegue'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

# Al publicar una release
on:
  release:
    types: [published]

# Combinación habitual en CI/CD completo
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
```

### Contextos de evento útiles

```yaml
# Distinguir push de PR dentro del workflow
if: github.event_name == 'push'
if: github.event_name == 'pull_request'

# Solo en push a main (no en PRs)
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

---

## 4. Permissions — qué puede hacer el token

El token `GITHUB_TOKEN` se genera automáticamente en cada ejecución. Sus permisos se restringen con `permissions:`.

```yaml
permissions:
  contents: read          # Leer el código del repositorio
  contents: write         # Leer + escribir (necesario para git push del badge)
  checks: write           # Crear/actualizar checks (dorny/test-reporter)
  pull-requests: write    # Comentar en PRs
  packages: write         # Publicar en GitHub Packages
  actions: read           # Leer workflows
  statuses: write         # Actualizar estados de commits
```

### Combinaciones habituales por caso de uso

```yaml
# Solo tests y reportes (sin escribir en el repo)
permissions:
  contents: read
  checks: write

# Tests + badge de cobertura (necesita push al repo)
permissions:
  contents: write
  checks: write

# Pipeline completo con Docker Hub (no necesita write en el repo)
permissions:
  contents: read
  checks: write

# Pipeline con badge de Jacoco (necesita push del .svg)
permissions:
  contents: write
  checks: write
  pull-requests: write
```

---

## 5. Jobs — patrones y configuración

### Job de compilación (Build)

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: maven           # Cachea ~/.m2 entre ejecuciones del mismo repo

    - run: chmod +x ./mvnw     # Necesario: mvnw puede no tener permisos de ejecución

    - run: ./mvnw compile --no-transfer-progress
```

### Job de tests (con informes y badges)

```yaml
test:
  needs: build               # Espera a que build termine correctamente
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: maven

    - run: chmod +x ./mvnw

    - name: Ejecutar tests (unitarios + integración)
      run: ./mvnw verify --no-transfer-progress
      # verify = compile + test (Surefire) + integration-test (Failsafe)

    - name: Informe de tests en GitHub
      uses: dorny/test-reporter@v2
      if: always()             # Ejecutar aunque los tests fallen
      with:
        name: Resultados de Pruebas
        path: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
        reporter: java-junit
        fail-on-error: false

    - name: Subir XML como artefacto descargable
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-reports-xml
        path: |
          **/target/surefire-reports/*.xml
          **/target/failsafe-reports/*.xml
        retention-days: 7

    - name: Generar badge de cobertura Jacoco
      id: jacoco
      uses: cicirello/jacoco-badge-generator@v2
      with:
        generate-branches-badge: true
        jacoco-csv-file: target/site/jacoco/jacoco.csv

    - name: Commit y push del badge
      if: github.event_name == 'push'  # Solo en push, no en PRs
      run: |
        git config --global user.name "github-actions"
        git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
        git add .github/badges/
        git commit -m "Actualizar badge de cobertura Jacoco" || echo "Sin cambios"
        git push
```

### Job de Docker (build + push a Docker Hub)

```yaml
docker-build-and-push:
  needs: test
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Login a Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Configurar Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build y push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.DOCKERHUB_USERNAME }}/nombre-imagen:latest
          ${{ secrets.DOCKERHUB_USERNAME }}/nombre-imagen:${{ github.sha }}
        cache-from: type=gha    # Caché de capas Docker en GitHub Actions
        cache-to: type=gha,mode=max
```

### Job de despliegue en Kubernetes (self-hosted)

```yaml
deploy-to-kubernetes:
  needs: docker-build-and-push
  runs-on: self-hosted         # Corre en tu máquina local donde está kubectl
  steps:
    - uses: actions/checkout@v4

    - name: Crear namespace si no existe
      run: kubectl create namespace ips --dry-run=client -o yaml | kubectl apply -f -

    - name: Aplicar manifiestos
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml

    - name: Forzar actualización de imagen
      run: kubectl rollout restart deployment/springuma-deployment -n ips

    - name: Esperar a que el despliegue esté listo
      run: kubectl rollout status deployment/springuma-deployment -n ips --timeout=120s
```

### Job con múltiples entornos (matrix)

```yaml
test-matrix:
  runs-on: ubuntu-latest
  strategy:
    matrix:
      java: ['17', '21']       # Ejecuta el job para cada versión de Java
    fail-fast: false           # No cancela el resto si uno falla
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: ${{ matrix.java }}
        distribution: 'temurin'
    - run: ./mvnw test
```

---

## 6. Steps — tipos y sintaxis

### Los dos tipos de step

```yaml
# Tipo 1: invocar una Action del Marketplace
- name: Descripción
  uses: owner/action@version
  with:
    parametro: valor

# Tipo 2: ejecutar un comando shell
- name: Descripción
  run: echo "hola"

# Comando multilínea
- name: Varios comandos
  run: |
    chmod +x ./mvnw
    ./mvnw compile
    echo "Compilado"
```

### Versionado de Actions

```yaml
uses: actions/checkout@v4          # Tag de versión mayor (recomendado)
uses: actions/checkout@v4.1.7      # Tag exacto (máxima reproducibilidad)
uses: actions/checkout@abc1234     # Commit SHA (máxima seguridad)
# Nunca usar @latest en producción
```

### Capturar outputs de un step

```yaml
- name: Obtener versión
  id: get-version               # id para referenciar el output
  run: echo "version=1.0.0" >> $GITHUB_OUTPUT

- name: Usar el output
  run: echo "La versión es ${{ steps.get-version.outputs.version }}"
```

### Variables de entorno en steps

```yaml
- name: Step con variables
  run: echo "Hola $NOMBRE"
  env:
    NOMBRE: GitHub Actions
    TOKEN: ${{ secrets.MI_TOKEN }}
```

---

## 7. Actions esenciales del Marketplace

### actions/checkout — descargar el código

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0      # 0 = historial completo (necesario para algunos badges)
    ref: main           # Rama específica (por defecto la del trigger)
```

> **Importante:** Cada job corre en una máquina nueva. Si el job de test necesita el código, debe hacer checkout aunque el job de build ya lo haya hecho.

### actions/setup-java — configurar JDK

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'   # Eclipse Temurin (recomendado), también: 'corretto', 'zulu'
    cache: maven              # Cachea ~/.m2 → ahorra 1-2 min de descarga de dependencias
```

### actions/upload-artifact y download-artifact — pasar ficheros entre jobs

```yaml
# En el job que genera el fichero:
- uses: actions/upload-artifact@v4
  with:
    name: mi-artefacto        # Nombre del artefacto (único en el workflow)
    path: target/*.jar        # Qué subir
    retention-days: 7         # Cuántos días conservarlo (máx. 90)
    if-no-files-found: error  # error | warn | ignore

# En el job que lo consume:
- uses: actions/download-artifact@v4
  with:
    name: mi-artefacto        # Mismo nombre que al subir
    path: ./descargado/       # Dónde descargarlo
```

### dorny/test-reporter — informe de tests en la UI de GitHub

```yaml
- uses: dorny/test-reporter@v2      # Siempre v2, v1 falla con Node.js 24
  if: always()
  with:
    name: Resultados de Pruebas     # Nombre visible en la pestaña Checks
    path: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
    reporter: java-junit
    fail-on-error: false            # No falla el workflow si hay tests fallidos
```

> **Path correcto:** Usar `TEST-*.xml` en vez de `*.xml` para excluir `failsafe-summary.xml`, que no es formato JUnit y rompe el parser.

### cicirello/jacoco-badge-generator — badge de cobertura

```yaml
- uses: cicirello/jacoco-badge-generator@v2
  id: jacoco
  with:
    generate-branches-badge: true
    jacoco-csv-file: target/site/jacoco/jacoco.csv
    # El CSV lo genera el plugin jacoco-maven-plugin en el pom.xml
    # con <goal>report</goal>

# Acceder al porcentaje generado:
- run: echo "Cobertura: ${{ steps.jacoco.outputs.coverage }}"
```

**Requisito en `pom.xml`:** el plugin `jacoco-maven-plugin` debe estar configurado y ejecutarse en la fase `verify`.

### docker/login-action, setup-buildx-action, build-push-action

```yaml
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}   # Token con permisos Read & Write

- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v5
  with:
    context: .            # Directorio con el Dockerfile
    push: true
    tags: |
      usuario/imagen:latest
      usuario/imagen:${{ github.sha }}
```

---

## 8. Secrets y variables de entorno

### Jerarquía de variables

| Tipo | Dónde se define | Alcance |
|---|---|---|
| `secrets.*` | GitHub → Settings → Secrets | Cifrado, no visible en logs |
| `vars.*` | GitHub → Settings → Variables | Texto plano, visible |
| `env:` en el workflow | En el fichero YAML | Solo ese workflow |
| `$GITHUB_ENV` | Comando en un step | Steps posteriores del mismo job |
| `$GITHUB_OUTPUT` | Comando en un step | Steps posteriores (como output) |

### Configurar secrets en GitHub

```
Repositorio → Settings → Secrets and variables → Actions → New repository secret
```

Secrets imprescindibles para el pipeline completo:
- `DOCKERHUB_USERNAME` → tu usuario de Docker Hub
- `DOCKERHUB_TOKEN` → token con permisos **Read & Write** (nunca la contraseña)

### Usar secrets y variables en el YAML

```yaml
env:
  DB_URL: ${{ secrets.DATABASE_URL }}
  APP_ENV: ${{ vars.ENVIRONMENT }}      # Variable no secreta

steps:
  - run: echo "Usuario: ${{ secrets.DOCKERHUB_USERNAME }}"
  # GitHub enmascara automáticamente el valor en los logs → aparece ***
```

### Variables de contexto predefinidas

```yaml
${{ github.sha }}           # SHA del commit actual (ej: abc1234...)
${{ github.ref }}           # Ref del trigger (ej: refs/heads/main)
${{ github.event_name }}    # Nombre del evento (push, pull_request...)
${{ github.actor }}         # Usuario que disparó el workflow
${{ github.repository }}    # owner/repo
${{ github.run_number }}    # Número de ejecución del workflow
${{ runner.os }}            # Linux, Windows, macOS
```

---

## 9. Condicionales — if:

### A nivel de job

```yaml
jobs:
  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    # Solo despliega en push a main, nunca en PRs
```

### A nivel de step

```yaml
steps:
  - name: Solo en push
    if: github.event_name == 'push'
    run: echo "Es un push"

  - name: Solo si el step anterior falló
    if: failure()
    run: echo "Algo fue mal"

  - name: Siempre (incluso si hay fallos anteriores)
    if: always()
    run: echo "Siempre se ejecuta"

  - name: Solo si todo ha ido bien hasta aquí
    if: success()
    run: echo "Todo OK"
```

### Funciones de estado disponibles

| Función | Cuándo se ejecuta el step |
|---|---|
| `success()` | Solo si todos los steps anteriores pasaron (por defecto) |
| `failure()` | Solo si algún step anterior falló |
| `always()` | Siempre, independientemente del estado |
| `cancelled()` | Solo si el workflow fue cancelado |

### Operadores en condiciones

```yaml
if: github.event_name == 'push'                          # Igualdad
if: github.event_name != 'pull_request'                  # Desigualdad
if: github.ref == 'refs/heads/main' && success()         # AND
if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'  # OR
if: contains(github.ref, 'main')                         # Contiene
if: startsWith(github.ref, 'refs/tags/')                 # Empieza por
```

---

## 10. Artefactos — subir y descargar ficheros entre jobs

Los jobs corren en máquinas independientes. Para pasar el `.jar` compilado del job `build` al job `test`, o guardar reportes para descargar manualmente, se usan artefactos.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./mvnw package -DskipTests

      - name: Subir JAR como artefacto
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 1

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Descargar JAR
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: ./artifacts/

      - run: ls ./artifacts/
```

### Cuándo usar artefactos vs caché

| | Artefactos | Caché (`cache: maven`) |
|---|---|---|
| **Propósito** | Pasar ficheros entre jobs o guardar resultados | Acelerar builds reutilizando dependencias |
| **Cuándo expira** | `retention-days` (7-90 días) | Automáticamente (7 días de inactividad) |
| **Casos de uso** | .jar, reportes XML, badges | `~/.m2`, `node_modules` |

---

## 11. Docker en el pipeline

### Flujo completo: Dockerfile → Docker Hub

```yaml
docker-build-and-push:
  needs: test
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Login a Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Configurar Docker Buildx
      uses: docker/setup-buildx-action@v3
      # Buildx habilita builds multi-plataforma y caché avanzada

    - name: Build y push con múltiples tags
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.DOCKERHUB_USERNAME }}/mi-app:latest
          ${{ secrets.DOCKERHUB_USERNAME }}/mi-app:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### Dockerfile multi-stage (referencia)

```dockerfile
# Stage 1: compilación
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /src
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -q    # Cachea dependencias como capa Docker
COPY src/ src/
RUN ./mvnw package -DskipTests -q

# Stage 2: imagen final ligera
FROM eclipse-temurin:21-jre-alpine
WORKDIR /src
COPY --from=builder /src/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Por qué multi-stage:** la imagen final solo contiene el JRE + el .jar. Sin Maven, JDK ni código fuente → ~200 MB en vez de 600+ MB.

---

## 12. Kubernetes y self-hosted runner

### Instalar el runner como servicio

```bash
# 1. Descargar desde: GitHub → repo → Settings → Actions → Runners → New
mkdir actions-runner && cd actions-runner
curl -o runner.tar.gz -L https://github.com/actions/runner/releases/download/vX.X.X/actions-runner-linux-x64-X.X.X.tar.gz
tar xzf runner.tar.gz

# 2. Registrar contra el repositorio (GitHub te da el token)
./config.sh --url https://github.com/USUARIO/REPO --token TU_TOKEN

# 3. Instalar como servicio systemd (arranca automáticamente)
sudo ./svc.sh install
sudo ./svc.sh start

# Verificar estado
sudo ./svc.sh status
```

### Job de despliegue completo

```yaml
deploy-to-kubernetes:
  needs: docker-build-and-push
  runs-on: self-hosted         # Usa tu máquina donde está kubectl configurado
  steps:
    - uses: actions/checkout@v4

    - name: Crear namespace (idempotente)
      run: kubectl create namespace ips --dry-run=client -o yaml | kubectl apply -f -

    - name: Aplicar manifiestos
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml

    - name: Forzar descarga de la imagen nueva
      run: kubectl rollout restart deployment/springuma-deployment -n ips
      # Necesario porque :latest no indica a K8s que la imagen cambió

    - name: Esperar a que el despliegue esté listo
      run: kubectl rollout status deployment/springuma-deployment -n ips --timeout=120s
      # Falla el pipeline si la app crashea al arrancar
```

### Por qué el runner puede usar kubectl sin configuración extra

El runner corre como tu usuario local, que tiene `~/.kube/config` generado por Minikube/k3s/kind. kubectl lo lee automáticamente. No se necesita ninguna configuración adicional en el YAML del workflow.

---

## 13. Badges — estado del workflow y cobertura

### Badge de estado del workflow

Añadir en el `README.md`:

```markdown
![CI](https://github.com/USUARIO/REPO/actions/workflows/ci.yml/badge.svg)

<!-- Con enlace al workflow: -->
[![CI](https://github.com/USUARIO/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/USUARIO/REPO/actions/workflows/ci.yml)

<!-- Para una rama específica: -->
![CI](https://github.com/USUARIO/REPO/actions/workflows/ci.yml/badge.svg?branch=main)
```

Muestra automáticamente `passing` (verde) o `failing` (rojo) según el último run del workflow.

### Badge de cobertura con Jacoco

El step `cicirello/jacoco-badge-generator@v2` genera los ficheros `.svg` en `.github/badges/`. El step de commit y push los sube al repositorio. Luego se referencian en el README:

```markdown
![Cobertura](/.github/badges/jacoco.svg)
![Ramas](/.github/badges/branches.svg)
```

**Requisito en `pom.xml`:** el plugin jacoco debe generar el CSV:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

El CSV se genera en `target/site/jacoco/jacoco.csv` al ejecutar `./mvnw verify`.

---

## 14. Errores comunes y soluciones

| Error | Causa | Solución |
|---|---|---|
| Step partido en dos ítems de lista | `name:` y `uses:` en líneas de lista separadas | Poner ambos bajo el mismo `-` |
| `mvn` en vez de `./mvnw` | Maven del sistema puede no estar instalado | Siempre usar `./mvnw` |
| `chmod +x ./mvnw` olvidado | `mvnw` no tiene permisos de ejecución en Linux | Añadir el step de chmod antes de usarlo |
| `dorny/test-reporter@v1` falla | Incompatible con Node.js 24 del runner | Usar `dorny/test-reporter@v2` |
| `failsafe-summary.xml` rompe el parser | Path `*.xml` incluye ficheros que no son JUnit | Usar `TEST-*.xml` para solo los reportes |
| `401 Unauthorized` en Docker Hub | Token sin permisos de escritura | Crear token con permisos **Read & Write** |
| `${{ secrets.DOCKERHUB_TOKEN }}` en el tag | Error de copiar-pegar | El tag de imagen siempre usa `DOCKERHUB_USERNAME` |
| `on:` dentro de un job | `on:` es clave raíz del workflow | Eliminar el `on:` del job |
| `needs:` referencia un job inexistente | Nombre incorrecto o job no definido | Verificar que el nombre coincide exactamente |
| Badge de Jacoco no se actualiza | `contents: write` falta en permissions | Añadir `contents: write` al bloque permissions |
| Push del badge falla con 403 | Token sin permisos de escritura | Añadir `contents: write` en permissions |
| Job de deploy no arranca | Runner self-hosted no está activo | Ejecutar `sudo ./svc.sh start` o `./run.cmd` |
| `kubectl: command not found` | El runner no tiene kubectl en el PATH | Instalar kubectl en la máquina del runner |
| Jacoco CSV no encontrado | El plugin no se ejecutó o la ruta es incorrecta | Verificar que `./mvnw verify` ejecuta el goal `report` de jacoco |

---

## 15. Pipelines completos de referencia

### Pipeline básico — Solo CI (build + test + informes)

```yaml
name: CI Pipeline
on: [push, pull_request]

permissions:
  contents: write
  checks: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      - run: chmod +x ./mvnw
      - run: ./mvnw compile --no-transfer-progress

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      - run: chmod +x ./mvnw
      - run: ./mvnw verify --no-transfer-progress

      - uses: dorny/test-reporter@v2
        if: always()
        with:
          name: Resultados de Pruebas
          path: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
          reporter: java-junit
          fail-on-error: false

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-reports-xml
          path: |
            **/target/surefire-reports/*.xml
            **/target/failsafe-reports/*.xml
          retention-days: 7

      - uses: cicirello/jacoco-badge-generator@v2
        id: jacoco
        with:
          generate-branches-badge: true
          jacoco-csv-file: target/site/jacoco/jacoco.csv

      - name: Commit y push del badge
        if: github.event_name == 'push'
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add .github/badges/
          git commit -m "Actualizar badge de cobertura Jacoco" || echo "Sin cambios"
          git push
```

---

### Pipeline completo — CI/CD con Docker y Kubernetes

```yaml
name: CI/CD Pipeline
on: [push, pull_request]

permissions:
  contents: write
  checks: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      - run: chmod +x ./mvnw
      - run: ./mvnw compile --no-transfer-progress

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      - run: chmod +x ./mvnw
      - run: ./mvnw verify --no-transfer-progress

      - uses: dorny/test-reporter@v2
        if: always()
        with:
          name: Resultados de Pruebas
          path: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
          reporter: java-junit
          fail-on-error: false

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-reports-xml
          path: |
            **/target/surefire-reports/*.xml
            **/target/failsafe-reports/*.xml
          retention-days: 7

      - uses: cicirello/jacoco-badge-generator@v2
        id: jacoco
        with:
          generate-branches-badge: true
          jacoco-csv-file: target/site/jacoco/jacoco.csv

      - name: Commit y push del badge
        if: github.event_name == 'push'
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add .github/badges/
          git commit -m "Actualizar badge de cobertura Jacoco" || echo "Sin cambios"
          git push

  docker-build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/nombre-imagen:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/nombre-imagen:${{ github.sha }}

  deploy-to-kubernetes:
    needs: docker-build-and-push
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - run: kubectl create namespace ips --dry-run=client -o yaml | kubectl apply -f -

      - run: |
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml

      - run: kubectl rollout restart deployment/springuma-deployment -n ips

      - run: kubectl rollout status deployment/springuma-deployment -n ips --timeout=120s
```
