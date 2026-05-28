# Práctica 6 — CI/CD con GitHub Actions y Docker
> Infraestructuras de Soporte · UMA · 2026 · Recap para el examen

---

## 1. Estructura de un Workflow de GitHub Actions

Todo workflow vive en `.github/workflows/nombre.yml`. GitHub lo detecta automáticamente y lo ejecuta según el disparador configurado en `on:`.

```yaml
name: Nombre del pipeline

on: [push, pull_request]      # cuándo se dispara

permissions:
  contents: read
  checks: write

jobs:
  nombre-job:
    runs-on: ubuntu-latest
    needs: otro-job           # dependencia entre jobs
    steps:
      - name: Descripción
        uses: action@version  # acción externa del Marketplace
      - name: Descripción
        run: comando          # comando de shell
```

**Conceptos clave:**

- `on:` controla el disparador. `push` para commits, `pull_request` para PRs.
- `needs:` encadena jobs — si un job falla, todos los que dependen de él no se ejecutan.
- `runs-on:` define la máquina donde corre el job. Siempre `ubuntu-latest` salvo casos especiales.
- `uses:` invoca una Action del GitHub Marketplace (código reutilizable ya hecho).
- `run:` ejecuta comandos de shell directamente en la máquina del runner.
- Los jobs corren en **máquinas nuevas y limpias** cada vez — por eso hay que repetir `checkout` y `setup-java` en cada job.
- `secrets.*` son variables cifradas en GitHub Settings. Nunca se escriben en claro en el YAML.

**Diferencia entre los comandos de Maven:**

| Comando | Qué hace |
|---|---|
| `./mvnw compile` | Solo compila el código fuente |
| `./mvnw test` | Compila + tests unitarios (Surefire) |
| `./mvnw verify` | Todo lo anterior + tests de integración (Failsafe, clases `*IT`) |

---

## 2. Los Jobs Típicos del Pipeline

### Job 1 — Build (compilación)

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-java@v4
      with:
        distribution: temurin
        java-version: '21'
        cache: maven          # cachea ~/.m2 entre ejecuciones

    - run: chmod +x ./mvnw
    - run: ./mvnw compile --no-transfer-progress
```

### Job 2 — Test (unitarios + integración + informe)

```yaml
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

    - name: Test report en Github
      uses: dorny/test-reporter@v2      # ← v2, no v1
      if: always()
      with:
        name: Resultados de Pruebas
        path: '**/target/surefire-reports/*.xml,**/target/failsafe-reports/*IT.xml'
        reporter: java-junit

    - name: Subir artefactos XML
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-reports-xml
        path: |
          **/target/surefire-reports/*.xml
          **/target/failsafe-reports/*.xml
        retention-days: 7
```

- `if: always()` hace que el informe se genere aunque los tests fallen — fundamental para ver qué ha fallado.
- El path `*IT.xml` excluye `failsafe-summary.xml`, que no es un reporte JUnit y rompe el parser.

---

## 3. Docker en el Pipeline

### Job 3 — Build y push a Docker Hub

```yaml
docker-build-and-push:
  needs: test
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}    # token con permisos Read & Write

    - uses: docker/setup-buildx-action@v3

    - uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.DOCKERHUB_USERNAME }}/nombre-imagen:latest
          ${{ secrets.DOCKERHUB_USERNAME }}/nombre-imagen:${{ github.sha }}
```

**Por qué dos tags:**
- `:latest` siempre apunta a la versión más reciente. Kubernetes la usará para descargar la imagen.
- `:${{ github.sha }}` identifica la imagen con el commit exacto que la generó. Permite rollback a cualquier versión anterior.

**Configurar los secrets en GitHub:**  
GitHub → repositorio → Settings → Secrets and variables → Actions → New repository secret

- `DOCKERHUB_USERNAME` = tu usuario de Docker Hub
- `DOCKERHUB_TOKEN` = token de hub.docker.com con permisos **Read & Write**

---

## 4. El Dockerfile — Multi-Stage Build

```dockerfile
# Etapa 1: compilación (imagen pesada con JDK y Maven)
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /src

# Copiar primero los ficheros de Maven (optimización de caché)
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -q    # descarga deps sin compilar

# Copiar el código y compilar
COPY src/ src/
RUN ./mvnw package -DskipTests -q

# Etapa 2: imagen final (solo JRE, sin Maven ni código fuente)
FROM eclipse-temurin:21-jre-alpine

WORKDIR /src
COPY --from=builder /src/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Por qué multi-stage build:**
- La imagen final solo contiene el **JRE** y el **.jar**. No incluye Maven, JDK ni código fuente.
- Resultado: imagen mucho más pequeña (~200 MB vs 600+ MB) y más segura (menos superficie de ataque).

**Optimización de caché de Docker:**  
Docker construye por capas. Si una capa no cambia, reutiliza la caché. El orden importa:
- Se copia `pom.xml` y `.mvn/` primero → se descargan las dependencias → capa cacheada.
- Solo después se copia `src/` con el código fuente.
- Si solo cambia el código (no el pom.xml), Docker reutiliza la capa de dependencias y no las vuelve a descargar.

---

## 5. Manifiestos de Kubernetes

### deployment.yaml — define cómo corre la aplicación

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springuma-deployment
  namespace: ips
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springuma         # debe coincidir con el label del pod
  template:
    metadata:
      labels:
        app: springuma
    spec:
      containers:
        - name: springuma
          image: usuario/imagen:latest
          imagePullPolicy: Always    # siempre descarga, no usa caché local
          ports:
            - containerPort: 8080
```

### service.yaml — expone la aplicación al exterior

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springuma-service
  namespace: ips
spec:
  type: NodePort
  selector:
    app: springuma           # debe coincidir con el label del Deployment
  ports:
    - port: 8080             # puerto dentro del clúster
      targetPort: 8080       # puerto del contenedor
      nodePort: 30080        # puerto accesible desde fuera del clúster
```

**Puntos críticos:**
- `selector` y `labels` deben coincidir entre Deployment y Service. Si no coinciden, el Service no conecta con ningún pod y las peticiones no llegan.
- `imagePullPolicy: Always` es necesario con tag `:latest` para forzar que Kubernetes descargue la nueva imagen en cada despliegue.

### Comandos kubectl en el pipeline

```bash
# Crear namespace si no existe (idempotente)
kubectl create namespace ips --dry-run=client -o yaml | kubectl apply -f -

# Aplicar manifiestos (declarativo: crea o actualiza)
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Forzar descarga de la nueva imagen (:latest no indica cambio a K8s)
kubectl rollout restart deployment/springuma-deployment -n ips

# Esperar confirmación de que el despliegue fue exitoso
kubectl rollout status deployment/springuma-deployment -n ips --timeout=120s
```

- `kubectl apply` es declarativo: calcula el diff y solo cambia lo necesario.
- `rollout restart` es necesario porque con el tag `:latest` Kubernetes no detecta que la imagen cambió.
- `rollout status` bloquea el pipeline hasta confirmar que todos los pods están `Ready`. Sin este paso el pipeline marcaría verde aunque la app haya crasheado.

---

## 6. Errores Comunes

| Error | Causa | Solución |
|---|---|---|
| Step partido en dos ítems de lista | `name:` y `uses:` en líneas de lista separadas | Poner ambos bajo el mismo guión (`-`) |
| `mvn` en vez de `./mvnw` | Usa Maven del sistema, puede no estar instalado | Siempre usar `./mvnw` |
| `dorny/test-reporter` TypeError | Versión v1 incompatible con Node.js 24 del runner | Usar `dorny/test-reporter@v2` |
| `failsafe-summary.xml` no es JUnit | Path `*.xml` captura todos los XML de failsafe | Usar `*IT.xml` para solo los tests |
| 401 Unauthorized en Docker Hub | Token sin permisos de escritura | Crear token con permisos Read & Write |
| `DOCKERHUB_TOKEN` en el tag de imagen | Error de copiar-pegar | El tag siempre usa `DOCKERHUB_USERNAME` |
| `on:` dentro de un job | `on:` es clave raíz del workflow, no de un job | Eliminarlo del job |
| `needs:` referencia un job que no existe | Nombre incorrecto o job no definido | Verificar que el nombre coincide exactamente |

---

## 7. Pipeline Completo de Referencia

```yaml
name: CI/CD Pipeline
on: [push, pull_request]

permissions:
  contents: read
  checks: write

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
          path: '**/target/surefire-reports/*.xml,**/target/failsafe-reports/*IT.xml'
          reporter: java-junit
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-reports-xml
          path: |
            **/target/surefire-reports/*.xml
            **/target/failsafe-reports/*.xml
          retention-days: 7

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
