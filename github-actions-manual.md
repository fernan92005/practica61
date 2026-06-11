# GitHub Actions — Manual de Referencia Rápida
> Infraestructuras de Soporte · UMA · 2026

---

## Índice

1. [Estructura base de un workflow](#1-estructura-base)
2. [Triggers](#2-triggers)
3. [Permissions](#3-permissions)
4. [Jobs y dependencias](#4-jobs-y-dependencias)
5. [Steps: uses vs run](#5-steps)
6. [BUILD](#6-build)
   - [Java + Maven](#java--maven)
   - [Python](#python)
7. [TEST](#7-test)
   - [Java — unitarios + integración + informe](#java--unitarios--integración--informe)
   - [Python](#python-1)
8. [DOCKER](#8-docker)
   - [Login + Build + Push a Docker Hub](#login--build--push-a-docker-hub)
   - [Dockerfile multi-stage (referencia)](#dockerfile-multi-stage)
9. [KUBERNETES (self-hosted)](#9-kubernetes-self-hosted)
10. [Artefactos](#10-artefactos)
11. [Secrets y variables de contexto](#11-secrets-y-variables-de-contexto)
12. [Condicionales](#12-condicionales)
13. [BADGES](#13-badges)
    - [Badge de estado del workflow](#badge-de-estado-del-workflow)
    - [Badge de cobertura Jacoco](#badge-de-cobertura-jacoco)
14. [Actions del Marketplace — resumen](#14-actions-del-marketplace)
15. [Pipelines completos de referencia](#15-pipelines-completos-de-referencia)
16. [Errores comunes](#16-errores-comunes)

---

## 1. Estructura base

```
.github/workflows/nombre.yml   ← GitHub lo detecta automáticamente
```

```yaml
name: Nombre del pipeline          # Visible en la pestaña Actions

on: [push, pull_request]           # Cuándo se dispara

permissions:
  contents: read
  checks: write

env:                               # Variables globales (opcionales)
  JAVA_VERSION: '21'

jobs:
  nombre-job:
    runs-on: ubuntu-latest
    needs: [otro-job]              # Dependencia (opcional)
    if: github.event_name == 'push'

    steps:
      - name: Descripción
        uses: owner/action@version
        with:
          param: valor

      - name: Comando shell
        run: echo "hola"

      - name: Múltiples comandos
        run: |
          echo "línea 1"
          echo "línea 2"
```

**Reglas clave:**
- Los jobs corren en **máquinas nuevas e independientes** — hay que repetir `checkout` y `setup-java` en cada job.
- Jobs en paralelo por defecto. `needs:` fuerza secuencialidad.
- Para pasar ficheros entre jobs → **artefactos** (`upload-artifact` / `download-artifact`).

---

## 2. Triggers

```yaml
# Push o PR en cualquier rama
on: [push, pull_request]

# Solo en main
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

# Programado (cron)
on:
  schedule:
    - cron: '0 6 * * 1-5'    # Lunes-viernes a las 6:00 UTC

# Manual desde la UI
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Entorno'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

# Al publicar una release
on:
  release:
    types: [published]

# Combinación habitual CI/CD completo
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
```

---

## 3. Permissions

```yaml
permissions:
  contents: read          # Leer el repo
  contents: write         # Leer + escribir (necesario para push de badges)
  checks: write           # Crear checks (dorny/test-reporter)
  pull-requests: write    # Comentar en PRs
  packages: write         # Publicar en GitHub Packages
```

**Combinaciones habituales:**

```yaml
# Solo tests y reportes
permissions:
  contents: read
  checks: write

# Tests + badge de cobertura (necesita push al repo)
permissions:
  contents: write
  checks: write
  pull-requests: write

# Pipeline con Docker Hub (no necesita write en el repo)
permissions:
  contents: read
  checks: write
```

---

## 4. Jobs y dependencias

```yaml
jobs:
  compile:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    needs: compile          # Espera a compile
    runs-on: ubuntu-latest
    steps: [...]

  build:
    needs: test             # Espera a test
    runs-on: ubuntu-latest
    steps: [...]

  integration-test:
    needs: build            # Espera a build
    runs-on: ubuntu-latest
    steps: [...]
```

**Con múltiples dependencias:**
```yaml
deploy:
  needs: [build, test]     # Espera a ambos
```

**Solo en push (no en PRs):**
```yaml
docker-push:
  needs: test
  if: github.event_name == 'push'
```

---

## 5. Steps

```yaml
# Tipo 1: Action del Marketplace
- name: Descripción
  uses: owner/action@v4
  with:
    parametro: valor

# Tipo 2: Comando shell
- name: Descripción
  run: echo "hola"

# Comando multilínea
- name: Varios comandos
  run: |
    chmod +x ./mvnw
    ./mvnw compile

# Variables de entorno locales al step
- name: Con variables
  run: echo "$MI_TOKEN"
  env:
    MI_TOKEN: ${{ secrets.TOKEN }}

# Capturar output de un step
- name: Obtener versión
  id: get-version
  run: echo "version=1.0.0" >> $GITHUB_OUTPUT

- name: Usar el output
  run: echo "${{ steps.get-version.outputs.version }}"
```

---

## 6. BUILD

### Java + Maven

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - name: Descargar el código
      uses: actions/checkout@v4

    - name: Configurar JDK 21
      uses: actions/setup-java@v4
      with:
        distribution: temurin      # También: 'corretto', 'zulu'
        java-version: '21'
        cache: maven               # Cachea ~/.m2 entre ejecuciones

    - name: Dar permisos al wrapper
      run: chmod +x ./mvnw

    - name: Compilar
      run: ./mvnw compile --no-transfer-progress

    # Opcional: generar JAR
    - name: Empaquetar
      run: ./mvnw -B clean package --file pom.xml

    # Opcional: subir JAR como artefacto
    - name: Subir JAR como artefacto
      uses: actions/upload-artifact@v4
      with:
        name: app-jar
        path: target/*.jar
```

**Diferencias entre comandos Maven:**

| Comando | Qué hace |
|---|---|
| `./mvnw compile` | Solo compila el código fuente |
| `./mvnw test` | Compila + tests unitarios (Surefire) |
| `./mvnw package` | Compile + test + genera el JAR |
| `./mvnw verify` | Todo lo anterior + tests de integración (Failsafe, clases `*IT`) |

### Python

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - name: Descargar el código
      uses: actions/checkout@v4

    - name: Configurar Python 3.12
      uses: actions/setup-python@v5
      with:
        python-version: '3.12'
        cache: 'pip'               # Cachea ~/.cache/pip

    - name: Instalar dependencias
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt

    - name: Lint con flake8
      run: |
        pip install flake8
        flake8 src/ --max-line-length=120
```

---

## 7. TEST

### Java — unitarios + integración + informe

```yaml
test:
  needs: build
  runs-on: ubuntu-latest
  steps:
    - name: Descargar el código
      uses: actions/checkout@v4

    - name: Configurar JDK 21
      uses: actions/setup-java@v4
      with:
        distribution: temurin
        java-version: '21'
        cache: maven

    - name: Dar permisos al wrapper
      run: chmod +x ./mvnw

    - name: Ejecutar tests (unitarios + integración)
      run: ./mvnw verify --no-transfer-progress

    # Informe visual en la pestaña Checks de GitHub
    - name: Publicar informe de tests en GitHub
      uses: dorny/test-reporter@v2      # ← SIEMPRE v2, v1 falla con Node.js 24
      if: always()                      # Ejecutar aunque los tests fallen
      with:
        name: Resultados de Pruebas
        path: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
        reporter: java-junit
        fail-on-error: false

    # Guardar XML como artefacto descargable
    - name: Subir XMLs de tests como artefacto
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-reports-xml
        path: |
          **/target/surefire-reports/*.xml
          **/target/failsafe-reports/*.xml
        retention-days: 7
```

> **Path crítico:** usar `TEST-*.xml` en vez de `*.xml` para excluir `failsafe-summary.xml`, que no es formato JUnit y rompe el parser.

### Python

```yaml
test:
  needs: build
  runs-on: ubuntu-latest
  steps:
    - name: Descargar el código
      uses: actions/checkout@v4

    - name: Configurar Python 3.12
      uses: actions/setup-python@v5
      with:
        python-version: '3.12'
        cache: 'pip'

    - name: Instalar dependencias
      run: pip install -r requirements.txt

    - name: Tests con pytest
      run: pytest tests/ --junitxml=test-results.xml

    - name: Subir resultados como artefacto
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results
        path: test-results.xml
        retention-days: 7
```

---

## 8. DOCKER

### Login + Build + Push a Docker Hub (básico)

```yaml
docker-build-and-push:
  needs: test
  runs-on: ubuntu-latest
  if: github.event_name == 'push'    # Solo en push, no en PRs
  steps:
    - name: Descargar el código
      uses: actions/checkout@v4

    - name: Login a Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}    # Token Read & Write, nunca la contraseña

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
        cache-from: type=gha         # Caché de capas Docker en GitHub Actions
        cache-to: type=gha,mode=max
```

**Por qué dos tags:**
- `:latest` → Kubernetes siempre descarga la versión más reciente.
- `:${{ github.sha }}` → trazabilidad exacta al commit. Permite rollback.

**Configurar secrets:**
```
GitHub → repo → Settings → Secrets and variables → Actions → New repository secret
  DOCKERHUB_USERNAME  = tu usuario de Docker Hub
  DOCKERHUB_TOKEN     = token de hub.docker.com con permisos Read & Write
```

### Login + Build + Push con QEMU + metadatos (multi-plataforma)

Versión completa con soporte para **múltiples arquitecturas** (amd64 + arm64) y **tags automáticos** generados a partir de los metadatos del commit.

```yaml
build-and-push:
  needs: test
  runs-on: ubuntu-latest
  steps:
    - name: Descargar el código
      uses: actions/checkout@v4

    - name: Configurar Docker QEMU
      uses: docker/setup-qemu-action@v3
      # QEMU emula otras arquitecturas (arm64, etc.) en la máquina x86 del runner
      # Sin esto solo puedes compilar para linux/amd64

    - name: Configurar Docker Buildx
      uses: docker/setup-buildx-action@v3
      # Buildx es necesario para builds multi-plataforma

    - name: Login en Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Extraer metadatos
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ secrets.DOCKERHUB_USERNAME }}/nombre-imagen
        tags: |
          type=raw,value=latest          # Tag fijo :latest
          type=sha                       # Tag con el SHA corto del commit (ej: sha-abc1234)
      # El step genera ${{ steps.meta.outputs.tags }} y ${{ steps.meta.outputs.labels }}

    - name: Build and Push
      uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        platforms: linux/amd64,linux/arm64   # Build para ambas arquitecturas
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
```

**Diferencias respecto al básico:**

| | Básico | Con QEMU + metadata-action |
|---|---|---|
| Arquitecturas | Solo `linux/amd64` | `linux/amd64` + `linux/arm64` (y más) |
| Tags | Manuales en el YAML | Generados automáticamente por `metadata-action` |
| Trazabilidad | `github.sha` completo | SHA corto (`sha-abc1234`) |
| Compatibilidad | Servidores x86 | Servidores x86 + Macs Apple Silicon + Raspberry Pi |

**Tipos de tag disponibles en `metadata-action`:**

```yaml
tags: |
  type=raw,value=latest          # Tag fijo con valor literal
  type=sha                       # SHA corto del commit
  type=ref,event=branch          # Nombre de la rama (ej: main)
  type=ref,event=pr              # Número de PR (ej: pr-42)
  type=semver,pattern={{version}}# Versión de la release (ej: 1.2.3)
  type=semver,pattern={{major}}  # Solo major (ej: 1)
```

### Dockerfile multi-stage

```dockerfile
# Stage 1: compilación (imagen pesada con JDK + Maven)
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /src
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -q    # Descarga deps → capa cacheada
COPY src/ src/
RUN ./mvnw package -DskipTests -q

# Stage 2: imagen final (solo JRE, sin Maven ni código fuente)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /src
COPY --from=builder /src/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Resultado:** ~200 MB en vez de 600+ MB. Sin JDK, Maven ni fuentes → más seguro.

**Optimización de caché Docker:** copiar `pom.xml` antes que `src/` → si solo cambia el código, Docker reutiliza la capa de dependencias.

---

## 9. KUBERNETES (self-hosted)

```yaml
deploy-to-kubernetes:
  needs: docker-build-and-push
  runs-on: self-hosted       # Tu máquina donde está kubectl configurado
  steps:
    - name: Descargar el código (para acceder a los manifiestos k8s/)
      uses: actions/checkout@v4

    - name: Crear namespace si no existe (idempotente)
      run: kubectl create namespace ips --dry-run=client -o yaml | kubectl apply -f -

    - name: Aplicar manifiestos
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml

    - name: Forzar descarga de la imagen nueva
      run: kubectl rollout restart deployment/springuma-deployment -n ips
      # Necesario: con tag :latest K8s no detecta que la imagen cambió

    - name: Esperar confirmación del despliegue
      run: kubectl rollout status deployment/springuma-deployment -n ips --timeout=120s
      # Falla el pipeline si la app crashea al arrancar
```

**Instalar el runner como servicio:**
```bash
# Descargar desde: GitHub → repo → Settings → Actions → Runners → New
mkdir actions-runner && cd actions-runner
./config.sh --url https://github.com/USUARIO/REPO --token TU_TOKEN
sudo ./svc.sh install
sudo ./svc.sh start
```

---

## 10. Artefactos

```yaml
# Subir (en el job que genera el fichero)
- name: Subir artefacto
  uses: actions/upload-artifact@v4
  with:
    name: mi-artefacto        # Nombre único en el workflow
    path: target/*.jar
    retention-days: 7         # Máx. 90. Por defecto 90.
    if-no-files-found: error  # error | warn | ignore

# Descargar (en otro job que lo necesita)
- name: Descargar artefacto
  uses: actions/download-artifact@v4
  with:
    name: mi-artefacto
    path: ./descargado/
```

| | Artefactos | Caché (`cache: maven`) |
|---|---|---|
| **Propósito** | Pasar ficheros entre jobs / guardar resultados | Acelerar builds reutilizando dependencias |
| **Expira** | `retention-days` | 7 días sin uso |
| **Casos de uso** | .jar, XML de tests, badges | `~/.m2`, `node_modules` |

---

## 11. Secrets y variables de contexto

### Tipos de variables

| Tipo | Dónde se define | Alcance |
|---|---|---|
| `secrets.*` | GitHub → Settings → Secrets | Cifrado, enmascarado en logs |
| `vars.*` | GitHub → Settings → Variables | Texto plano, visible |
| `env:` en YAML | En el fichero | Solo ese workflow/job/step |
| `$GITHUB_OUTPUT` | `echo "k=v" >> $GITHUB_OUTPUT` | Steps posteriores del mismo job |

### Variables de contexto predefinidas

```yaml
${{ github.sha }}           # SHA del commit (ej: abc1234...)
${{ github.ref }}           # Ref del trigger (ej: refs/heads/main)
${{ github.event_name }}    # Nombre del evento (push, pull_request...)
${{ github.actor }}         # Usuario que disparó el workflow
${{ github.repository }}    # owner/repo
${{ github.run_number }}    # Número de ejecución
${{ runner.os }}            # Linux, Windows, macOS
```

---

## 12. Condicionales

### A nivel de job

```yaml
deploy:
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

### A nivel de step

```yaml
- if: always()             # Siempre (incluso si algo falló antes)
- if: success()            # Solo si todo ha ido bien (valor por defecto)
- if: failure()            # Solo si algo falló
- if: github.event_name == 'push'
- if: contains(github.ref, 'main')
- if: startsWith(github.ref, 'refs/tags/')
```

---

## 13. BADGES

### Badge de estado del workflow

Formato de la URL:
```
https://github.com/USUARIO/REPO/actions/workflows/FICHERO.yml/badge.svg
```

En el `README.md`:

```markdown
<!-- Badge simple -->
![CI](https://github.com/USUARIO/REPO/actions/workflows/ci.yml/badge.svg)

<!-- Badge con enlace al workflow -->
[![CI](https://github.com/USUARIO/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/USUARIO/REPO/actions/workflows/ci.yml)

<!-- Para una rama específica -->
![CI](https://github.com/USUARIO/REPO/actions/workflows/ci.yml/badge.svg?branch=main)
```

**Un badge por workflow** → si tienes `compile.yml`, `test.yml`, `build.yml` separados, cada uno tiene su propio badge:

```markdown
![Compile](https://github.com/fernan92005/CICDTaskManager/actions/workflows/compile.yml/badge.svg?branch=main)
![Test](https://github.com/fernan92005/CICDTaskManager/actions/workflows/test.yml/badge.svg?branch=main)
![Build](https://github.com/fernan92005/CICDTaskManager/actions/workflows/build.yml/badge.svg?branch=main)
![Integration Test](https://github.com/fernan92005/CICDTaskManager/actions/workflows/integration-test.yml/badge.svg?branch=main)
![Javadoc](https://github.com/fernan92005/CICDTaskManager/actions/workflows/javadoc.yml/badge.svg?branch=main)
```

Muestra automáticamente `passing` (verde) o `failing` (rojo) según el último run.

### Badge de cobertura Jacoco

**Paso 1 — Añadir el plugin en `pom.xml`:**

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

Genera `target/site/jacoco/jacoco.csv` al ejecutar `./mvnw verify`.

**Paso 2 — Steps en el workflow (dentro del job de test):**

```yaml
- name: Generar badge de cobertura Jacoco
  id: jacoco
  uses: cicirello/jacoco-badge-generator@v2
  with:
    generate-branches-badge: true
    jacoco-csv-file: target/site/jacoco/jacoco.csv

# Acceder al porcentaje si se necesita
- run: echo "Cobertura ${{ steps.jacoco.outputs.coverage }}%"

- name: Commit y push del badge
  if: github.event_name == 'push'    # Solo en push, no en PRs
  run: |
    git config --global user.name "github-actions"
    git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
    git add .github/badges/
    git commit -m "Actualizar badge de cobertura Jacoco" || echo "Sin cambios"
    git push
```

**Paso 3 — En el `README.md`:**

```markdown
![Cobertura](/.github/badges/jacoco.svg)
![Ramas](/.github/badges/branches.svg)
```

> **Requisito:** `permissions: contents: write` — sin esto el push falla con 403.

---

## 14. Actions del Marketplace

| Action | Uso |
|---|---|
| `actions/checkout@v4` | Descargar el código del repo |
| `actions/setup-java@v4` | Configurar JDK (distribution + java-version + cache) |
| `actions/setup-python@v5` | Configurar Python (python-version + cache) |
| `actions/upload-artifact@v4` | Subir ficheros como artefacto |
| `actions/download-artifact@v4` | Descargar artefacto en otro job |
| `dorny/test-reporter@v2` | Informe visual de tests JUnit en GitHub Checks |
| `cicirello/jacoco-badge-generator@v2` | Genera SVG de cobertura desde jacoco.csv |
| `docker/login-action@v3` | Login en Docker Hub |
| `docker/setup-buildx-action@v3` | Habilita builds multi-plataforma |
| `docker/build-push-action@v5` | Build y push de imagen Docker |

---

## 15. Pipelines completos de referencia

### CI mínimo — Java (un solo job)

```yaml
name: CI (Single Job)
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
      - run: mvn compile --batch-mode
      - run: mvn test --batch-mode
      - run: mvn package --batch-mode
      - run: mvn integration-test --batch-mode
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/
      - uses: actions/upload-artifact@v4
        if: success()
        with:
          name: app-jar
          path: target/*.jar
```

### CI multi-job — Java (compile → test → build → integration-test)

```yaml
name: CI (Multi Job)
on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  compile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
      - run: mvn compile --batch-mode

  test:
    needs: compile
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
      - run: mvn test --batch-mode
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
      - run: mvn -B clean package
      - uses: actions/upload-artifact@v4
        if: success()
        with:
          name: app-jar
          path: target/*.jar

  integration-test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
      - run: mvn integration-test --batch-mode
```

### CI/CD completo — Java + Docker Hub + Kubernetes

```yaml
name: CI/CD Pipeline
on: [push, pull_request]

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: 'true'

permissions:
  contents: write
  checks: write
  pull-requests: write

jobs:

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
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
          distribution: temurin
          java-version: '21'
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
          git commit -m "Actualizar badge de cobertura" || echo "Sin cambios"
          git push

  docker-build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
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
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - run: kubectl create namespace ips --dry-run=client -o yaml | kubectl apply -f -
      - run: |
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml
      - run: kubectl rollout restart deployment/springuma-deployment -n ips
      - run: kubectl rollout status deployment/springuma-deployment -n ips --timeout=120s
```

---

## 16. Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `src refspec main does not match` | No hay ningún commit todavía | `git add . && git commit -m "init" && git push` |
| `mvn: command not found` | Maven del sistema no instalado | Usar siempre `./mvnw` |
| `chmod` olvidado | `mvnw` sin permisos en Linux | Añadir `run: chmod +x ./mvnw` antes de usarlo |
| `dorny/test-reporter@v1` TypeError | Incompatible con Node.js 24 del runner | Cambiar a `dorny/test-reporter@v2` |
| `failsafe-summary.xml` rompe el parser | Path `*.xml` incluye ficheros no-JUnit | Usar `TEST-*.xml` y `*IT.xml` |
| `401 Unauthorized` en Docker Hub | Token sin permisos de escritura | Crear token con permisos **Read & Write** |
| `DOCKERHUB_TOKEN` en el tag de imagen | Copy-paste incorrecto | El tag usa `DOCKERHUB_USERNAME`, no el token |
| `on:` dentro de un job | `on:` es clave raíz del workflow | Sacarlo al nivel raíz |
| `needs:` referencia job inexistente | Nombre incorrecto | Verificar que el nombre coincide exactamente |
| Badge de Jacoco no se actualiza (403) | Falta `contents: write` | Añadir `contents: write` en `permissions:` |
| Runner self-hosted no arranca jobs | Servicio parado | `sudo ./svc.sh start` |
| `kubectl: command not found` en runner | kubectl no instalado en la máquina | Instalar kubectl en la máquina del runner |
| Jacoco CSV no encontrado | Plugin no configurado o no se ejecutó `verify` | Verificar que `jacoco-maven-plugin` tiene el goal `report` en phase `verify` |
| Pod en `ImagePullBackOff` bloqueando rollout | Imagen no existe en el registry | `kubectl rollout undo deployment/nombre` |
