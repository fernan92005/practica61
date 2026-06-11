# CI/CD, GitHub Actions, Pruebas de Integración y Monitorización — Apuntes Teóricos
> Infraestructuras y Procesos de Soporte · UMA · 2025/2026

---

## Índice

**Tema 4.1 — CI/CD: Conceptos y Fundamentos**
1. [Qué es la Integración Continua (CI)](#1-qué-es-la-integración-continua-ci)
2. [Los 4 principios de CI](#2-los-4-principios-de-ci)
3. [Pipeline y flujo de trabajo](#3-pipeline-y-flujo-de-trabajo)
4. [Métricas y herramientas](#4-métricas-y-herramientas)
5. [Continuous Delivery vs Continuous Deployment](#5-continuous-delivery-vs-continuous-deployment)

**Tema 4.2 — GitHub Actions**
6. [Qué es GitHub Actions](#6-qué-es-github-actions)
7. [Arquitectura y componentes](#7-arquitectura-y-componentes)
8. [Triggers (on:)](#8-triggers-on)
9. [Jobs y Steps](#9-jobs-y-steps)
10. [Actions, Runners y Artefactos](#10-actions-runners-y-artefactos)
11. [Buenas prácticas en GitHub Actions](#11-buenas-prácticas-en-github-actions)
12. [Estrategias de organización de workflows](#12-estrategias-de-organización-de-workflows)

**Tema 4.3 — Pruebas de Integración**
13. [Unitarias vs Integración](#13-unitarias-vs-integración)
14. [Reglas para tests de integración](#14-reglas-para-tests-de-integración)
15. [Spring Boot y APIs REST](#15-spring-boot-y-apis-rest)

**Tema 4.4 — GitHub Actions con Docker y Code Review**
16. [CI/CD con Docker Hub](#16-cicd-con-docker-hub)
17. [Self-hosted Runners](#17-self-hosted-runners)
18. [Helm y Kubernetes](#18-helm-y-kubernetes)
19. [Code Review — Flujo colaborativo](#19-code-review--flujo-colaborativo)

**Tema 5.1 — Monitorización**
20. [Necesidad de la monitorización](#20-necesidad-de-la-monitorización)
21. [Prometheus](#21-prometheus)
22. [PromQL](#22-promql)
23. [Grafana](#23-grafana)
24. [Instrumentalización de aplicaciones](#24-instrumentalización-de-aplicaciones)
25. [Alertas con Prometheus y Alertmanager](#25-alertas-con-prometheus-y-alertmanager)

---

# Tema 4.1 — CI/CD: Conceptos y Fundamentos

## 1. Qué es la Integración Continua (CI)

En proyectos software grandes, los equipos se enfrentan a cuatro problemas recurrentes:
- **Coordinación:** con varios desarrolladores tocando el mismo código, los conflictos de merge se acumulan.
- **Calidad:** sin verificación automática, los defectos solo se detectan tarde.
- **Riesgo en producción:** los despliegues manuales introducen errores humanos.
- **Productividad:** los desarrolladores pierden tiempo en tareas repetitivas.

La solución es un principio simple: **"automatiza todo lo que se pueda automatizar"**.

**Definición de Martin Fowler (2006):**
> "Continuous Integration is a software development practice where members of a team integrate their work frequently, usually each person integrates at least daily — leading to multiple integrations per day."

### Por qué la frecuencia importa

El nombre es engañoso: lo "continuo" no significa que el proceso no pare, sino que la integración ocurre **muy frecuentemente**. Cuanto más pequeños los cambios que se integran, más pequeños son los conflictos y más rápido se detectan.

| Enfoque | Frecuencia | Riesgo | Esfuerzo de reparación |
|---|---|---|---|
| Tradicional | Semanal/Mensual | Alto | Días/Semanas |
| CI básico | Diario | Medio | Horas |
| CI óptimo | Múltiple al día | Bajo | Minutos |

**Dato real:** sin CI, los equipos dedican ~40% del tiempo a tareas de integración ("merge hell"). Con CI, se reduce a ~5%.

---

## 2. Los 4 principios de CI

### 1. Single Source of Truth (Única fuente de la verdad)

Todo el código debe vivir en un **repositorio centralizado** (Git). Sin repositorio único:
- Cada persona tiene "su versión" del código.
- No hay historial de cambios compartido.
- La colaboración es imposible o muy costosa.

Con Git centralizado: historial completo, no se pierde código, colaboración fluida.

### 2. Automate the Build (Automatización del build)

El proceso de compilación y empaquetado debe ejecutarse con **un único comando**, sin intervención humana, en el servidor de CI después de cada commit.

```bash
./build.sh    # O mvn package, npm build, etc.
```

El objetivo es que cualquier desarrollador (o servidor CI) pueda reproducir el build exactamente, sin pasos manuales ni configuraciones particulares de un ordenador concreto.

### 3. Test Automation (Automatización de pruebas)

Toda modificación de código debe verificarse automáticamente contra un conjunto de pruebas. La pirámide de pruebas típica:

```
        /\
       /E2E\        10%   (lentos, costosos, pocos)
      /──────\
     /Integrac\     20%   (comprueban que los módulos cooperan)
    /──────────\
   / Unitarias  \   70%   (rápidos, aislados, muchos)
  ──────────────────
```

Los tests son la red de seguridad que permite hacer cambios con confianza.

### 4. Fast Feedback (Retroalimentación rápida)

El ciclo `commit → resultado` debe ser **lo más corto posible**. Un ciclo de 2 horas es inútil: el desarrollador ya está en otro contexto cuando llega el resultado.

**Objetivo:** < 10 minutos en total.

```
Commit → Build (2-3 min) → Tests (5-7 min) → Feedback (30 seg)
```

Canales de feedback: email, Slack/Teams, dashboards de CI, integraciones en el IDE.

**Protocolo ante fallos:**
1. Notificación inmediata a todo el equipo y al autor.
2. El build roto bloquea nuevos commits ajenos hasta que se arregle.
3. El responsable debe priorizar arreglarlo ahora, no más tarde.
4. La filosofía es: **"Nobody goes home with a broken build"**.

---

## 3. Pipeline y flujo de trabajo

Un **pipeline de CI** es la secuencia de pasos que se ejecutan automáticamente tras un commit. Esquema típico:

```
Developer Push
     │
     ▼
Webhook Trigger        ← El servidor CI detecta el cambio
     │
     ▼
VM / Container prep    ← Se arranca un entorno limpio (Runner)
     │
     ▼
Checkout Code          ← git clone del repositorio
     │
     ▼
Install Dependencies   ← npm install / mvn install
     │
     ▼
Build / Compile        ← Compilar el código
     │
     ▼
Run Tests              ← Tests unitarios e integración
     │
     ▼
Upload Artifacts       ← JAR, binarios, reportes de cobertura
     │
     ▼
Notification           ← ✅ Success o ❌ Failure
```

**Por qué cada ejecución en un entorno limpio:** garantiza que el build no depende de nada que solo exista en la máquina del desarrollador. Si pasa en el entorno limpio, pasa en cualquier sitio.

---

## 4. Métricas y herramientas

### Métricas para evaluar la salud del pipeline

| Métrica | Descripción | Objetivo |
|---|---|---|
| **Build time** | Tiempo total del pipeline | < 10 minutos |
| **Pass rate** | % de builds que pasan | > 90% |
| **Coverage** | % de código cubierto por tests | > 70-80% según contexto |
| **Time to fix** | Tiempo medio para arreglar un build roto | Minutos, no horas |

### Herramientas de CI

| Categoría | Herramientas | Cuándo usarlas |
|---|---|---|
| Cloud-based (SaaS) | **GitHub Actions**, GitLab CI, CircleCI | Startups, open source, sin infra propia |
| Self-hosted | Jenkins, TeamCity, Bamboo | Empresas con control total on-premise |
| Enterprise cloud | Azure DevOps, AWS CodeBuild | Organizaciones atadas a un ecosistema cloud |

**Ventajas de GitHub Actions:**
- Integración 100% nativa con GitHub (código + CI en el mismo sitio).
- Más de 20.000 acciones reutilizables en el Marketplace.
- Capa gratuita generosa para open source y estudiantes.
- Configuración en YAML, igual que el resto de la industria.

---

## 5. Continuous Delivery vs Continuous Deployment

Son dos conceptos distintos que comparten las siglas "CD". La diferencia clave es el **control humano** antes de producción.

### Continuous Delivery (Entrega Continua)

> "Software siempre en estado listo para desplegar, pero la decisión de cuándo hacerlo es humana."

- CI completo → despliegue automático hasta **staging/pre-producción**.
- **Aprobación manual** antes de subir a producción.
- El timing del despliegue lo decide el negocio ("podemos desplegar en cualquier momento, decidimos cuándo").

```
[Build] → [Tests] → [Staging auto] → [✋ APROBACIÓN] → [Producción]
```

### Continuous Deployment (Despliegue Continuo)

> "Todo cambio que pasa los tests va a producción automáticamente, sin intervención humana."

- No hay aprobación manual. La confianza es total en los tests automatizados.
- Se apoya en rollback automático si las métricas post-despliegue empeoran.
- Requiere una batería de tests muy robusta y monitoring en tiempo real.

```
[Build] → [Tests] → [Staging auto] → [Producción auto] → [Monitor/Auto-rollback]
```

### Comparación

| Aspecto | Continuous Delivery | Continuous Deployment |
|---|---|---|
| Aprobación a producción | Manual (humano decide) | Automática |
| Riesgo | Menor (control humano) | Mayor (depende 100% de tests) |
| Velocidad al mercado | Más lento (aprobación) | Muy rápido |
| Madurez necesaria | Media | Alta (tests exhaustivos) |
| Uso típico | Mayoría de empresas | Startups muy maduras, grandes tech |

---

# Tema 4.2 — GitHub Actions

## 6. Qué es GitHub Actions

GitHub Actions es la plataforma CI/CD **integrada de forma nativa en GitHub**. En lugar de conectar un servidor CI externo (Jenkins, CircleCI) a tu repositorio de GitHub, todo vive en el mismo lugar.

**Características clave:**
- **Event-driven:** se activa por eventos de GitHub (push, PR, release, cron, etc.)
- **Hosted Runners:** GitHub proporciona servidores virtuales listos para usar, sin que tengas que mantener infraestructura.
- **Self-hosted Runners:** puedes usar tus propios servidores para jobs que necesitan acceso a red privada o hardware específico.
- **Marketplace:** más de 20.000 acciones prehechas para reutilizar.

**Limitaciones del plan gratuito:**
- Tiempo máximo por job: 6 horas.
- Concurrencia máxima: 20 jobs simultáneos.
- Hardware del runner: 2-core CPU, 7 GB RAM, 14 GB SSD.

**Vendor lock-in:** los workflows usan la sintaxis específica de GitHub Actions, por lo que migrar a otra plataforma requiere reescribir la configuración.

---

## 7. Arquitectura y componentes

GitHub Actions tiene una jerarquía de componentes anidados:

```
Repositorio
└── N Workflows  (ficheros .yml en .github/workflows/)
    └── M Jobs   (se ejecutan en paralelo por defecto)
        └── K Steps  (se ejecutan en serie dentro del mismo Runner)
             ├── uses: → invoca una Action del Marketplace
             └── run: → ejecuta un comando shell directamente
```

**Cada job corre en su propio Runner** (máquina virtual efímera e independiente). Esto significa que dos jobs no comparten sistema de ficheros ni memoria — para pasar datos entre ellos se usan **Artifacts** u **Outputs**.

### Estructura mínima de un workflow

```yaml
name: CI básico

on: [push]          # Trigger: cuándo se activa

jobs:
  build:
    runs-on: ubuntu-latest    # Qué Runner usar

    steps:
      - name: Descargar código
        uses: actions/checkout@v4

      - name: Compilar y testear
        run: |
          npm install
          npm test
```

Todo fichero `.yml` en `.github/workflows/` es un workflow independiente. GitHub los detecta automáticamente.

---

## 8. Triggers (on:)

El bloque `on:` define **qué evento activa el workflow**. El abanico es muy amplio:

```yaml
# Push a una rama específica
on:
  push:
    branches:
      - main
      - develop

# Pull Request (abre, actualiza o reabre)
on:
  pull_request:

# Programado con cron (lunes y miércoles a las 5:30)
on:
  schedule:
    - cron: '30 5 * * 1,3'

# Manual desde la interfaz web o API
on:
  workflow_dispatch:

# Al publicar una release
on:
  release:
    types: [published]
```

**Sintaxis cron:** `minuto hora día-del-mes mes día-de-la-semana`
- `'0 8 * * 1-5'` → 8:00 de lunes a viernes
- `'30 5 * * 1,3'` → 5:30 los lunes y miércoles

---

## 9. Jobs y Steps

### Jobs

Un job es un **bloque lógico de pasos** que se ejecutan en el mismo Runner. Características:
- **Paralelo por defecto:** si no se declaran dependencias, todos los jobs de un workflow se ejecutan a la vez en Runners separados.
- **Dependencias con `needs:`:** para ejecutarlos en secuencia.
- **Fail fast:** si un job falla, los que dependen de él se cancelan automáticamente.

```yaml
jobs:
  compile:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    needs: compile          # Solo corre si compile termina con éxito
    runs-on: ubuntu-latest
    steps: [...]

  build:
    needs: test             # Solo corre si test termina con éxito
    runs-on: ubuntu-latest
    steps: [...]
```

**Cadena resultante:** `compile → test → build`

Si `compile` falla, `test` y `build` se cancelan. Esto implementa la filosofía **fail fast**: no malgastes tiempo ejecutando lo que de todas formas va a fallar.

### Steps

Un step es una **unidad de trabajo dentro de un job**. Se ejecutan en serie, uno tras otro. Hay dos tipos:

**`uses:`** — invoca una Action (código reutilizable del Marketplace o propia):
```yaml
- name: Descargar el código
  uses: actions/checkout@v4

- name: Configurar Java 21
  uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '21'
    cache: maven
```

**`run:`** — ejecuta un comando shell directamente en el Runner:
```yaml
- name: Dar permisos al wrapper de Maven
  run: chmod +x ./mvnw

- name: Ejecutar tests
  run: |
    ./mvnw test --batch-mode
    ./mvnw jacoco:report
```

### Condicionales en steps

```yaml
# Se ejecuta SIEMPRE, aunque pasos anteriores fallen
- name: Subir reporte de tests
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: target/surefire-reports/

# Solo si TODO ha ido bien (comportamiento por defecto)
- name: Subir JAR
  if: success()
  uses: actions/upload-artifact@v4
  with:
    name: app-jar
    path: target/*.jar

# Solo si algo ha fallado
- name: Notificar fallo
  if: failure()
  run: echo "Algo ha fallado"
```

**¿Por qué `if: always()` en el reporte de tests?** Porque cuando los tests fallan, el step sin condicional se saltaría — y perderíamos precisamente el reporte que nos dice qué ha fallado.

---

## 10. Actions, Runners y Artefactos

### Actions

Una Action es una **unidad reutilizable de código** que hace una tarea específica. Se usan con `uses:` en los steps. No necesitas saber cómo están implementadas internamente.

**Versionado de Actions:**
```yaml
uses: actions/checkout@v4          # Tag mayor (recomendado: equilibrio entre reproducibilidad y actualizaciones de seguridad)
uses: actions/checkout@v4.1.7      # Tag exacto (máxima reproducibilidad)
uses: actions/checkout@abc1234     # SHA de commit (máxima seguridad)
```

**Buena práctica:** siempre fijar una versión (`@v4`). Nunca usar `@main` o `@latest` en producción — si el autor cambia la Action, tu pipeline se rompe sin previo aviso.

**Parámetros con `with:`:**
```yaml
- name: Configurar Java
  uses: actions/setup-java@v4
  with:
    distribution: temurin      # Distribución del JDK
    java-version: '21'
    cache: maven               # Activa caché de ~/.m2
```

### Runners

El Runner es la **máquina virtual donde se ejecuta un job**. GitHub proporciona runners hosted (gratuitos, efímeros) o puedes usar los tuyos (self-hosted).

```yaml
runs-on: ubuntu-latest    # Runner GitHub (Linux Ubuntu)
runs-on: windows-latest   # Runner GitHub (Windows)
runs-on: macos-latest     # Runner GitHub (macOS)
runs-on: self-hosted       # Tu propio servidor
```

**Hardware del runner hosted:** 2 cores, 7 GB RAM, 14 GB SSD. Preinstalado con Node.js, Python, Java, Maven, Docker, Git, etc.

### Artefactos

Los artefactos son **ficheros que se persisten tras finalizar un job** (por defecto 90 días). Sirven para:
- Guardar el JAR/binario para descarga.
- Persistir reportes de tests para depuración.
- **Compartir ficheros entre jobs** que corren en Runners distintos.

```yaml
# Subir artefacto
- name: Subir JAR
  uses: actions/upload-artifact@v4
  with:
    name: app-jar
    path: target/*.jar
    retention-days: 7     # Opcional, por defecto 90 días

# Descargar artefacto en otro job
- name: Descargar JAR
  uses: actions/download-artifact@v4
  with:
    name: app-jar
```

---

## 11. Buenas prácticas en GitHub Actions

### Caché de dependencias

Descargar dependencias (Maven, npm, pip…) en cada ejecución consume tiempo y ancho de banda. La caché persiste ficheros entre ejecuciones del mismo workflow.

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '21'
    cache: maven     # Cachea ~/.m2/repository automáticamente
```

Primera ejecución: descarga todo y guarda en caché. Ejecuciones posteriores: restaura la caché. Ahorro típico: 1-3 minutos por job.

### Flag `--batch-mode` en Maven

En CI no hay terminal interactiva. Maven por defecto muestra barras de progreso y puede pedir confirmaciones al usuario. El flag `-B` (o `--batch-mode`) desactiva esto:

```bash
./mvnw test --batch-mode
# Equivalente: ./mvnw -B test
```

Logs más limpios, sin bloqueos, sin colores innecesarios.

### Badges de estado

Los badges muestran el estado del workflow directamente en el `README.md`:

```markdown
![CI](https://github.com/USUARIO/REPO/actions/workflows/ci.yml/badge.svg?branch=main)
```

Formato: `https://github.com/{USUARIO}/{REPO}/actions/workflows/{FICHERO}.yml/badge.svg`

---

## 12. Estrategias de organización de workflows

Hay tres formas habituales de estructurar un pipeline CI/CD:

### A) Un workflow por fase

Cada fase tiene su propio fichero `.yml` y su propio badge:
```
compile.yml → test.yml → build.yml → integration-test.yml
```
- Máxima visibilidad: sabes exactamente qué fase falla.
- Más ficheros que mantener.
- Sin orden de ejecución entre workflows (corren independientes).

### B) Single-job — Todo en un job

Un solo fichero, un solo job, steps en serie:
```yaml
jobs:
  build:
    steps:
      - Compile
      - Test
      - Package
      - Integration Test
```
- Simple y suficiente para proyectos pequeños.
- Un solo badge.
- Si un step falla, los siguientes no se ejecutan.

### C) Multi-job — Jobs con dependencias

```yaml
jobs:
  compile: ...
  test:
    needs: compile
  build:
    needs: test
  integration-test:
    needs: build
  coverage:
    needs: compile    # Corre en paralelo con test
  javadoc:
    needs: compile    # Corre en paralelo con test
```

- Un fichero, múltiples jobs conectados con `needs:`.
- Fail fast: si compile falla, todo lo que depende se cancela.
- Jobs independientes corren en paralelo (ahorro de tiempo).
- GitHub muestra el grafo de dependencias visualmente en la pestaña Actions.

### Comparativa

| Criterio | Por fase | Single-job | Multi-job |
|---|---|---|---|
| Ficheros | Muchos | 1 | 1 |
| Badges | 1 por fase | 1 global | 1 global |
| Fail fast | No | Sí (serie) | Sí (`needs:`) |
| Paralelismo | Total | No | Sí (configurable) |
| Visibilidad | Alta | Baja | Alta (grafo) |
| Ideal para | Dashboards | Proyectos pequeños | Pipelines complejos |

---

# Tema 4.3 — Pruebas de Integración

## 13. Unitarias vs Integración

### Tests unitarios

Una prueba unitaria cumple tres requisitos:
1. **Verifica una única unidad de comportamiento** (función, clase, módulo).
2. **Es rápida** (milisegundos).
3. **Está aislada** de otras pruebas y dependencias externas (bases de datos, APIs, ficheros).

Los tests unitarios dan confianza en la lógica interna de cada componente. Pero **no garantizan que el sistema completo funcione correctamente** cuando los componentes interactúan entre sí.

### Tests de integración

Una prueba que no cumple al menos uno de los tres requisitos anteriores es un test de integración. El principal valor es la **fidelidad**: el test refleja el comportamiento real del sistema, con sus dependencias reales.

**Problemas que solo detectan los tests de integración:**
- Dos módulos tienen lógicas incompatibles aunque cada uno funcione solo.
- Datos que se corrompen al pasar de un módulo a otro.
- Interfaces incorrectas con la base de datos.
- Interfaces incorrectas con servicios externos.

### ¿Cuántos de cada tipo?

```
Pirámide de tests:
- Tests unitarios (70%): muchos, rápidos, baratos, verifican lógica de negocio.
- Tests de integración (20%): menos, más lentos, verifican colaboración entre módulos.
- Tests E2E (10%): muy pocos, lentos y costosos, verifican flujos completos.
```

**Estrategia práctica:**
- Unitarios → cubre todos los casos extremos de lógica de negocio.
- Integración → cubre el "camino feliz" (ruta principal sin errores) y los casos extremos que no se pueden probar en aislamiento.
- Tener pocas pruebas de integración globales por escenario de negocio es suficiente para garantizar la corrección del sistema.

---

## 14. Reglas para tests de integración

### Tipos de dependencias

- **Dependencias gestionadas:** base de datos, colas de mensajes, sistemas de ficheros. Las controla el equipo.
- **Dependencias no gestionadas:** APIs externas de terceros, servicios de email, pasarelas de pago. No se controlan.

### Regla de uso de dependencias

```
Dependencias gestionadas  → Dejar tal cual (usar la real) o versión ligera (in-memory)
Dependencias no gestionadas → Reemplazar con mocks
```

Por qué no mockear todo: los mocks dan falsa confianza. Si el mock no refleja el comportamiento real de la base de datos, las pruebas pasan aunque el sistema real falle. En tests de integración, lo que se quiere probar es precisamente esa interacción real.

**Versiones ligeras (in-memory):** para tests, se puede reemplazar PostgreSQL por H2 (base de datos en memoria), o un broker de mensajes real por una implementación en memoria. El test es más fiel que un mock pero más rápido que conectar con la base de datos real.

---

## 15. Spring Boot y APIs REST

### Spring Boot

Spring Boot es un framework Java para construir APIs RESTful. Simplifica enormemente la configuración de Spring:
- Servidor HTTP (Tomcat) embebido: no hay que desplegar un WAR en un servidor externo.
- Autoconfiguración basada en las dependencias del `pom.xml`.
- Base de datos H2 embebida para tests.

### Anotaciones fundamentales

```java
// Punto de entrada de la aplicación
@SpringBootApplication
public class MiApp {
    public static void main(String[] args) {
        SpringApplication.run(MiApp.class, args);
    }
}

// Define un controlador REST (la "C" del MVC)
@RestController
public class HelloController {

    // Responde a GET /hello
    @GetMapping("/hello")
    public String hello() {
        return "Hola";
    }

    // Responde a GET /cuenta — serializa automáticamente a JSON
    @GetMapping("/cuenta")
    public Cuenta getCuenta() {
        return new Cuenta(123);
    }

    // Parámetro en la URL: GET /nombre?name=Juan
    @GetMapping("/nombre")
    public String nombre(@RequestParam(value = "name", defaultValue = "alumno") String name) {
        return "Hola " + name;
    }

    // Cuerpo en la petición: POST /cuenta con JSON
    @PostMapping(value = "/cuenta", consumes = "application/json")
    public ResponseEntity<?> save(@RequestBody Cuenta cuenta) {
        if (listCuentas.contains(cuenta))
            return ResponseEntity.internalServerError().body("Ya existe");
        listCuentas.add(cuenta);
        return ResponseEntity.ok().body("Añadida");
    }
}
```

### Anotaciones HTTP

| Anotación | Método HTTP | Uso típico |
|---|---|---|
| `@GetMapping` | GET | Obtener recurso |
| `@PostMapping` | POST | Crear recurso |
| `@PutMapping` | PUT | Actualizar recurso completo |
| `@DeleteMapping` | DELETE | Eliminar recurso |

### ResponseEntity

Permite construir respuestas HTTP con control total sobre el código de estado, cabeceras y cuerpo:

```java
return ResponseEntity.ok().body("Todo bien");           // 200
return ResponseEntity.notFound().build();               // 404
return ResponseEntity.internalServerError().body("Error"); // 500
return ResponseEntity.status(201).body(nuevo);          // 201 Created
```

---

# Tema 4.4 — GitHub Actions con Docker y Code Review

## 16. CI/CD con Docker Hub

### El flujo completo: de commit a contenedor

```
Developer push
      │
      ▼
GitHub Actions CI (compile + test)
      │
      ▼
docker/build-push-action → Construye imagen
      │
      ▼
Docker Hub → Sube la imagen
      │
      ▼
Servidor / Kubernetes → Descarga y despliega
```

### Generación de tokens en Docker Hub

**Nunca usar la contraseña real de Docker Hub en GitHub Actions.** En su lugar, generar un **Personal Access Token**:

1. `hub.docker.com` → Account Settings → Personal Access Tokens → New Access Token
2. Asignar nombre descriptivo (ej: "github-actions-ci")
3. Permisos: **Read & Write** (para que pueda hacer push)
4. Copiar el token inmediatamente — no se vuelve a mostrar

### Configurar los Secrets en GitHub

Nunca escribir credenciales en el YAML del workflow. Usar **GitHub Secrets**:

- Repositorio → Settings → Secrets and variables → Actions
- Secrets a crear:
  - `DOCKERHUB_USERNAME`: usuario de Docker Hub
  - `DOCKERHUB_TOKEN`: el Access Token generado

### Workflow completo con Docker

```yaml
name: CI/CD con Docker

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Login en Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extraer metadatos (tags y labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKERHUB_USERNAME }}/mi-app

      - name: Configurar QEMU (para builds multi-plataforma)
        uses: docker/setup-qemu-action@v3

      - name: Configurar Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build y Push de la imagen
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

**`docker/metadata-action`:** genera automáticamente tags inteligentes basados en la rama, el tag de Git, el SHA del commit, etc. Evita tener que hardcodear los tags manualmente.

**`docker/setup-qemu-action` + `platforms:`:** permite construir la imagen para múltiples arquitecturas (AMD64 para servidores, ARM64 para Apple Silicon o Raspberry Pi) desde un único runner.

---

## 17. Self-hosted Runners

### Qué son y para qué sirven

Un self-hosted runner es un **agente que instalas en tu propio servidor** para ejecutar jobs de GitHub Actions. El servidor se conecta a GitHub mediante un túnel seguro y ejecuta los jobs cuando hay trabajo pendiente.

**¿Por qué usarlos?**
- **Coste:** no consume minutos de GitHub Actions (que son limitados en el plan gratuito).
- **Acceso local:** puede conectarse a bases de datos internas, clústeres Kubernetes privados, o cualquier servicio que no esté expuesto a Internet.
- **Rendimiento:** hardware propio, potencialmente más potente que el runner estándar de 2 cores.
- **Despliegue en Kubernetes:** el runner puede tener kubectl configurado para desplegar directamente.

### Cómo configurar un self-hosted runner

1. Repositorio → Settings → Actions → Runners → New self-hosted runner
2. Seleccionar SO (Linux/Windows/macOS)
3. Ejecutar los comandos que proporciona GitHub:
   - Descargar el binario del agente
   - Configurar: vincula la máquina con un token único de GitHub
   - Ejecutar: `./run.sh` (o instalarlo como servicio para que arranque automáticamente)

```yaml
# Usar un self-hosted runner en el workflow
jobs:
  deploy:
    runs-on: self-hosted    # En lugar de ubuntu-latest
    steps:
      - run: kubectl apply -f k8s/deployment.yaml
```

---

## 18. Helm y Kubernetes

### El problema con los YAMLs de Kubernetes

Cuando hay muchos servicios, los YAMLs de Kubernetes presentan problemas:
- Configuración duplicada entre entornos (dev, staging, prod).
- Difícil reutilización de configuraciones similares.
- Errores manuales frecuentes al copiar y modificar.

### Qué es Helm

**Helm es el package manager de Kubernetes.** Es a Kubernetes lo que `apt` es a Ubuntu o `npm` a Node.js.

- Empaqueta las definiciones de Kubernetes en **Charts** (plantillas reutilizables).
- Permite instalar, actualizar y desinstalar aplicaciones completas con un solo comando.
- Usa un fichero `values.yaml` para variables dinámicas (lo que Kubernetes no tiene nativamente).
- Mantiene el historial de versiones de cada instalación.

```bash
helm install mi-app ./chart          # Instalar
helm upgrade mi-app ./chart          # Actualizar
helm rollback mi-app 1               # Volver a la versión 1
helm uninstall mi-app                # Desinstalar
helm list                            # Ver releases instalados
```

Un Chart tiene la estructura:
```
mi-chart/
├── Chart.yaml       # Metadatos del chart
├── values.yaml      # Valores por defecto (sobreescribibles)
└── templates/       # YAMLs parametrizados con Go templates
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## 19. Code Review — Flujo colaborativo

### ¿Por qué hacer code review?

El code review es la revisión por parte de otro desarrollador del código antes de integrarlo en la rama principal. Beneficios:
- **Detectar bugs** que los tests no cubren (lógica incorrecta, edge cases ignorados).
- **Conocimiento compartido** del código entre el equipo.
- **Mejora de calidad** del diseño y la legibilidad.
- **Cultura de equipo** de responsabilidad compartida del código.

### El flujo fork + Pull Request

```
Repositorio original (Propietario)
        │
        │ Fork
        ▼
Repositorio fork (Colaborador)
        │
        │ git checkout -b feature/mi-cambio
        │ Implementar + tests + commit + push
        ▼
Pull Request (fork → original)
        │
        │ Code review del Propietario
        │ Comentarios con prefijos
        ▼
Correcciones → Aprobación → Merge
```

### Checklist de revisión

**Corrección:**
- ¿El código hace lo que dice que hace?
- ¿Maneja correctamente los casos de error?
- ¿Funciona para valores límite (0, null, negativos)?

**Calidad:**
- ¿Los nombres de métodos y variables son descriptivos?
- ¿Hay lógica duplicada que podría extraerse?
- ¿El diseño es el adecuado para este problema?

**Tests:**
- ¿Se testea el camino feliz de cada método nuevo?
- ¿Se testean los casos de error?
- ¿Faltan tests relevantes (edge cases)?

**Estilo:**
- ¿El código sigue las convenciones del proyecto?
- ¿Los imports están organizados?

### Prefijos de comentarios en la revisión

Los comentarios en una PR deben ser claros sobre su importancia:

| Prefijo | Significado | ¿Bloquea el merge? |
|---|---|---|
| `blocker:` | Error crítico que debe corregirse antes del merge | Sí |
| `suggestion:` | Mejora recomendada, pero no obligatoria | No |
| `nit:` | Detalle menor de estilo o nomenclatura | No |
| `question:` | Duda o petición de aclaración | No |

**Ejemplos:**
```
blocker: missing tests for edge cases in power method (negative exponent, zero). Add them before merging.

suggestion: the exception message in divide could include the values for easier debugging: "Cannot divide " + a + " by zero"

nit: @DisplayName of divideByZeroThrowsException could be more descriptive.

question: what happens with power(0, 0)? Math.pow returns 1.0, but is that the expected behavior?
```

### Ciclo de revisión completo

1. **Colaborador abre la PR** con descripción clara: qué hace, cómo probarlo, checklist de autor.
2. **Propietario revisa** y escribe comentarios con prefijos.
3. Si hay `blocker:`, marca la PR como **"Changes requested"** — no se puede hacer merge hasta resolverlos.
4. **Colaborador responde** a cada comentario: aplica el cambio o justifica por qué no.
5. **Propietario aprueba** cuando todos los blockers están resueltos.
6. **Merge** y limpieza de la rama de feature.

---

# Tema 5.1 — Monitorización

## 20. Necesidad de la monitorización

En sistemas modernos (microservicios, contenedores, entornos cloud), la complejidad es tal que los problemas no son predecibles. La monitorización permite:
- **Detectar problemas antes de que afecten a los usuarios.**
- **Optimizar el rendimiento** con datos reales, no intuiciones.
- **Tomar decisiones informadas** sobre capacidad, escalado y recursos.
- **Diagnosticar** post-mortem qué ocurrió cuando algo falló.

Sin monitorización, operar un sistema distribuido es volar a ciegas.

---

## 21. Prometheus

### Qué es

Prometheus es un **sistema de monitorización y alertas open source** diseñado específicamente para entornos dinámicos (contenedores, Kubernetes). Fue creado en SoundCloud y actualmente es un proyecto de la CNCF.

**Características:**
- **Modelo pull (scraping):** Prometheus va él mismo a preguntar a las aplicaciones por sus métricas (HTTP GET a un endpoint `/metrics`). Esto simplifica la arquitectura — las apps no necesitan saber dónde está Prometheus.
- **Series temporales:** almacena datos identificados por nombre de métrica + pares clave-valor (labels). Ej: `http_requests_total{method="GET", status="200"}`
- **PromQL:** lenguaje de consultas propio, potente y flexible.
- **Autónomo:** cada servidor Prometheus es independiente, sin almacenamiento distribuido.

### Arquitectura

```
┌─────────────────────────────────────────────┐
│  Servidor Prometheus                         │
│  ┌──────────┐  ┌──────┐  ┌────────────────┐ │
│  │ Retrieval│→ │ TSDB │  │  HTTP Server   │ │
│  │(scraping)│  │(BBDD)│  │ (API + UI web) │ │
│  └──────────┘  └──────┘  └────────────────┘ │
└─────────────────────────────────────────────┘
         │ scraping (HTTP pull)
         ▼
┌───────────────────────────────────────────────────┐
│  Targets (fuentes de métricas)                     │
│  - Exporters (node-exporter, cAdvisor, etc.)       │
│  - Aplicaciones instrumentadas (Spring Boot, Flask)│
└───────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────────┐
│  Alertmanager    │     │  Grafana          │
│  (alertas)       │     │  (dashboards)     │
└──────────────────┘     └──────────────────┘
```

### Componentes

- **Retrieval:** realiza el scraping periódico de las métricas desde los targets configurados.
- **TSDB (Time Series Database):** base de datos de series temporales donde se almacenan las métricas.
- **HTTP Server:** expone la API REST y la UI web para consultas.
- **Exporters:** adaptadores que exponen métricas de sistemas que no hablan nativamente con Prometheus (node-exporter para el SO, cAdvisor para contenedores).
- **Pushgateway:** para jobs batch o efímeros que no pueden ser scrapeados (son ellos quienes empujan las métricas).
- **Alertmanager:** gestiona las alertas: agrupa duplicados, las enruta a email/Slack/PagerDuty.

### Configuración básica (`prometheus.yml`)

```yaml
global:
  scrape_interval: 15s       # Frecuencia de scraping
  evaluation_interval: 15s   # Frecuencia de evaluación de reglas de alerta

scrape_configs:
  - job_name: 'prometheus'           # El propio Prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'        # Métricas del SO
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'mi-app-spring-boot'   # Aplicación Spring Boot
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: /actuator/prometheus  # Ruta custom del endpoint
```

### Node Exporter — métricas del sistema operativo

Node Exporter expone métricas de hardware y SO de máquinas Unix/Linux:
- **CPU:** uso por modo (idle, user, system, iowait…)
- **Memoria:** total, disponible, usada, buffers, caché
- **Disco:** espacio usado/libre, operaciones de I/O, latencia
- **Red:** bytes enviados/recibidos, errores, paquetes
- **Load average**

```yaml
# docker-compose.yml
node-exporter:
  image: prom/node-exporter:v1.7.0
  ports:
    - "9100:9100"
  networks:
    - monitoring
```

Las métricas están disponibles en `http://localhost:9100/metrics`.

### cAdvisor — métricas de contenedores

cAdvisor (Container Advisor) expone métricas de uso de recursos por contenedor:
- CPU usage por contenedor
- Memoria utilizada y límites
- Network I/O
- Filesystem I/O

```yaml
# docker-compose.yml
cadvisor:
  image: gcr.io/cadvisor/cadvisor:v0.56.2
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
  ports:
    - "8080:8080"
  privileged: true
```

---

## 22. PromQL

PromQL (Prometheus Query Language) es el lenguaje de consultas de Prometheus. Con él se filtran métricas, se calculan tasas, percentiles y agregaciones. Es la base para dashboards y alertas.

### Consulta simple

```promql
node_cpu_seconds_total
```

Devuelve la métrica `node_cpu_seconds_total` con todos sus labels.

### Filtrado con `{}`

```promql
node_cpu_seconds_total{mode="user"}         # Solo modo usuario
node_cpu_seconds_total{mode="user"}[1m]     # Valores en el último minuto (vector de rango)
container_cpu_usage_seconds_total{name="mi-contenedor"}
```

### Función `rate()` — tasa de cambio por segundo

```promql
rate(node_cpu_seconds_total{mode="user"}[1m])
```

`rate()` calcula la tasa promedio de incremento por segundo en el intervalo dado. **Importante:** para métricas que solo incrementan (contadores), siempre usa `rate()` o `increase()`, no el valor directo.

### Agregaciones

```promql
avg(rate(node_cpu_seconds_total[5m]))              # Promedio de todos los núcleos
sum(rate(node_cpu_seconds_total[5m]))              # Suma total
count by (mode) (rate(node_cpu_seconds_total[5m])) # Contar agrupando por modo
sum by (mode) (rate(node_cpu_seconds_total[5m]))   # Suma agrupando por modo
```

### Percentiles

```promql
quantile(0.95, rate(node_cpu_seconds_total{mode="user"}[5m]))   # Percentil 95
```

### Consultas útiles de node-exporter

```promql
# CPU libre (modo idle)
rate(node_cpu_seconds_total{mode="idle"}[1m])

# Memoria disponible en GB
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# Porcentaje de memoria usada
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))

# Espacio en disco usado
node_filesystem_size_bytes - node_filesystem_free_bytes
```

### Consultas útiles de cAdvisor

```promql
# CPU por contenedor (ascendente)
sort(rate(container_cpu_usage_seconds_total[5m]))

# Memoria por contenedor (descendente) en GB
sort_desc(container_memory_usage_bytes) / 1024 / 1024 / 1024
```

---

## 23. Grafana

### Qué es

Grafana es una plataforma open source de **visualización y análisis de métricas**. Permite crear dashboards interactivos y conectarse a múltiples fuentes de datos (Prometheus, InfluxDB, Elasticsearch, MySQL, PostgreSQL, etc.).

**Funcionalidades clave:**
- Dashboards con múltiples tipos de paneles: time series, gauges, stats, tablas, heatmaps.
- Filtros dinámicos e interactivos (drill-down).
- Sistema de alertas propio (evalúa condiciones y notifica).
- Anotaciones para marcar eventos en los gráficos.
- Miles de dashboards prediseñados en la comunidad.

### Conectar Grafana con Prometheus

1. Configuration → Data Sources → Add data source → Prometheus
2. URL: `http://prometheus:9090` (si están en la misma red de Docker)
3. Save & Test

### Importar dashboards prediseñados

Grafana Labs ofrece dashboards listos para usar identificados por un ID:

```
Node Exporter Full          → ID: 1860
Docker Container & Host     → ID: 10619 (cAdvisor)
Kubernetes Cluster          → ID: 7249
```

Grafana → Dashboards → Import → Introducir el ID → Load

### Crear paneles personalizados

1. New Dashboard → Add panel
2. Escribir la consulta PromQL en el editor de métricas
3. Seleccionar el tipo de visualización (panel derecho)
4. Configurar umbrales, colores y opciones del eje
5. Guardar el panel y el dashboard

---

## 24. Instrumentalización de aplicaciones

### Qué es la instrumentalización

Instrumentalizar una aplicación es **añadir código que expone métricas internas** de la propia aplicación: cuántas requests recibe, cuánto tardan, cuántos usuarios hay activos, cuántos errores ocurren. Sin instrumentalización, Prometheus solo puede ver métricas del sistema (CPU, memoria), no de la aplicación en sí.

### Tipos de métricas

| Tipo | Descripción | Ejemplo |
|---|---|---|
| **Counter** | Solo incrementa. Nunca baja. | Total de requests, total de errores |
| **Gauge** | Valor que sube y baja. Estado actual. | Usuarios activos, memoria usada, conexiones abiertas |
| **Histogram** | Distribución de valores en buckets. | Latencias, tamaños de respuesta |
| **Timer** | Mide tiempo de ejecución de una operación | Tiempo de una llamada a API |

### Instrumentalización en Python (prometheus_client)

```python
from flask import Flask
from prometheus_client import Counter, Histogram, Gauge, generate_latest

app = Flask(__name__)

# Definir métricas
request_count = Counter('app_requests_total', 'Total de requests', ['method', 'endpoint'])
request_duration = Histogram('app_request_duration_seconds', 'Duración de requests')
active_users = Gauge('app_active_users', 'Usuarios activos actualmente')

@app.route('/api/data')
@request_duration.time()    # Mide automáticamente la duración
def get_data():
    request_count.labels(method='GET', endpoint='/api/data').inc()
    return {"message": "OK"}

@app.route('/metrics')
def metrics():
    return generate_latest()    # Expone métricas en formato Prometheus
```

### Instrumentalización en Java con Spring Boot (Micrometer)

**Micrometer** es la librería de instrumentalización estándar para aplicaciones JVM. Spring Boot Actuator la integra automáticamente.

**Dependencias en `pom.xml`:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**Configuración en `application.properties`:**
```properties
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.endpoint.prometheus.enabled=true
management.prometheus.metrics.export.enabled=true
management.metrics.tags.application=mi-app   # Tag global para identificar la app
```

Con esto, sin escribir nada más, Spring Boot expone automáticamente en `/actuator/prometheus`:
- Uso de CPU
- Memoria JVM (heap y non-heap)
- Número de requests HTTP (por endpoint, método, código de respuesta)
- Threads activos
- Garbage Collector

### Métricas personalizadas con Micrometer

**Counter (contar eventos):**
```java
@Component
public class BooksMetrics {
    private final Counter booksSavedCounter;

    public BooksMetrics(MeterRegistry registry) {
        this.booksSavedCounter = Counter.builder("books.save.total")
            .description("Total de libros guardados")
            .register(registry);
    }

    public void increment() {
        booksSavedCounter.increment();
    }
}

// En el controlador:
@PostMapping("/libros")
public String save(Libro libro) {
    libroService.add(libro);
    booksMetrics.increment();    // Incrementar contador
    return "ok";
}
```

**Timer con `@Timed` (medir latencia):**
```java
// Configuración necesaria (una vez)
@Configuration
public class MetricsConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

// Uso en el controlador
@PostMapping("/libros")
@Timed("book.creation.latency")    // Mide tiempo de ejecución de este método
public ResponseEntity<?> save(@RequestBody Libro libro) { ... }
```

**Gauge (valor actual — usuarios conectados):**
```java
@Component
public class ConnectedUsersMetrics implements HttpSessionListener {
    private final AtomicInteger connectedUsers = new AtomicInteger(0);

    public ConnectedUsersMetrics(MeterRegistry registry) {
        Gauge.builder("usuarios.conectados", connectedUsers, AtomicInteger::get)
            .description("Usuarios con sesión HTTP activa")
            .register(registry);
    }

    @Override
    public void sessionCreated(HttpSessionEvent se) {
        connectedUsers.incrementAndGet();
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        connectedUsers.updateAndGet(current -> Math.max(0, current - 1));
    }
}
```

### Métricas JVM relevantes (disponibles automáticamente)

```promql
jvm_memory_used_bytes{area="heap"}            # Memoria heap usada (detecta fugas)
jvm_threads_live_threads                       # Hilos activos (detecta bloqueos)
process_uptime_seconds                         # Segundos desde último arranque
http_server_requests_seconds_sum               # Latencia acumulada por endpoint
http_server_requests_seconds_count             # Total de requests por endpoint
```

---

## 25. Alertas con Prometheus y Alertmanager

### Arquitectura de alertas

```
Prometheus evalúa reglas (alerts.yml)
      │
      │ si condición se cumple durante X tiempo
      ▼
Alertmanager recibe la alerta
      │
      │ deduplica, agrupa, enruta
      ▼
Receptores: email, Slack, PagerDuty, Grafana...
```

### Configuración en Prometheus

Enlazar el fichero de reglas en `prometheus.yml`:

```yaml
rule_files:
  - "alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### Formato de reglas de alerta (`alerts.yml`)

```yaml
groups:
  - name: mi-sistema
    rules:
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_memory_limit_bytes) * 100 > 80
        for: 2m          # Debe cumplirse durante 2 minutos seguidos (evita falsos positivos)
        labels:
          severity: warning
        annotations:
          summary: "Alto uso de memoria en contenedor"
          description: "El contenedor {{ $labels.name }} usa más del 80% de memoria"
```

**Campos:**
- `alert`: nombre de la alerta (obligatorio)
- `expr`: condición en PromQL (obligatorio)
- `for`: tiempo mínimo que debe cumplirse antes de disparar (recomendado — evita alertas por picos momentáneos)
- `labels`: metadatos adicionales (severity, team, etc.)
- `annotations`: descripción legible por humanos

### Alertmanager (`alertmanager.yml`)

**Configuración mínima (sin notificaciones, solo visualizar en Grafana):**
```yaml
global:
  resolve_timeout: 1m
route:
  receiver: "null-receiver"
receivers:
  - name: "null-receiver"
```

**Configuración con notificación por email:**
```yaml
global:
  resolve_timeout: 1m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'tu-email@gmail.com'
  smtp_auth_username: 'tu-email@gmail.com'
  smtp_auth_password: 'tu-app-password-16-letras'   # Contraseña de aplicación, no la real
  smtp_require_tls: true

route:
  receiver: 'email-alertas'
receivers:
  - name: 'email-alertas'
    email_configs:
      - to: 'equipo@empresa.com'
```

### Alertmanager en Docker Compose

```yaml
alertmanager:
  image: prom/alertmanager:v0.28.0
  volumes:
    - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
  command:
    - '--config.file=/etc/alertmanager/alertmanager.yml'
  ports:
    - "9093:9093"
  networks:
    - monitoring
```

### Ver alertas en Grafana

Grafana → Configuration → Data Sources → Add data source → Alertmanager
- URL: `http://alertmanager:9093`

Grafana → Alerting → Alert rules → Ver el estado de las alertas definidas en Prometheus (firing, pending, inactive).

---

## Resumen: Stack completo de observabilidad

```
Aplicación (Spring Boot/Flask)
    │  /actuator/prometheus o /metrics
    │
    ▼
Prometheus (scraping cada 15s)
    │  PromQL queries
    ▼
Grafana (dashboards + alertas)
    │
    ▼
Alertmanager (notificaciones)
    │
    ▼
Email / Slack / PagerDuty

Exporters:
- node-exporter  → métricas del SO (CPU, RAM, disco, red)
- cAdvisor       → métricas de contenedores Docker
- kube-state-metrics → estado de objetos Kubernetes
```
