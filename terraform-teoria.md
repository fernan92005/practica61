# Terraform — Apuntes Teóricos Completos
> Infraestructuras y Procesos de Soporte · UMA · 2025/2026

---

## Índice

1. [Infrastructure as Code (IaC)](#1-infrastructure-as-code-iac)
2. [Qué es Terraform](#2-qué-es-terraform)
3. [Arquitectura de Terraform](#3-arquitectura-de-terraform)
4. [HCL — El lenguaje de Terraform](#4-hcl--el-lenguaje-de-terraform)
5. [Bloques principales](#5-bloques-principales)
6. [Providers](#6-providers)
7. [Resources](#7-resources)
8. [Variables de entrada](#8-variables-de-entrada)
9. [Local Values](#9-local-values)
10. [Output Values](#10-output-values)
11. [Tipos de datos](#11-tipos-de-datos)
12. [Expresiones y funciones](#12-expresiones-y-funciones)
13. [Data Sources](#13-data-sources)
14. [Estado (terraform.tfstate)](#14-estado-terraformtfstate)
15. [Ciclo de vida de los recursos](#15-ciclo-de-vida-de-los-recursos)
16. [Módulos](#16-módulos)
17. [Providers locales](#17-providers-locales)
18. [Comandos esenciales](#18-comandos-esenciales)
19. [Buenas prácticas y recomendaciones](#19-buenas-prácticas-y-recomendaciones)

---

## 1. Infrastructure as Code (IaC)

**Infrastructure as Code** es la práctica de gestionar y aprovisionar infraestructura mediante código legible por máquinas, en lugar de procesos manuales o configuraciones interactivas.

El enfoque tradicional (hacer clic en consolas web, ejecutar scripts ad-hoc) tiene un problema fundamental: no es reproducible ni auditable. IaC soluciona esto tratando la infraestructura igual que el código de una aplicación.

**Beneficios:**

- **Reproducibilidad:** la misma configuración produce exactamente el mismo resultado en cualquier entorno. No hay "funciona en mi máquina".
- **Versionado:** los ficheros `.tf` se guardan en Git. Se puede ver quién cambió qué, cuándo y por qué.
- **Automatización:** elimina los errores humanos de los procesos manuales. Se puede integrar en pipelines CI/CD.
- **Documentación:** el código es la documentación real del estado de la infraestructura. No hay documentos desactualizados.
- **Velocidad:** desplegar un entorno nuevo pasa de días a minutos.

---

## 2. Qué es Terraform

Terraform es la herramienta de IaC de HashiCorp. Permite **definir, crear y gestionar infraestructura de forma declarativa**: describes el estado final que quieres (qué recursos deben existir y cómo deben estar configurados), y Terraform calcula qué acciones ejecutar para llegar a ese estado.

**Características clave:**

- **Declarativo vs. imperativo:** no describes los pasos ("crea esto, luego aquello"), sino el resultado deseado. Terraform decide el orden y las acciones.
- **Multi-provider:** un único lenguaje para gestionar Docker, Kubernetes, AWS, Azure, GCP, ficheros locales, y miles de servicios más. Hay miles de providers y módulos disponibles.
- **Plan antes de actuar:** `terraform plan` muestra exactamente qué va a hacer antes de hacer nada. Esto evita sorpresas y permite controlar los costes en cloud.
- **Gestión de bajo y alto nivel:** desde máquinas virtuales y redes hasta registros DNS, certificados TLS o configuraciones de SaaS.
- **Extensible:** providers oficiales de HashiCorp, providers verificados por terceros, providers de la comunidad, y puedes escribir los tuyos.

**Versiones importantes:**

- **Terraform Community Edition:** versión gratuita (la que se usa en clase). No es 100% open source.
- **OpenTofu:** fork open source de Terraform creado en 2023 cuando HashiCorp cambió la licencia. Compatible con la mayoría del ecosistema de Terraform. `opentofu.org`
- **Terraform Cloud:** plataforma gestionada de HashiCorp con colaboración, estado remoto y automatización.

---

## 3. Arquitectura de Terraform

Terraform tiene tres capas que trabajan juntas:

```
┌─────────────────────────────────────┐
│   Configuración (.tf files)         │
│   - Variables                       │
│   - Resources                       │
│   - Outputs                         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Terraform Core                    │
│   - Parser HCL                      │
│   - Dependency Graph                │
│   - State Management                │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Providers (plugins)               │
│   - Local, Docker, AWS, GCP...      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Infraestructura real              │
│   - Ficheros, Contenedores, VMs...  │
└─────────────────────────────────────┘
```

**Los tres conceptos clave de la arquitectura:**

- **Providers:** plugins que saben hablar con un sistema concreto (Docker, AWS, el sistema de ficheros local…). Cada provider expone tipos de recursos y data sources.
- **Resources:** los elementos de infraestructura que Terraform crea, actualiza o destruye (un fichero, un contenedor, una VM, una red…).
- **State:** el fichero JSON donde Terraform registra qué recursos ha creado y cuál es su estado actual. Sin él, Terraform no sabe qué existe.

**Grafo de dependencias:** Terraform construye internamente un grafo dirigido acíclico (DAG) con todos los recursos y sus dependencias. Esto le permite crear recursos en paralelo cuando no dependen entre sí, y en orden cuando sí lo hacen.

---

## 4. HCL — El lenguaje de Terraform

**HCL (HashiCorp Configuration Language)** es el lenguaje en el que se escriben los ficheros `.tf`. Es un lenguaje declarativo diseñado específicamente para describir infraestructura.

**Filosofía de diseño:**
- Claridad antes que concisión — es más importante que sea legible que corto.
- Facilita la lectura humana.
- Declarativo: describes *qué* quieres obtener, no *cómo* conseguirlo.
- Orientado a reproducibilidad.

**Sintaxis básica:** todo en HCL se organiza en **bloques**. Cada bloque tiene un tipo, opcionalmente etiquetas identificadoras, y un cuerpo con atributos entre llaves `{}`.

```hcl
# Comentario de una línea

/*
  Comentario
  multilínea
*/

# Plantilla genérica de bloque:
<TIPO_BLOQUE> "<ETIQUETA1>" "<ETIQUETA2>" {
  atributo1 = "valor"
  atributo2 = 123

  bloque_anidado {
    opcion = true
  }
}
```

**Ejemplo real:**

```hcl
resource "local_file" "readme" {
  filename = "${path.module}/output/README.md"
  content  = "# Mi proyecto\n"
}
```

Aquí `resource` es el tipo de bloque, `"local_file"` es el tipo de recurso (viene del provider), y `"readme"` es el nombre local con el que referenciaremos este recurso en el resto de la configuración.

---

## 5. Bloques principales

Terraform reconoce varios tipos de bloque con significados específicos:

### `terraform` — configuración global

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}
```

Declara qué providers necesita el proyecto y con qué versiones. `terraform init` descarga automáticamente los providers declarados aquí.

### `provider` — configuración del provider

```hcl
provider "docker" {
  host = "unix:///var/run/docker.sock"
}
```

Algunos providers necesitan configuración adicional (credenciales, región, URL de API). El provider `local` no necesita bloque `provider` porque no tiene configuración.

### `resource` — recursos de infraestructura

```hcl
resource "local_file" "config" {
  filename = "${path.module}/config.txt"
  content  = "clave=valor"
}
```

El bloque más importante. Define un elemento de infraestructura que Terraform gestionará.

### `variable` — variables de entrada

```hcl
variable "project_name" {
  type    = string
  default = "mi-proyecto"
}
```

### `locals` — valores calculados internos

```hcl
locals {
  ruta_completa = "${path.module}/output/${var.project_name}"
}
```

### `output` — valores de salida

```hcl
output "ruta_fichero" {
  value = local_file.config.filename
}
```

### `data` — data sources (consulta de recursos existentes)

```hcl
data "local_file" "existente" {
  filename = "config.txt"
}
```

### `module` — llamada a un módulo

```hcl
module "mi_modulo" {
  source = "./modules/proyecto"
  nombre = "api"
}
```

---

## 6. Providers

Los providers son **plugins** que permiten a Terraform interactuar con diferentes plataformas y servicios. Son la "traducción" entre el código HCL y la API real de cada sistema.

Cada provider aporta dos cosas al lenguaje HCL:

1. **Tipos de recursos (`resource`):** elementos que Terraform puede crear, modificar o destruir. Ejemplos: `local_file`, `docker_container`, `aws_instance`.
2. **Data sources (`data`):** información que Terraform puede consultar pero no gestiona directamente. Ejemplos: una imagen Docker existente, una red existente.

**Cómo se declara un provider:**

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"   # Dónde descargarlo (Terraform Registry)
      version = "~> 2.0"            # Rango de versiones aceptadas
    }
  }
}
```

**Operadores de versión:**

| Operador | Significado |
|---|---|
| `"= 2.4.0"` | Exactamente esa versión |
| `">= 2.0"` | Cualquier versión 2.0 o superior |
| `"~> 2.0"` | `>= 2.0` y `< 3.0` (minor libre, major fijo) |
| `">= 2.0, < 3.0"` | Rango explícito |

> `"~> 2.0"` es el operador más habitual: permite actualizaciones de parche y minor pero protege contra cambios de major que podrían romper la compatibilidad.

**Dónde encontrar providers:** `registry.terraform.io`. Hay providers oficiales (HashiCorp), verificados (terceros) y de la comunidad.

> **Regla importante:** siempre revisar la documentación de la versión exacta del provider que estés usando. Las APIs cambian entre versiones y algunos atributos pueden desaparecer o comportarse diferente.

---

## 7. Resources

Los resources son los componentes de infraestructura que Terraform gestiona. Representan objetos reales: un fichero, un contenedor Docker, una máquina virtual, una red, una base de datos.

**Sintaxis:**

```hcl
resource "<tipo_provider>_<tipo_recurso>" "<nombre_local>" {
  # atributos de configuración
}
```

**Ejemplo:**

```hcl
resource "local_file" "readme" {
  filename = "${path.module}/output/README.md"
  content  = "# Proyecto\n"
}
```

- `local_file` → tipo de recurso del provider `local`
- `readme` → nombre local (identificador dentro de Terraform, no afecta al mundo real)
- El recurso se referencia como `local_file.readme` desde otros lugares del código

**Referenciando atributos de otros recursos (dependencias implícitas):**

```hcl
resource "docker_container" "app" {
  image = docker_image.nginx.image_id   # referencia a otro recurso
  name  = "mi-app"
}
```

Cuando un recurso referencia el atributo de otro, Terraform crea automáticamente una dependencia entre ambos: primero creará `docker_image.nginx` y luego `docker_container.app`. Esto se llama **dependencia implícita** y es la forma preferida.

---

## 8. Variables de entrada

Las variables permiten parametrizar la configuración para hacerla reutilizable y flexible. Se definen en `variables.tf` y se usan en el código con `var.<nombre>`.

**Declaración:**

```hcl
variable "project_name" {
  type        = string
  description = "Nombre del proyecto"
  default     = "my-project"    # Si no hay default, la variable es obligatoria
}

variable "author" {
  type        = string
  description = "Nombre del autor"
  # Sin default → Terraform la pedirá interactivamente o hay que pasarla
}

variable "environment" {
  type        = string
  description = "Entorno de despliegue"
  default     = "dev"
}
```

**Uso:**

```hcl
resource "local_file" "readme" {
  filename = "${path.module}/${var.project_name}/README.md"
  content  = "Autor: ${var.author}\nEntorno: ${var.environment}"
}
```

**Formas de asignar valores (de menor a mayor prioridad):**

| Prioridad | Método |
|---|---|
| 1 (menor) | Valor `default` en la declaración |
| 2 | Variable de entorno `TF_VAR_nombre` |
| 3 | Fichero `terraform.tfvars` (se carga automáticamente) |
| 4 | Ficheros `*.auto.tfvars` |
| 5 | Fichero especificado con `-var-file` |
| 6 (mayor) | Flag `-var="nombre=valor"` en línea de comandos |

**Ejemplos:**

```bash
# Fichero terraform.tfvars (se carga automáticamente)
project_name = "api-gateway"
author       = "Ana García"

# Línea de comandos (máxima prioridad)
terraform apply -var="author=Ana García" -var="project_name=api-gateway"

# Variable de entorno
export TF_VAR_author="Ana García"
terraform apply
```

---

## 9. Local Values

Los locals son **valores calculados internamente**. No son inputs (no los pasa el usuario) ni outputs (no los ve el usuario fuera del módulo). Son como constantes privadas que simplifican o evitan repetir expresiones complejas.

```hcl
locals {
  # Ruta base calculada a partir de una variable
  project_dir = "${path.module}/output/${var.project_name}"

  # Contenido calculado
  gitignore_content = <<-EOT
    .env
    node_modules/
    *.log
  EOT

  # Expresión condicional
  modo = var.environment == "production" ? "prod" : "dev"
}
```

**Uso:**

```hcl
resource "local_file" "readme" {
  filename = "${local.project_dir}/README.md"   # local.nombre
  content  = "Entorno: ${upper(var.environment)}"
}

resource "local_file" "gitignore" {
  filename = "${local.project_dir}/.gitignore"
  content  = local.gitignore_content
}
```

**¿Cuándo usar locals en vez de variables?**

- Variables → para valores que vienen del exterior (el usuario, un fichero `.tfvars`)
- Locals → para valores que se calculan dentro del módulo a partir de otras cosas

**`path.module` vs `path.root`:**

| Variable | Valor |
|---|---|
| `path.module` | Ruta del directorio donde está el fichero `.tf` que lo usa |
| `path.root` | Ruta del root module (donde se ejecuta `terraform apply`) |

En el root module son idénticos. En un módulo anidado, `path.module` apunta al directorio del módulo y `path.root` al directorio raíz del proyecto. Por eso en la práctica DevForge se pasa `base_dir = path.module` desde el root y dentro del módulo se usa `var.base_dir` en lugar de `path.module`.

---

## 10. Output Values

Los outputs exponen valores al exterior una vez que Terraform termina de aplicar los cambios. Tienen dos usos:

1. **Mostrar información** al operador (IPs, rutas de ficheros creados, IDs…)
2. **Pasar datos** de un módulo a otro (el root module accede a los outputs de sus módulos hijos)

**Declaración:**

```hcl
output "readme_path" {
  description = "Ruta del fichero generado"
  value       = local_file.readme.filename
}

output "resumen" {
  description = "Resumen de configuración"
  value = {
    proyecto = var.project_name
    autor    = var.author
    entorno  = upper(var.environment)
  }
}

# Output sensible (no se muestra en logs)
output "password" {
  value     = random_password.pwd.result
  sensitive = true
}
```

**Comandos:**

```bash
terraform output             # Muestra todos los outputs
terraform output readme_path # Muestra un output específico
```

**Acceder al output de un módulo desde el root:**

```hcl
# Si el módulo expone: output "project_path" { value = ... }
# El root lo accede así:
module.backend.project_path
module.frontend.project_path
```

---

## 11. Tipos de datos

### Tipos primitivos

```hcl
# string: cadena de texto
variable "nombre" {
  type    = string
  default = "mi-app"
}

# number: número entero o decimal
variable "puerto" {
  type    = number
  default = 8080
}

# bool: verdadero o falso
variable "debug" {
  type    = bool
  default = false
}
```

### Tipos compuestos

**`list` — lista ordenada de valores del mismo tipo**

```hcl
variable "puertos" {
  type    = list(number)
  default = [80, 443, 8080]
}
# Acceso: var.puertos[0] → 80
```

Se usa con `count` para crear N recursos:

```hcl
variable "ficheros" {
  type    = list(string)
  default = ["config.txt", "data.txt", "log.txt"]
}

resource "local_file" "files" {
  count    = length(var.ficheros)           # 3 iteraciones
  filename = "${path.module}/${var.ficheros[count.index]}"
  content  = "Archivo número ${count.index + 1}"
}
# Crea 3 ficheros: config.txt, data.txt, log.txt
```

**`map` — diccionario clave-valor**

```hcl
variable "contenidos" {
  type = map(string)
  default = {
    "readme.txt"  = "Bienvenido al proyecto"
    "config.txt"  = "env=development"
    "version.txt" = "1.0.0"
  }
}
# Acceso: var.contenidos["readme.txt"] → "Bienvenido al proyecto"
```

Se usa con `for_each` (itera sobre cada par clave-valor):

```hcl
resource "local_file" "config_files" {
  for_each = var.contenidos
  filename = "${path.module}/${each.key}"
  content  = each.value
}
# each.key → nombre del fichero, each.value → contenido
```

**`object` — estructura con campos tipados**

```hcl
variable "servidor" {
  type = object({
    hostname = string
    ip       = string
    puerto   = number
  })
  default = {
    hostname = "webserver"
    ip       = "192.168.1.10"
    puerto   = 8080
  }
}
# Acceso: var.servidor.hostname → "webserver"
```

**`set` — colección de valores únicos** (sin orden garantizado)

```hcl
variable "entornos" {
  type    = set(string)
  default = ["dev", "staging", "prod"]
}
# Se usa con for_each, igual que map
```

### `count` vs `for_each` — cuándo usar cada uno

| | `count` | `for_each` |
|---|---|---|
| **Input** | Número entero | `map` o `set` |
| **Identificador** | Índice numérico (`count.index`) | Clave (`each.key`) |
| **Referencia** | `resource.name[0]` | `resource.name["clave"]` |
| **Problema** | Si eliminas un elemento del medio, los índices cambian y Terraform destruye y recrea todo lo que sigue | Cada recurso se identifica por su clave, cambios localizados |
| **Cuándo usarlo** | Cuando todos los recursos son idénticos y solo varía el número | Cuando los recursos tienen configuración diferente o necesitas referenciarlos por nombre |

---

## 12. Expresiones y funciones

### Interpolación de strings

```hcl
"El entorno es ${var.environment}"
"${var.prefix}-${var.project_name}-service"
"${local.project_dir}/README.md"
```

### Condicional (operador ternario)

```hcl
content = var.environment == "production" ? "modo=prod\ndebug=false" : "modo=dev\ndebug=true"
```

### Strings multilínea (heredoc)

```hcl
content = <<-EOT
  Línea 1
  Línea 2
  Nombre: ${var.project_name}
EOT
```

El `<<-` (con guión) elimina la indentación común del bloque. Sin el guión (`<<EOT`) el texto se toma literalmente con toda la indentación incluida.

### Funciones de strings

```hcl
upper("dev")                          # → "DEV"
lower("HELLO")                        # → "hello"
title("hello world")                  # → "Hello World"
trim("  texto  ")                     # → "texto"
replace("hello-world", "-", "_")      # → "hello_world"
substr("terraform", 0, 5)             # → "terra"
```

### Funciones de listas y mapas

```hcl
length([1, 2, 3])                     # → 3
concat([1, 2], [3, 4])                # → [1, 2, 3, 4]
contains([1, 2, 3], 2)                # → true
distinct([1, 2, 2, 3])                # → [1, 2, 3]
element(["a", "b", "c"], 1)           # → "b"
join(", ", ["a", "b", "c"])           # → "a, b, c"
keys({a = 1, b = 2})                  # → ["a", "b"]
values({a = 1, b = 2})                # → [1, 2]
merge(map1, map2)                     # Combina dos mapas
```

### Funciones numéricas

```hcl
max(1, 5, 3)    # → 5
min(1, 5, 3)    # → 1
abs(-10)        # → 10
ceil(1.3)       # → 2
floor(1.7)      # → 1
```

### Funciones de codificación

```hcl
jsonencode(var.config)       # Convierte un objeto HCL a JSON string
yamlencode(var.config)       # Convierte a YAML string
base64encode("texto")        # Codifica en base64
base64decode("dGV4dG8=")     # Decodifica de base64
```

### Expresión `for` (transformar colecciones)

```hcl
# Transformar una lista
[for s in ["a", "b", "c"] : upper(s)]   # → ["A", "B", "C"]

# Construir un mapa desde módulos
{
  for name, vm in module.vms : name => vm.ip_publica
}
```

---

## 13. Data Sources

Los data sources permiten **consultar información de recursos existentes** o externos sin gestionarlos. Terraform los lee pero no los crea ni los destruye.

**Diferencia clave:**
- `resource` → Terraform crea y gestiona el recurso
- `data` → Terraform solo consulta, no toca nada

**Sintaxis:**

```hcl
data "<provider>_<tipo>" "<nombre>" {
  # parámetros de búsqueda
}
```

**Acceso a sus atributos:**

```hcl
data.<provider>_<tipo>.<nombre>.<atributo>
```

**Ejemplo con el provider local:**

```hcl
# Leer un fichero que ya existe en disco
data "local_file" "config_existente" {
  filename   = "config.txt"
  depends_on = [local_file.config]   # Esperar a que se cree primero
}

output "contenido" {
  value = data.local_file.config_existente.content
}
```

**¿Para qué sirven?**
- Evitar duplicación de información (leer datos que ya están en otro sistema)
- Integrar con recursos que no gestiona Terraform (creados manualmente, por otro equipo…)
- Obtener IDs, IPs u otros atributos de recursos existentes para usarlos como input

---

## 14. Estado (terraform.tfstate)

El estado es el fichero JSON donde Terraform guarda el **mapeo entre el código HCL y los recursos reales de la infraestructura**.

**Sin estado, Terraform no puede funcionar:** no sabría qué recursos ha creado, si han cambiado, cuáles eliminar al hacer `destroy`, o en qué orden actuar.

**Contenido típico del tfstate:**

```json
{
  "resources": [
    {
      "type": "local_file",
      "name": "readme",
      "instances": [{
        "attributes": {
          "id": "sha1:abc123...",
          "filename": "./output/README.md",
          "content": "# Mi proyecto\n"
        }
      }]
    }
  ]
}
```

Guarda: lista de recursos gestionados, IDs reales asignados por los providers, atributos conocidos, dependencias entre recursos, versiones de Terraform y providers.

**Por qué es importante:**
- **Seguimiento:** Terraform sabe qué existe realmente.
- **Rendimiento:** no necesita consultar la infraestructura en cada operación.
- **`plan` y `apply` correctos:** calcula el diff entre el estado deseado (`.tf`) y el estado actual (`.tfstate`).

**Comandos del estado:**

```bash
terraform show              # Muestra el estado en formato legible
terraform state list        # Lista todos los recursos en el estado
terraform state show <rec>  # Detalle de un recurso concreto
```

### Problema del estado local

El estado por defecto se guarda en `terraform.tfstate` en el directorio de trabajo. En equipos esto es un problema:

- **Riesgo de pérdida:** si se pierde el fichero, Terraform pierde el control de toda la infraestructura.
- **Aislamiento:** solo lo tiene quien ejecutó `apply`. Nadie más puede aplicar cambios.
- **Sin bloqueo:** dos personas pueden ejecutar `apply` simultáneamente y corromper el estado.
- **Seguridad:** puede contener contraseñas, tokens y datos sensibles en texto plano.

> **Regla de oro: nunca subir `terraform.tfstate` a Git.**

### Estado remoto (Remote State)

La solución es mover el estado a un backend remoto compartido: AWS S3, Azure Blob Storage, Google Cloud Storage, Terraform Cloud, etc.

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-bucket-tfstate"
    key            = "proyecto/dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock-table"   # Bloqueo para evitar conflictos
    encrypt        = true                      # Cifrado en reposo
  }
}
```

El backend remoto aporta bloqueo automático (solo una persona puede hacer `apply` a la vez) y versionado del estado.

---

## 15. Ciclo de vida de los recursos

Cuando Terraform aplica cambios, cada recurso puede estar en una de estas situaciones:

- `+` **create:** el recurso no existe en el estado → se crea
- `~` **update in-place:** el recurso existe y algún atributo admite cambio en caliente → se modifica
- `-/+` **replace (destroy + create):** el recurso existe pero el cambio requiere destruirlo y recrearlo (ej: cambiar el puerto de un contenedor Docker)
- `-` **destroy:** el recurso existe en el estado pero no en el código → se elimina

### Meta-argumento `lifecycle`

Permite controlar este comportamiento:

**`create_before_destroy`:** crea el nuevo recurso antes de destruir el antiguo. Útil para recursos críticos donde no puede haber downtime.

```hcl
resource "local_file" "config" {
  filename = "${path.module}/config.txt"
  content  = "version=2"

  lifecycle {
    create_before_destroy = true
  }
}
```

**`prevent_destroy`:** impide que Terraform elimine este recurso. Si alguien intenta hacer `destroy` o eliminarlo del código, Terraform falla con error. Ideal para bases de datos o recursos críticos.

```hcl
resource "local_file" "datos_criticos" {
  filename = "${path.module}/datos.txt"
  content  = "Datos importantes"

  lifecycle {
    prevent_destroy = true
  }
}
```

**`ignore_changes`:** ignora cambios en ciertos atributos. Si el atributo cambia fuera de Terraform (alguien modifica el fichero manualmente), en el próximo `plan` Terraform no lo marcará para actualizar.

```hcl
resource "local_file" "config" {
  filename = "${path.module}/config.txt"
  content  = "Puerto: 8080\nTimestamp: ${timestamp()}"

  lifecycle {
    ignore_changes = [content]   # Ignora cambios en el contenido
  }
}
```

### Meta-argumento `depends_on`

Define **dependencias explícitas** entre recursos cuando la dependencia no es obvia en el código (es decir, cuando no se referencia el atributo de otro recurso directamente).

```hcl
resource "local_file" "base" {
  filename = "${path.module}/base.txt"
  content  = "Fichero base"
}

resource "local_file" "dependiente" {
  filename   = "${path.module}/dependiente.txt"
  content    = "Depende del base"
  depends_on = [local_file.base]   # Terraform creará base primero
}
```

> **Cuándo usar `depends_on`:** solo cuando la dependencia no se puede inferir del código (no hay referencia directa a un atributo del otro recurso). Si hay referencia directa (`local_file.base.filename`), la dependencia ya es implícita y no hace falta `depends_on`.

---

## 16. Módulos

Un módulo es **un directorio con ficheros `.tf`**. Permiten encapsular lógica reutilizable y desplegarla varias veces con distintos parámetros.

**Tipos de módulos:**
- **Root module:** el directorio desde donde se ejecuta `terraform apply`. Siempre existe.
- **Módulos hijos (child modules):** subdirectorios con sus propios `.tf`. Se llaman desde el root module o desde otros módulos.
- **Módulos del Terraform Registry:** módulos públicos verificados y mantenidos por la comunidad (`registry.terraform.io/browse/modules`).

**Beneficios de los módulos:**
- **Reutilización:** define la lógica una vez, úsala N veces con distintos inputs.
- **Encapsulación:** el módulo oculta su implementación interna; solo expone variables y outputs.
- **Organización:** proyectos grandes se dividen en piezas manejables.
- **Pruebas independientes:** cada módulo se puede probar por separado.

### Estructura de un módulo

```
modules/
└── proyecto/
    ├── main.tf        ← recursos del módulo
    ├── variables.tf   ← inputs que acepta el módulo
    └── outputs.tf     ← datos que expone el módulo
```

### Definir un módulo — variables de entrada

```hcl
# modules/proyecto/variables.tf
variable "project_name" {
  type        = string
  description = "Nombre del proyecto"
  default     = "my-project"
}

variable "author" {
  type        = string
  description = "Nombre del autor"
  # Sin default → obligatorio
}

variable "base_dir" {
  type        = string
  description = "Directorio base"
}
```

### Definir un módulo — lógica principal

```hcl
# modules/proyecto/main.tf
locals {
  project_dir = "${var.base_dir}/output/${var.project_name}"
}

resource "local_file" "readme" {
  filename = "${local.project_dir}/README.md"
  content  = "# ${var.project_name}\nAutor: ${var.author}"
}
```

### Definir un módulo — outputs

```hcl
# modules/proyecto/outputs.tf
output "project_path" {
  description = "Ruta del directorio del proyecto"
  value       = local.project_dir
}
```

### Usar un módulo desde el root

```hcl
# main.tf (root module)
module "backend" {
  source       = "./modules/proyecto"   # Ruta al directorio del módulo
  project_name = "backend-api"
  author       = "Fernando"
  base_dir     = path.module            # Pasa la ruta del root al módulo
}

module "frontend" {
  source       = "./modules/proyecto"
  project_name = "frontend-app"
  author       = "Fernando"
  base_dir     = path.module
}
```

### Acceder a los outputs de un módulo

```hcl
# outputs.tf (root module)
output "rutas" {
  value = {
    backend  = module.backend.project_path    # module.<nombre>.<output>
    frontend = module.frontend.project_path
  }
}
```

> **Importante:** cada vez que añades un módulo nuevo (o cambias su `source`), hay que volver a ejecutar `terraform init` para que Terraform descargue o registre el módulo.

### Estructura de proyecto con múltiples entornos

```
proyecto/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── modules/
│   ├── web-app/
│   ├── database/
│   └── network/
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

Se aplica un entorno concreto con: `terraform apply -var-file="environments/dev.tfvars"`

---

## 17. Providers locales

Los providers locales interactúan con la **máquina donde se ejecuta Terraform**, no con servicios en la nube. Son ideales para aprender y para automatizar tareas del sistema de ficheros.

### Provider `local` (hashicorp/local) — el más importante para el examen

Crea y gestiona ficheros en el sistema de ficheros local.

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}
```

**Recursos disponibles:**

`local_file` — crea un fichero de texto:

```hcl
resource "local_file" "config" {
  filename        = "${path.module}/output/config.txt"
  content         = "clave=valor\nentorno=${var.environment}"
  file_permission = "0644"   # Opcional, permisos Unix
}
```

`local_sensitive_file` — igual que `local_file` pero el contenido se trata como sensible (no aparece en logs):

```hcl
resource "local_sensitive_file" "secreto" {
  filename = "${path.module}/secret.txt"
  content  = var.password
}
```

**Atributos exportados por `local_file`:**

```hcl
local_file.config.filename      # Ruta del fichero
local_file.config.id            # Hash SHA1 del contenido
local_file.config.content_md5   # Hash MD5 del contenido
```

**Data source `local_file` — leer un fichero existente:**

```hcl
data "local_file" "existente" {
  filename = "config.txt"
}

output "contenido" {
  value = data.local_file.existente.content
}
```

### Provider `random` (hashicorp/random) — generación de valores aleatorios

```hcl
resource "random_pet" "nombre" {
  length    = 2
  separator = "-"
}
# random_pet.nombre.id → "charming-tiger"

resource "random_password" "pwd" {
  length  = 16
  special = true
}
# random_password.pwd.result → contraseña aleatoria

resource "random_id" "uid" {
  byte_length = 4
}
# random_id.uid.hex → "a3b2c1d4"
```

### Provider `time` (hashicorp/time) — gestión de tiempos

```hcl
resource "time_static" "ahora" {}
# time_static.ahora.rfc3339 → "2025-11-29T17:37:42Z"
```

### Provider `null` (hashicorp/null) — ejecutar comandos locales

El null provider no crea recursos reales. Se usa con `local-exec` para ejecutar scripts o comandos:

```hcl
resource "null_resource" "echo" {
  provisioner "local-exec" {
    command = "echo 'Infraestructura creada'"
  }
}
```

---

## 18. Comandos esenciales

```bash
# Inicialización (siempre primero, y tras cambiar providers o módulos)
terraform init

# Formatear el código automáticamente
terraform fmt

# Validar sintaxis sin ejecutar nada
terraform validate

# Ver qué va a hacer antes de hacerlo (dry-run)
terraform plan

# Aplicar los cambios
terraform apply

# Aplicar sin pedir confirmación (útil en CI/CD)
terraform apply -auto-approve

# Pasar variables
terraform apply -var="autor=Ana" -var="proyecto=api"

# Destruir toda la infraestructura
terraform destroy

# Ver el estado actual en formato legible
terraform show

# Ver todos los outputs
terraform output

# Ver un output específico
terraform output nombre_output

# Ver el grafo de dependencias (formato DOT)
terraform graph

# Comandos de gestión del estado
terraform state list           # Listar recursos en el estado
terraform state show <recurso> # Ver detalle de un recurso
```

---

## 19. Buenas prácticas y recomendaciones

### Organización del código

Separar la configuración en ficheros por responsabilidad (no es obligatorio para Terraform, que lee todos los `.tf` del directorio, pero mejora la legibilidad):

- `main.tf` → recursos principales
- `variables.tf` → declaración de variables
- `outputs.tf` → outputs
- `terraform.tfvars` → valores concretos de variables (no subir a Git si tiene secretos)

### Gestión del estado

- Nunca subir `terraform.tfstate` a Git — puede contener secretos y genera conflictos.
- En proyectos reales: siempre usar **backend remoto** con bloqueo de estado.
- El estado puede contener información sensible (contraseñas, tokens). Tratarlo con el mismo nivel de seguridad que las credenciales.

### Variables y secretos

- Nunca poner credenciales en el código. Usar variables + `.tfvars` + variables de entorno.
- Usar `sensitive = true` en outputs que contengan datos sensibles.
- Separar configuración por entorno con ficheros `.tfvars` distintos (dev, staging, prod).

### Versionado

- Fijar versiones de providers con `~> X.Y` para reproducibilidad.
- El fichero `.terraform.lock.hcl` (generado por `init`) fija las versiones exactas. Debe subirse a Git.
- Especificar `required_version` de Terraform para que el proyecto no falle con versiones incompatibles.

### Flujo de trabajo seguro

```bash
terraform fmt       # Formatear
terraform validate  # Verificar sintaxis
terraform plan      # Revisar el plan con atención
terraform apply     # Aplicar solo si el plan es correcto
```

Leer el `terraform plan` completo antes de ejecutar `apply`. La línea resumen al final (`Plan: X to add, Y to change, Z to destroy`) no es suficiente.

### Seguridad

- Principio de mínimo privilegio: los providers de cloud deben usar credenciales con solo los permisos necesarios.
- Usar `lifecycle { prevent_destroy = true }` en recursos críticos como bases de datos de producción.
- Revisar cuidadosamente cualquier plan que muestre recursos a destruir.
