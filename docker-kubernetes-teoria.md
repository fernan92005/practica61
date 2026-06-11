# Docker y Kubernetes — Apuntes Teóricos
> Infraestructuras y Procesos de Soporte · UMA · 2025/2026

---

## Índice

**Parte 1 — Docker**
1. [De las VMs a los contenedores](#1-de-las-vms-a-los-contenedores)
2. [Qué es Docker](#2-qué-es-docker)
3. [Arquitectura de Docker](#3-arquitectura-de-docker)
4. [Imágenes, registros y contenedores](#4-imágenes-registros-y-contenedores)
5. [Ejecutar contenedores](#5-ejecutar-contenedores)
6. [Dockerfiles](#6-dockerfiles)
7. [Docker Compose](#7-docker-compose)
8. [Redes en Docker](#8-redes-en-docker)
9. [Volúmenes en Docker](#9-volúmenes-en-docker)

**Parte 2 — Kubernetes**
10. [Qué es Kubernetes y por qué](#10-qué-es-kubernetes-y-por-qué)
11. [Arquitectura de Kubernetes](#11-arquitectura-de-kubernetes)
12. [Objetos fundamentales: Pod, Deployment, Service](#12-objetos-fundamentales-pod-deployment-service)
13. [Despliegue con YAML](#13-despliegue-con-yaml)
14. [ConfigMaps y Secrets](#14-configmaps-y-secrets)
15. [Volúmenes persistentes en Kubernetes](#15-volúmenes-persistentes-en-kubernetes)
16. [Health Checks (Probes)](#16-health-checks-probes)
17. [Escalado de aplicaciones](#17-escalado-de-aplicaciones)
18. [Actualizaciones y Rollbacks](#18-actualizaciones-y-rollbacks)

---

# Parte 1 — Docker

## 1. De las VMs a los contenedores

### ¿Qué es la virtualización?

Virtualizar es crear una versión lógica de un recurso hardware (CPU, memoria, red, disco). El software que lo hace se llama **hipervisor**:

- **Nivel 1 (bare metal):** corre directamente sobre el hardware. VMware ESXi, Proxmox, Hyper-V. Más eficiente.
- **Nivel 2 (hosted):** corre sobre un SO existente. VirtualBox, VMware Workstation, QEMU. Más fácil de instalar.

### El problema de las Máquinas Virtuales

Las VMs son potentes pero pesadas: cada VM incluye un **sistema operativo completo**, que consume CPU, RAM y disco sin producir valor de negocio. Además:
- Cada SO tiene su propio mantenimiento y actualizaciones.
- Cada SO puede requerir licencias.
- Arrancar una VM lleva minutos.

### ¿Qué son los contenedores?

Los contenedores son **virtualización a nivel de sistema operativo** (no a nivel de hardware). En lugar de virtualizar el hardware, virtualizan el entorno de ejecución de las aplicaciones.

**La diferencia clave:** una VM reproduce un ordenador entero (hardware virtualizado + SO completo + aplicación). Un contenedor solo empaqueta la aplicación con sus dependencias, y comparte el kernel del SO del host.

```
┌──────────────────────┐    ┌──────────────────────┐
│  Máquina Virtual     │    │  Contenedor           │
│  ┌────────────────┐  │    │  ┌────────────────┐   │
│  │   Aplicación   │  │    │  │   Aplicación   │   │
│  ├────────────────┤  │    │  ├────────────────┤   │
│  │  SO completo   │  │    │  │   Deps/Libs    │   │
│  │  (500 MB+)     │  │    │  └────────────────┘   │
│  ├────────────────┤  │    │   (comparte kernel)    │
│  │  Hardware      │  │    │                        │
│  │  Virtualizado  │  │    └──────────────────────┘
│  └────────────────┘  │
└──────────────────────┘
```

### Ventajas de los contenedores sobre las VMs

- **Arranque en milisegundos** (vs minutos de las VMs). Solo hay que ejecutar un proceso, no arrancar un SO.
- **Mucho más ligeros:** una imagen base de Alpine son ~5 MB. Una VM mínima son cientos de MB.
- **Misma imagen en cualquier entorno:** desarrollo, staging, producción. "Funciona en mi máquina" desaparece.
- **Aislamiento:** cada contenedor tiene su propio sistema de ficheros, red y procesos. Un `ps` dentro de un contenedor solo muestra los procesos del contenedor.
- **Escalabilidad:** se pueden arrancar decenas de copias del mismo contenedor en segundos.

### Cómo funcionan internamente — cgroups y namespaces

Los contenedores se basan en dos características del kernel Linux introducidas en 2008:

- **cgroups (control groups):** limitan y priorizan recursos. Permiten decirle al kernel "este conjunto de procesos no puede usar más de 512 MB de RAM ni más del 50% de CPU". Sin cgroups, un contenedor podría consumir todos los recursos del host.
- **namespaces:** aíslan el entorno. Cada contenedor tiene su propio "espacio de nombres" para procesos (PID), red, usuarios, sistema de ficheros. Los procesos de un contenedor no pueden ver ni interferir con los de otro.

---

## 2. Qué es Docker

Docker es la plataforma que popularizó los contenedores al hacerlos **fáciles de usar**. Antes de Docker, crear y gestionar contenedores LXC requería configurar namespaces y cgroups manualmente.

**La plataforma Docker incluye:**

- **Docker Engine:** el núcleo. Compuesto por el daemon, la API REST y el cliente CLI (`docker`).
- **Docker Hub:** registro público de imágenes. `hub.docker.com`
- **Docker Compose:** gestión de aplicaciones multi-contenedor con YAML.
- **Docker Desktop:** aplicación de escritorio que integra todo para Windows/Mac.
- **Docker Scout:** análisis de vulnerabilidades en imágenes.
- **BuildKit:** motor optimizado para construir imágenes.

**El ciclo de vida que Docker facilita:**
```
Código → Dockerfile → Imagen → Registro → Contenedor
(Construir)           (Guardar)            (Ejecutar)
```

**Organismos de estandarización:**
- **OCI (Open Container Initiative):** estándar de formatos y runtimes de contenedores, creado en 2015 por Docker + Linux Foundation.
- **CNCF (Cloud Native Computing Foundation):** fundación que gestiona proyectos cloud-native como Docker Engine, Kubernetes, Prometheus, etc.

---

## 3. Arquitectura de Docker

Docker usa una **arquitectura cliente-servidor**:

```
┌─────────────┐         ┌──────────────────────────────────────┐
│  Docker CLI │ ──────► │  Docker Daemon (dockerd)              │
│  (cliente)  │  REST   │  ┌──────────────┐  ┌──────────────┐ │
└─────────────┘  API    │  │  containerd  │  │  Registros   │ │
                         │  │  ┌────────┐  │  │  (Hub, ECR) │ │
                         │  │  │  runc  │  │  └──────────────┘ │
                         │  │  └────────┘  │                    │
                         │  └──────────────┘                    │
                         └──────────────────────────────────────┘
```

**Capas de ejecución:**
- `dockerd` (Docker daemon): gestiona imágenes, redes, volúmenes y el ciclo de vida de los contenedores. El cliente se comunica con él.
- `containerd`: runtime de alto nivel. Gestiona el ciclo de vida de contenedores (pull de imágenes, snapshots, etc.). Lo usa tanto Docker como Kubernetes.
- `runc`: runtime de bajo nivel. El que realmente ejecuta el contenedor usando las syscalls del kernel (namespaces + cgroups). Implementa el estándar OCI.

**Cliente y servidor pueden estar en la misma máquina** (lo habitual en desarrollo) o separados (producción remota).

---

## 4. Imágenes, registros y contenedores

### Imágenes

Una imagen es una **plantilla de solo lectura** a partir de la cual se crean contenedores. Contiene el sistema de ficheros (SO base, dependencias, código de la aplicación) y metadatos sobre cómo ejecutarse.

**Características clave:**
- **Inmutables:** ningún contenedor puede modificar la imagen base. Si un contenedor escribe un fichero, ese cambio va a su capa propia, no a la imagen.
- **Por capas:** cada instrucción del Dockerfile añade una capa. Las capas se cachean y se reutilizan entre imágenes que comparten una base común. Esto hace que las descargas sean eficientes.
- **Etiquetas (tags):** identifican versiones o variantes. `nginx:latest`, `ubuntu:24.04`, `python:3.11-slim`. Si no se especifica tag, se usa `latest`.

### Registros

Un registro es un **almacén de imágenes**. Los repositorios de imágenes viven en registros.

- **Docker Hub (`hub.docker.com`):** el registro público oficial. Contiene imágenes oficiales (mantenidas por el proyecto) y de la comunidad.
- **Registros privados:** GitHub Container Registry (GHCR), AWS ECR, Azure ACR, Google Artifact Registry, GitLab Container Registry. Para imágenes propietarias o internas.

### Contenedores

Un contenedor es una **instancia en ejecución de una imagen**. Es a la imagen lo que un proceso es a un ejecutable.

Cuando arranca un contenedor, Docker añade una **capa de lectura/escritura** encima de la imagen (que es de solo lectura). Todo lo que el contenedor escribe va a esa capa. Si el contenedor se elimina, esa capa desaparece con él — **los datos se pierden** (para evitarlo: volúmenes).

```
┌─────────────────────────────┐  ← Capa R/W del contenedor (efímera)
├─────────────────────────────┤
│    Capa app (tu código)     │  ← Capas de la imagen (solo lectura)
├─────────────────────────────┤
│    Capa dependencias        │
├─────────────────────────────┤
│    SO base (ubuntu/alpine)  │
└─────────────────────────────┘
```

---

## 5. Ejecutar contenedores

### Modos de ejecución

**Modo foreground (interactivo):** el terminal queda bloqueado con la ejecución del contenedor.

```bash
docker run -it ubuntu bash
# -i: mantiene STDIN abierto (interactivo)
# -t: asigna una pseudo-terminal (TTY)
# Combinado: -it → sesión de shell interactiva
```

**Modo detached (segundo plano):** el contenedor corre en background. El terminal queda libre.

```bash
docker run -d -p 8080:80 --name mi-nginx nginx
# -d: detached mode
# -p 8080:80: puerto del host:puerto del contenedor
# --name: nombre del contenedor
```

### Qué hace Docker al ejecutar `docker run`

1. Busca la imagen localmente. Si no existe, la descarga del registro.
2. Crea un nuevo contenedor a partir de la imagen.
3. Crea un sistema de ficheros y la capa R/W del contenedor.
4. Crea una interfaz de red bridge (normalmente `docker0`) y asigna una IP.
5. Ejecuta el proceso indicado.

### Comandos esenciales

```bash
# Ver contenedores en ejecución
docker ps
docker ps -a        # incluye los detenidos

# Ver logs
docker logs mi-nginx
docker logs -f mi-nginx   # -f: follow, en tiempo real

# Ejecutar un comando en un contenedor en marcha
docker exec -it mi-nginx bash

# Parar / arrancar / eliminar
docker stop mi-nginx
docker start mi-nginx
docker rm mi-nginx          # Eliminar (debe estar parado)
docker rm -f mi-nginx       # Forzar eliminación aunque esté corriendo

# Reconectarse a un contenedor en foreground
docker attach mi-nginx      # Cuidado: CTRL+C pararía el contenedor

# Variables de entorno
docker run -e NOMBRE=Juan ubuntu bash -c "echo Hola $NOMBRE"
docker run --env-file .env ubuntu bash

# Port-forwarding
docker run -p 8080:80 nginx   # host:contenedor

# Limpieza
docker system prune           # Elimina imágenes, contenedores y redes sin uso
docker volume prune           # Elimina volúmenes sin uso
```

---

## 6. Dockerfiles

Un Dockerfile es un **fichero de texto con instrucciones** que describe cómo construir una imagen Docker. Es reproducible: cualquiera con el Dockerfile obtiene exactamente la misma imagen.

### Instrucciones principales

```dockerfile
# Imagen base (obligatoria, primera instrucción)
FROM python:3.11-slim

# Variables de entorno (disponibles en construcción y ejecución)
ENV APP_HOME=/app \
    PORT=8080

# Directorio de trabajo (todas las instrucciones siguientes operan aquí)
WORKDIR /app

# Copiar ficheros del host al contenedor
COPY requirements.txt .
COPY src/ ./src/

# Ejecutar comandos durante la construcción (instalar deps, compilar...)
RUN pip install --no-cache-dir -r requirements.txt

# Documenta el puerto que usa la app (no lo publica realmente)
EXPOSE 8080

# Comando por defecto al arrancar el contenedor (sobreescribible)
CMD ["python", "src/main.py"]

# Proceso principal (no sobreescribible por argumentos al docker run)
ENTRYPOINT ["python", "src/main.py"]
```

### CMD vs ENTRYPOINT

| | `ENTRYPOINT` | `CMD` |
|---|---|---|
| **Rol** | Define el proceso principal, el "ejecutable" del contenedor | Define argumentos por defecto para ENTRYPOINT, o el comando si no hay ENTRYPOINT |
| **Se sobreescribe con** | `docker run --entrypoint` | Cualquier argumento pasado a `docker run imagen <aquí>` |
| **Uso típico** | Contenedores que actúan como binarios | Argumentos configurables |

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run imagen → ejecuta: python app.py
# docker run imagen test.py → ejecuta: python test.py (CMD sobreescrito)
```

### Construir una imagen

```bash
docker build -t mi-app:v1 .
# -t: nombre:tag de la imagen
# . : contexto de construcción (directorio actual)

docker build -t mi-app:v1 -f MiDockerfile .
# -f: especificar un fichero Dockerfile con nombre diferente

docker images   # Ver las imágenes locales
```

### Estrategias para imágenes eficientes

**1. Usar imágenes base ligeras:**
- `alpine` (~5 MB): mínima. Usa musl libc en vez de glibc — puede haber incompatibilidades.
- `slim` (~130 MB): basadas en Debian. Buen equilibrio.
- Imágenes completas cuando sea necesario.

**2. Minimizar el número de capas:** cada instrucción `RUN` añade una capa. Combinar comandos relacionados:

```dockerfile
# MAL: 3 capas
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# BIEN: 1 capa
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

**3. Aprovechar la caché de capas:** Docker cachea cada capa. Si una capa cambia, invalida todas las posteriores. Ordenar de **menos cambiante a más cambiante**:

```dockerfile
# BIEN: las dependencias cambian poco, el código cambia mucho
COPY requirements.txt .          # copia solo el pom/requirements primero
RUN pip install -r requirements.txt   # instala deps (cacheado si requirements.txt no cambió)
COPY src/ .                     # copia el código (siempre reconstruido si cambia)

# MAL: copiar todo primero invalida el install con cualquier cambio de código
COPY . .
RUN pip install -r requirements.txt
```

**4. Usar `.dockerignore`:** igual que `.gitignore` pero para el contexto de build. Excluye `node_modules`, `.git`, logs, etc. para no copiar ficheros innecesarios.

**5. Multi-stage builds:** usar una imagen pesada para compilar y una imagen ligera para el resultado final:

```dockerfile
# Etapa 1: compilación (imagen con JDK + Maven, pesada)
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /src
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -q
COPY src/ src/
RUN ./mvnw package -DskipTests -q

# Etapa 2: imagen final (solo JRE, ligera)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /src
COPY --from=builder /src/target/*.jar app.jar   # Copia SOLO el jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Resultado: la imagen final no contiene Maven, JDK, ni código fuente. Solo JRE + jar. Pasa de ~600 MB a ~200 MB.

### Subir una imagen a Docker Hub

```bash
docker login                              # Autenticarse
docker tag mi-app:v1 usuario/mi-app:v1   # Etiquetar con el nombre del usuario
docker push usuario/mi-app:v1            # Subir al registro
docker run usuario/mi-app:v1             # Cualquiera puede usarla
```

---

## 7. Docker Compose

### Qué es y para qué sirve

Docker Compose es una herramienta para **definir y ejecutar aplicaciones multi-contenedor**. En lugar de ejecutar múltiples `docker run` con flags complicados, describes todos los servicios, redes y volúmenes en un único fichero YAML y los arrancas con un solo comando.

**Ideal para:** desarrollo local, testing, CI/CD.
**No recomendado para:** producción a escala (para eso: Kubernetes).

### Estructura básica del `compose.yaml`

```yaml
services:
  frontend:
    image: nginx:latest
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - app-net
    environment:
      - API_URL=http://backend:5000

  backend:
    build: ./backend          # Construir la imagen desde un Dockerfile
    ports:
      - "5000:5000"
    depends_on:
      - database
    networks:
      - app-net
    volumes:
      - ./backend:/app        # Bind mount para desarrollo en caliente

  database:
    image: postgres:15-alpine
    networks:
      - app-net
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - db-data:/var/lib/postgresql/data   # Named volume para persistencia

networks:
  app-net:
    driver: bridge

volumes:
  db-data:
```

### Comandos básicos

```bash
docker compose up           # Arranca todos los servicios (foreground)
docker compose up -d        # En background
docker compose up --build   # Reconstruye imágenes antes de arrancar
docker compose down         # Para y elimina contenedores y redes
docker compose down -v      # También elimina volúmenes
docker compose stop         # Solo para (no elimina)
docker compose start        # Arranca (si ya habían sido creados)
docker compose ps           # Ver estado de los servicios
docker compose logs -f backend   # Logs de un servicio específico
docker compose restart frontend  # Reiniciar un servicio
```

### Gestión de dependencias y health checks

`depends_on` controla el **orden de arranque**, pero por defecto solo espera a que el contenedor se haya iniciado, no a que esté listo para servir peticiones. Esto puede causar que la aplicación arranque antes que la base de datos.

**Solución:** health checks con condición `service_healthy`:

```yaml
services:
  database:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s       # Frecuencia del check
      timeout: 5s         # Máximo tiempo para responder
      retries: 5          # Intentos antes de marcar como unhealthy

  api:
    image: my-api
    depends_on:
      database:
        condition: service_healthy   # Espera a que DB esté healthy
```

### Variables de entorno

```yaml
# Opción 1: directamente en el YAML
services:
  app:
    environment:
      - NODE_ENV=production
      - PORT=3000

# Opción 2: fichero .env (se carga automáticamente)
# .env:
# NODE_ENV=production
# PORT=3000
services:
  app:
    environment:
      - NODE_ENV=${NODE_ENV}
      - PORT=${PORT}

# Opción 3: fichero de variables específico
services:
  app:
    env_file:
      - ./config/.env.production

# Valores por defecto
services:
  app:
    environment:
      - USER=${USER:-admin}          # Si USER no existe, usa 'admin'
      - PASSWORD=${PASSWORD:-12345}
```

### Build Context

Cuando se necesita construir la imagen:

```yaml
services:
  web:
    build:
      context: .                    # Directorio del contexto de build
      dockerfile: Dockerfile.prod   # Fichero Dockerfile específico
    # O en forma abreviada:
    build: ./frontend   # Usa el Dockerfile de ./frontend
```

### Limitar recursos

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.50'    # 50% de una CPU
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

---

## 8. Redes en Docker

### Por qué existen las redes en Docker

Los contenedores están aislados por defecto. Para que se comuniquen entre sí (o con el exterior) necesitan redes.

Al instalar Docker, se crea automáticamente una interfaz bridge `docker0` en el host. Esta interfaz actúa de gateway entre los contenedores y el mundo exterior.

**Cada contenedor** recibe una dirección IP dentro de la red de Docker (típicamente `172.17.0.x`). El host es siempre `172.17.0.1`.

### Tipos de redes

| Tipo | Descripción | Uso típico |
|---|---|---|
| `bridge` | Red privada interna (por defecto) | Contenedores en el mismo host |
| `host` | El contenedor comparte la red del host directamente | Alto rendimiento, sin NAT |
| `none` | Sin red (aislado completamente) | Procesos que no necesitan red |
| `overlay` | Conecta múltiples hosts (Swarm/Kubernetes) | Clústeres multi-host |
| `macvlan` | El contenedor recibe una IP en la red física real | Integración en redes corporativas |

### Red bridge por defecto vs. redes personalizadas

La red bridge que Docker crea por defecto **no tiene DNS interno**: los contenedores no se pueden descubrir por nombre, solo por IP (que cambia en cada arranque).

**Solución:** crear una red personalizada. Las redes bridge de usuario sí tienen DNS interno:

```bash
docker network create mi-red
docker run -d --name web --network mi-red nginx
docker run -d --name db --network mi-red postgres

# Dentro de 'web', se puede acceder a 'db' por nombre:
# psql -h db -U user   ← el nombre 'db' funciona como hostname
```

### Comandos de redes

```bash
docker network ls                      # Listar redes
docker network create mi-red           # Crear red
docker network inspect mi-red          # Detalles de la red
docker network rm mi-red               # Eliminar red
docker network connect mi-red contenedor    # Conectar contenedor a red
docker inspect contenedor              # Ver redes del contenedor (entre otras cosas)
```

### Redes en Docker Compose

Compose crea automáticamente una red para todos los servicios del fichero. Los servicios se descubren por su nombre de servicio:

```yaml
services:
  web:
    image: nginx
  api:
    image: node:lts
    # Puede acceder a 'web' simplemente con: http://web:80
```

**Redes personalizadas** para segmentar qué puede hablar con qué:

```yaml
services:
  frontend:
    networks:
      - frontend-net
      - backend-net
  api:
    networks:
      - backend-net
      - db-net
  database:
    networks:
      - db-net    # Solo accesible desde api

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
  db-net:
    driver: bridge
    internal: true   # Sin acceso desde fuera del clúster Compose
```

---

## 9. Volúmenes en Docker

### El problema: los contenedores son efímeros

Cuando un contenedor se elimina, **todo lo que se escribió en su sistema de ficheros se pierde**. Esto es intencional: los contenedores son unidades inmutables y reproducibles. Pero hay datos que necesitan sobrevivir (bases de datos, uploads de usuarios, logs…).

**Solución:** volúmenes. Almacenamiento que existe fuera del ciclo de vida del contenedor.

### Tipos de volúmenes

| Tipo | Persistencia | Quién lo gestiona | Uso típico |
|---|---|---|---|
| **Named Volume** | Sí | Docker | Bases de datos en producción |
| **Bind Mount** | Sí | El usuario | Desarrollo (código en caliente) |
| **tmpfs** | No (vive en RAM) | Kernel | Datos temporales o secretos efímeros |
| **Anonymous** | Sí | Docker (sin nombre) | Datos temporales de imágenes |

### Named Volumes (volúmenes con nombre)

Docker los gestiona completamente. No importa dónde estén físicamente en el host.

```bash
docker volume create mi-datos
docker run -v mi-datos:/data nginx           # Montar en /data
docker run --mount type=volume,src=mi-datos,dst=/data nginx  # Sintaxis explícita
docker volume ls
docker volume inspect mi-datos
docker volume rm mi-datos
```

### Bind Mounts

Montan una carpeta o fichero **real del host** dentro del contenedor. Los cambios son bidireccionales: el contenedor ve los cambios del host y viceversa.

```bash
docker run -v /home/user/app:/app node    # ruta_host:ruta_contenedor
docker run -v $(pwd):/app node            # Directorio actual
docker run -v $(pwd):/app:ro node         # Solo lectura (ro)
```

**Ideal para desarrollo:** editar el código en el host y que el contenedor lo vea inmediatamente, sin reconstruir la imagen.

**Riesgo:** si se cambia la estructura de directorios del host, el contenedor puede fallar. No usar en producción.

### tmpfs

```bash
docker run --tmpfs /app/cache myimage
```

Los datos van a RAM, no a disco. Se pierden al parar el contenedor. Útil para caché temporal o datos sensibles que no deben escribirse en disco.

### Volúmenes en Docker Compose

```yaml
services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data    # Named volume (producción)

  app:
    image: node
    volumes:
      - .:/app                  # Bind mount: código del host (desarrollo)
      - /app/node_modules       # Excluir node_modules del bind mount

  web:
    image: nginx
    volumes:
      - ./config:/etc/nginx/conf.d:ro    # Solo lectura

volumes:
  db-data:    # Declarar el named volume
```

**Regla práctica:**
- **Named volumes** para datos de producción (base de datos, uploads).
- **Bind mounts** para código fuente en desarrollo.
- **`--mount` sobre `-v`** cuando se necesitan opciones avanzadas (más explícito y legible).

---

# Parte 2 — Kubernetes

## 10. Qué es Kubernetes y por qué

### El problema que resuelve

Con unos pocos contenedores Docker Compose funciona bien. Pero ¿qué pasa con decenas, cientos o miles de contenedores distribuidos en múltiples servidores?

- ¿Cómo decides en qué servidor va cada contenedor?
- ¿Qué pasa si un servidor falla?
- ¿Cómo balanceas la carga entre contenedores?
- ¿Cómo actualizas la aplicación sin downtime?
- ¿Cómo escala automáticamente según la demanda?

**Kubernetes resuelve todo esto.**

### Qué es Kubernetes

Kubernetes (K8s — 8 letras entre la K y la s) es una **plataforma de orquestación de contenedores**. Automatiza el despliegue, el escalado y la gestión de aplicaciones en contenedores sobre un clúster de servidores.

- Desarrollado por Google en 2014, basado en su sistema interno "Borg" (2 billones de contenedores/semana).
- Cedido a la CNCF (Cloud Native Computing Foundation) como primer proyecto open source.
- Estándar de facto en la industria: Google, Netflix, Spotify, Airbnb...
- Puede usar Docker/containerd/CRI-O como runtime de contenedores.

### Capacidades clave

- **Autorrecuperación:** si un pod falla, Kubernetes lo reinicia automáticamente. Si un nodo falla, mueve los pods a otro nodo.
- **Escalado automático:** ajusta el número de réplicas según la carga de CPU/memoria.
- **Despliegues sin downtime:** actualiza la aplicación gradualmente (rolling update). Si algo falla, hace rollback.
- **Balanceo de carga:** distribuye el tráfico entre los pods automáticamente.
- **Descubrimiento de servicios:** los pods se encuentran entre sí por nombre DNS interno.
- **Gestión de configuración y secretos:** ConfigMaps y Secrets.
- **Declarativo:** defines el estado deseado. Kubernetes se encarga de alcanzarlo y mantenerlo.

---

## 11. Arquitectura de Kubernetes

Un clúster Kubernetes tiene dos tipos de nodos:

```
┌─────────────────────────────────────────────────────────────────┐
│  CONTROL PLANE (Master)                                          │
│  ┌─────────────┐  ┌───────────────┐  ┌──────────────────────┐  │
│  │  kube-api   │  │     etcd      │  │  Controller Manager  │  │
│  │  server     │  │  (estado del  │  │  (mantiene el estado │  │
│  │  (REST API) │  │   clúster)    │  │   deseado)           │  │
│  └─────────────┘  └───────────────┘  └──────────────────────┘  │
│  ┌─────────────┐                                                 │
│  │  Scheduler  │  (decide en qué nodo va cada pod)              │
│  └─────────────┘                                                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
         ┌─────────────────────┼──────────────────┐
         │                     │                  │
┌────────┴──────┐   ┌──────────┴──────┐   ┌──────┴──────────┐
│  Worker Node  │   │  Worker Node    │   │  Worker Node    │
│  ┌─────────┐  │   │  ┌─────────┐   │   │  ┌─────────┐   │
│  │  kubelet│  │   │  │ kubelet │   │   │  │ kubelet │   │
│  │kube-prxy│  │   │  │kube-prxy│   │   │  │kube-prxy│   │
│  └─────────┘  │   │  └─────────┘   │   │  └─────────┘   │
│  [Pod][Pod]   │   │  [Pod][Pod]    │   │  [Pod][Pod]    │
└───────────────┘   └───────────────-┘   └────────────────┘
```

### Control Plane

El "cerebro" del clúster. Puede ejecutarse en un nodo único o en varios (alta disponibilidad):

- **kube-apiserver:** punto de entrada de todo. Expone la API REST de Kubernetes. Todos los componentes (kubectl, otros componentes internos) se comunican a través de él. Toda modificación del clúster pasa por aquí y se persiste en etcd.
- **etcd:** base de datos clave-valor distribuida. Almacena **todo el estado del clúster**: qué recursos existen, cuál es su configuración, qué nodos hay. Si se pierde etcd, se pierde el clúster.
- **Scheduler:** decide en qué nodo Worker se ejecutará cada Pod nuevo, según recursos disponibles, afinidades, taints/tolerations, etc.
- **Controller Manager:** ejecuta los controllers que mantienen el estado deseado del clúster. Por ejemplo: si un pod muere y el Deployment dice que deben existir 3 replicas, el ReplicaSet Controller crea uno nuevo.

### Worker Nodes

Máquinas (físicas o virtuales) donde se ejecutan las aplicaciones:

- **kubelet:** agente que corre en cada nodo. Recibe instrucciones del API server y asegura que los pods estén en el estado correcto. Comunica el estado del nodo al Control Plane.
- **kube-proxy:** gestiona las reglas de red del nodo (iptables/ipvs) para implementar los Services. Es el responsable de que el tráfico llegue al pod correcto.
- **Container runtime:** el que realmente ejecuta los contenedores. Puede ser containerd, CRI-O o cualquier implementación del estándar CRI (Container Runtime Interface).

---

## 12. Objetos fundamentales: Pod, Deployment, Service

### Pod

El **Pod es la unidad mínima de ejecución en Kubernetes**. Un pod agrupa uno o más contenedores que deben ejecutarse juntos en el mismo nodo.

**Características:**
- Todos los contenedores de un pod comparten la misma red (misma IP, mismos puertos) y pueden compartir volúmenes.
- Cada pod tiene una IP única dentro del clúster (no accesible desde fuera sin un Service).
- Los pods son **efímeros**: cuando un pod muere, su contenido se pierde (como los contenedores Docker). No se "recupera" el mismo pod; Kubernetes crea uno nuevo.
- **Caso habitual:** un pod = un contenedor. Los pods multi-contenedor son para sidecars (logging, proxies) que deben vivir junto a la aplicación principal.

```bash
# Crear un pod directamente (poco habitual en producción)
kubectl run mi-pod --image=nginx:latest

# Ver pods
kubectl get pods
kubectl get pods -o wide    # Muestra nodo, IP, etc.

# Ver logs
kubectl logs mi-pod

# Acceder a un pod
kubectl exec -it mi-pod -- bash

# Port-forwarding para acceder desde local
kubectl port-forward pod/mi-pod 8080:80

# Eliminar
kubectl delete pod mi-pod
```

### Deployment

Crear pods directamente tiene un problema: si el pod falla, nadie lo vuelve a crear. **El Deployment es el recurso que garantiza que siempre haya un número determinado de réplicas corriendo**.

**Qué hace un Deployment:**
1. Crea un `ReplicaSet` que gestiona las réplicas.
2. El ReplicaSet crea y mantiene los pods.
3. Si un pod falla, el ReplicaSet crea uno nuevo automáticamente.
4. Gestiona las actualizaciones (rolling update) y los rollbacks.

```bash
# Crear un deployment
kubectl create deployment mi-app --image=nginx:1.25 --replicas=3

# Ver deployments
kubectl get deployments

# Escalar
kubectl scale deployment mi-app --replicas=5

# Port-forwarding al deployment
kubectl port-forward deployment/mi-app 8080:80

# Eliminar
kubectl delete deployment mi-app
```

### Service

Los pods tienen IPs que **cambian cada vez que se recrean**. Un Service es una **abstracción de red estable** que expone un conjunto de pods con una IP y un nombre DNS fijos.

**Funcionamiento:** el Service usa **etiquetas (labels) y selectores** para saber a qué pods debe redirigir el tráfico. Los pods que tienen el label `app: mi-app` son seleccionados por el Service con `selector: app: mi-app`. El Service balancea automáticamente el tráfico entre todos los pods seleccionados.

**Tipos de Service:**

| Tipo | Acceso | Uso |
|---|---|---|
| `ClusterIP` (por defecto) | Solo desde dentro del clúster | Comunicación interna entre servicios |
| `NodePort` | Desde fuera: `<IP-nodo>:<puerto>` (30000-32767) | Desarrollo/testing, acceso externo básico |
| `LoadBalancer` | IP pública externa asignada por el proveedor cloud | Producción en cloud |
| `ExternalName` | Alias DNS de un servicio externo | Conectar con recursos fuera del clúster |

```bash
# Exponer un deployment
kubectl expose deployment mi-app --type=NodePort --port=80

# Ver services
kubectl get services
kubectl get svc   # Versión corta

# Ver endpoints (IPs y puertos reales de los pods)
kubectl get endpointslice

# Eliminar service
kubectl delete service mi-app
```

### Etiquetas y selectores

Las etiquetas son pares clave-valor que se añaden a cualquier objeto Kubernetes. Los selectores filtran objetos por etiquetas. Esta es la forma en que los Services encuentran sus pods y los Deployments gestionan sus pods.

```bash
kubectl get pods --show-labels
kubectl get pods -l app=mi-app          # Filtrar por etiqueta
kubectl run mi-pod --image=nginx --labels="app=frontend,version=v2"
```

---

## 13. Despliegue con YAML

En producción no se usan comandos imperativos sino **ficheros YAML declarativos**. Ventajas: versionado en Git, reproducibilidad, integración con CI/CD.

**Todo objeto Kubernetes en YAML tiene 4 campos obligatorios:**
- `apiVersion`: versión de la API (`apps/v1`, `v1`, etc.)
- `kind`: tipo de recurso (`Deployment`, `Service`, `Pod`, etc.)
- `metadata`: nombre, namespace, etiquetas
- `spec`: la especificación del recurso (varía según el tipo)

### Deployment en YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx          # El deployment gestiona pods con este label
  template:               # Plantilla para crear los pods
    metadata:
      labels:
        app: nginx        # Label que se añade a los pods creados
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### Service en YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: nginx            # Selecciona pods con este label
  ports:
    - protocol: TCP
      port: 80            # Puerto del Service dentro del clúster
      targetPort: 80      # Puerto del contenedor
      nodePort: 30080     # Puerto externo en el nodo (30000-32767)
```

### Comandos con YAML

```bash
kubectl apply -f deployment.yaml         # Crear o actualizar (declarativo)
kubectl apply -f .                       # Aplicar todos los YAML del directorio
kubectl create -f deployment.yaml        # Solo crear (falla si existe)

kubectl get all                          # Ver todos los recursos
kubectl describe deployment web-deployment   # Detalle de un recurso
kubectl delete -f deployment.yaml        # Eliminar lo definido en el YAML

# Trucos útiles
kubectl run bb --image=busybox --dry-run=client -o yaml   # Generar YAML sin crear el recurso
kubectl get pod mi-pod -o yaml                             # Obtener el YAML de un recurso existente

# Múltiples recursos en un fichero (separados con ---)
# El mismo fichero puede tener Deployment + Service si se separan con ---
```

---

## 14. ConfigMaps y Secrets

### El problema del hardcoding

Poner configuración o credenciales directamente en el YAML del Deployment es peligroso:

```yaml
env:
  - name: DB_PASSWORD
    value: password123    # ¡Esto no debe ir en Git!
```

ConfigMaps y Secrets **separan la configuración del código de despliegue**.

### ConfigMap — configuración no sensible

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "Mi Aplicación"
  LOG_LEVEL: "info"
  API_URL: "https://api.ejemplo.com"
  APP_VERSION: "1.0"
```

### Secret — datos sensibles

Los Secrets almacenan los valores en **Base64** (no cifrado, solo codificado). Para cifrado real se necesita solución adicional (Vault, Sealed Secrets).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=     # "password123" en base64
  API_KEY: bWlfa2xhdmVfc2VjcmV0YQ==
```

Para codificar: `echo -n "password123" | base64`

### Usar ConfigMap y Secret en un Deployment

```yaml
containers:
  - name: app
    image: mi-app:1.0
    env:
      # Desde ConfigMap
      - name: APP_NAME
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: APP_NAME
      - name: LOG_LEVEL
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: LOG_LEVEL
      # Desde Secret
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: app-secrets
            key: DB_PASSWORD
```

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml

kubectl get configmap app-config
kubectl get secret app-secrets
kubectl exec -it <pod> -- env    # Verificar que las variables llegan
```

---

## 15. Volúmenes persistentes en Kubernetes

### El problema: pods efímeros y datos persistentes

Al igual que con Docker, los datos escritos en el sistema de ficheros de un pod se pierden cuando el pod muere. Kubernetes separa el **almacenamiento** de la **computación** mediante volúmenes.

### Tipos de volúmenes

- **`emptyDir`:** volumen temporal que se crea cuando el pod arranca y se elimina cuando el pod muere. Útil para compartir datos entre contenedores del **mismo pod**.
- **`hostPath`:** monta un directorio del nodo host. Persiste mientras el nodo exista, pero no es portable (no funciona si el pod migra a otro nodo). Solo para casos especiales (DaemonSets).
- **`ConfigMap`/`Secret` como volumen:** monta la configuración como ficheros dentro del contenedor.
- **PersistentVolumes (PV) + PersistentVolumeClaims (PVC):** la solución recomendada para producción.

### PersistentVolume (PV) y PersistentVolumeClaim (PVC)

Kubernetes separa dos responsabilidades:

- **PV (PersistentVolume):** el recurso de almacenamiento real en el clúster. Lo provisiona el administrador (manual) o Kubernetes automáticamente (dinámico con StorageClass). Tiene ciclo de vida independiente de los pods.
- **PVC (PersistentVolumeClaim):** la solicitud de almacenamiento por parte de un pod. Especifica cuánto necesita y con qué modo de acceso. Kubernetes vincula (binding) el PVC con un PV que cumpla los requisitos.

**Flujo:**
```
Administrador crea PV  →  Usuario crea PVC  →  Kubernetes hace binding  →  Pod usa PVC
```
O con StorageClass (aprovisionamiento dinámico):
```
Usuario crea PVC  →  Kubernetes crea PV automáticamente  →  Pod usa PVC
```

### StorageClass

Plantilla que define cómo crear volúmenes dinámicamente. Cada proveedor de cloud tiene su propia StorageClass. En local, Docker Desktop y Minikube incluyen una por defecto.

### Modos de acceso

| Modo | Abreviatura | Descripción |
|---|---|---|
| ReadWriteOnce | RWO | Un solo nodo puede montar el volumen en R/W |
| ReadOnlyMany | ROX | Múltiples nodos pueden montar en solo lectura |
| ReadWriteMany | RWX | Múltiples nodos pueden montar en R/W (requiere filesystem distribuido) |
| ReadWriteOncePod | RWOP | Un solo pod puede montar en R/W (K8s 1.22+) |

### Políticas de recuperación del PV

¿Qué pasa con el PV cuando se elimina el PVC?

- **`Retain`:** el PV permanece con los datos. Requiere limpieza manual. Más seguro para datos críticos.
- **`Delete`:** el PV se elimina automáticamente. Por defecto en cloud. Los datos se pierden.

### Ejemplo — aprovisionamiento manual

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mi-pv
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data       # En local; en cloud sería un disco real
---
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mi-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: manual
```

### Usar el PVC en un Deployment

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html    # Ruta dentro del contenedor
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: mi-pvc                     # Referencia al PVC
```

### Aprovisionamiento dinámico (más habitual)

```yaml
# Solo se necesita el PVC; el PV se crea automáticamente
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mi-pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # No se especifica storageClassName → usa la por defecto
```

```bash
kubectl get pv
kubectl get pvc
kubectl describe pvc mi-pvc
kubectl get storageclass
```

---

## 16. Health Checks (Probes)

Kubernetes necesita saber el estado real de cada contenedor para tomar decisiones (reiniciarlo, sacarlo del balanceo de carga, etc.). Para esto usa **probes (sondas)**.

### Tipos de probes

**Liveness Probe — "¿Está vivo el contenedor?"**
Si falla, Kubernetes **reinicia** el contenedor. Se usa para detectar deadlocks o estados corruptos de los que la aplicación no puede recuperarse sola.

**Readiness Probe — "¿Está listo para recibir tráfico?"**
Si falla, Kubernetes **quita el pod del Service** (no le llegan peticiones) pero no lo reinicia. Se usa para esperar a que la aplicación termine de inicializar o para sacarla del balanceo si está temporalmente sobrecargada.

**Startup Probe — "¿Ha terminado de arrancar?"**
Para aplicaciones con arranque lento. Mientras el startup probe no pase, Kubernetes **ignora** las probes de liveness y readiness. Evita que Kubernetes reinicie prematuramente una app que tarda en arrancar.

### Métodos de verificación

```yaml
# HTTP GET: el contenedor debe responder con 2xx o 3xx
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080

# TCP Socket: debe poder establecerse una conexión TCP
livenessProbe:
  tcpSocket:
    port: 3306

# Comando: se ejecuta dentro del contenedor; si devuelve 0 = sano
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

### Parámetros de configuración

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10   # Espera antes del primer check (tiempo de arranque)
  periodSeconds: 5          # Frecuencia de los checks
  timeoutSeconds: 2         # Timeout por intento
  successThreshold: 1       # Éxitos consecutivos para considerarse sano
  failureThreshold: 3       # Fallos consecutivos para considerarse muerto
```

### Ejemplo completo con los tres tipos

```yaml
containers:
  - name: app
    image: mi-app:1.0
    startupProbe:
      httpGet:
        path: /ready
        port: 8080
      failureThreshold: 10   # 10 * 5s = 50s máximo para arrancar
      periodSeconds: 5

    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 3    # Tolerar 3 fallos antes de reiniciar

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 3       # Checks más frecuentes
      failureThreshold: 1    # Sacar del balanceo rápidamente
```

**Principios guía:**
- **Liveness:** intervalos más largos y más tolerante a fallos. Un reinicio innecesario es peor que esperar.
- **Readiness:** intervalos más cortos y menos tolerante. Mejor sacar un pod del balanceo antes de que afecte a usuarios.
- `kubectl logs <pod> --previous` — para ver los logs de la instancia anterior (antes del reinicio).

---

## 17. Escalado de aplicaciones

### Tipos de escalado

- **Escalado horizontal (out/in):** añadir/quitar pods. La opción habitual en Kubernetes.
- **Escalado vertical (up/down):** pods más grandes (más CPU/RAM). Más limitado, requiere reiniciar los pods.

### Escalado manual

```bash
kubectl scale deployment mi-app --replicas=5
```

O en el YAML: `spec.replicas: 5` y `kubectl apply -f deployment.yaml`.

**Cuándo usar escalado manual:** eventos planificados con picos de tráfico conocidos, pruebas de carga, emergencias puntuales.

### Horizontal Pod Autoscaler (HPA)

El HPA escala **automáticamente** el número de réplicas de un Deployment según métricas (CPU, memoria, métricas custom).

**Cómo funciona:**
- Cada 15 segundos consulta las métricas actuales.
- Calcula las réplicas necesarias: `replicas_deseadas = ceil(replicas_actuales × (métrica_actual / métrica_objetivo))`
- Ejemplo: 2 pods, CPU al 80%, objetivo 50% → `ceil(2 × 80/50)` = `ceil(3.2)` = **4 pods**

**Requisitos:** el Metrics Server debe estar instalado en el clúster, y los pods deben tener definidos `resources.requests`.

```yaml
# Deployment con recursos definidos (obligatorio para HPA)
containers:
  - name: app
    image: mi-app:1.0
    resources:
      requests:
        cpu: "200m"      # 200 millicores = 20% de 1 CPU
      limits:
        cpu: "500m"      # Máximo 50% de 1 CPU
```

```yaml
# HPA en YAML
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mi-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mi-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50    # Objetivo: CPU < 50%
```

```bash
# Crear HPA desde línea de comandos
kubectl autoscale deployment mi-app --cpu-percent=50 --min=1 --max=10

kubectl get hpa                     # Ver HPAs
kubectl get hpa mi-app-hpa --watch  # Monitorizar en tiempo real
kubectl top pods                    # Ver uso de recursos actual
kubectl top nodes
```

---

## 18. Actualizaciones y Rollbacks

### El problema: actualizar sin downtime

En el enfoque tradicional: parar todo → actualizar → reiniciar. Resultado: minutos de downtime.

Kubernetes lo resuelve con **Rolling Updates**: actualiza los pods gradualmente, siempre manteniendo pods disponibles.

### Rolling Update — cómo funciona

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Pods extra permitidos durante la actualización
      maxUnavailable: 0   # Pods no disponibles permitidos durante la actualización
```

**Ejemplo con `maxSurge: 1, maxUnavailable: 0`, 3 réplicas:**

```
Estado inicial:     [v1] [v1] [v1]
Paso 1 (crea v2):   [v1] [v1] [v1] [v2]   ← maxSurge: 1 pod extra
Paso 2 (elimina v1):[v1] [v1] [v2]
Paso 3 (crea v2):   [v1] [v1] [v2] [v2]
Paso 4 (elimina v1):[v1] [v2] [v2]
...hasta que todos son v2
```

Con `maxUnavailable: 0` garantizamos que **siempre hay 3 pods disponibles**. Los health checks se usan para decidir cuándo el pod nuevo está listo antes de eliminar uno viejo.

### Comandos de actualización

```bash
# Actualizar la imagen
kubectl set image deployment/mi-app app=mi-app:2.0

# O aplicar el YAML con la nueva imagen:
kubectl apply -f deployment-v2.yaml

# Monitorizar el estado del rollout
kubectl rollout status deployment/mi-app

# Ver historial de cambios
kubectl rollout history deployment/mi-app
kubectl rollout history deployment/mi-app --revision=2   # Detalle de una revisión

# Pausar y reanudar (para pruebas graduales)
kubectl rollout pause deployment/mi-app
kubectl rollout resume deployment/mi-app
```

### Rollback — volver atrás

Si algo sale mal, Kubernetes guarda el ReplicaSet anterior (con 0 réplicas) para poder hacer rollback rápidamente:

```bash
# Ver el historial
kubectl rollout history deployment/mi-app

# Volver a la versión anterior
kubectl rollout undo deployment/mi-app

# Volver a una versión específica
kubectl rollout undo deployment/mi-app --to-revision=1
```

**Cómo funciona el rollback:** Kubernetes simplemente escala el ReplicaSet anterior de vuelta a las réplicas deseadas y escala el actual a 0. No hay nada "especial" — es el mismo mecanismo de rolling update pero en sentido inverso.

```bash
# Ver los ReplicaSets guardados
kubectl get rs
# Verás algo como:
# web-deployment-abc123   3         3         3       (actual)
# web-deployment-def456   0         0         0       (versión anterior, lista para rollback)
```

### Tabla resumen de comandos kubectl

| Acción | Comando |
|---|---|
| Ver nodos | `kubectl get nodes` |
| Ver pods | `kubectl get pods [-o wide]` |
| Ver deployments | `kubectl get deployments` |
| Ver services | `kubectl get svc` |
| Ver todo | `kubectl get all` |
| Ver detalles | `kubectl describe <tipo> <nombre>` |
| Ver logs | `kubectl logs <pod> [-f] [--previous]` |
| Entrar a un pod | `kubectl exec -it <pod> -- bash` |
| Aplicar YAML | `kubectl apply -f archivo.yaml` |
| Eliminar por YAML | `kubectl delete -f archivo.yaml` |
| Escalar | `kubectl scale deployment <name> --replicas=N` |
| Estado del rollout | `kubectl rollout status deployment/<name>` |
| Historial rollout | `kubectl rollout history deployment/<name>` |
| Rollback | `kubectl rollout undo deployment/<name>` |
| Port-forward | `kubectl port-forward pod/<name> 8080:80` |
| Ver métricas | `kubectl top pods` / `kubectl top nodes` |
| Recursos del API | `kubectl api-resources` |
| Generar YAML | `kubectl run x --image=y --dry-run=client -o yaml` |
