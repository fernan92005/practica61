# Terraform — Manual de Referencia (Proveedor Local)
> Infraestructuras de Soporte · UMA · 2026

---

## Índice

1. [Conceptos fundamentales](#1-conceptos-fundamentales)
2. [Comandos esenciales](#2-comandos-esenciales)
3. [Estructura de ficheros](#3-estructura-de-ficheros)
4. [Bloque terraform y provider](#4-bloque-terraform-y-provider)
5. [Variables](#5-variables)
6. [Locals](#6-locals)
7. [Outputs](#7-outputs)
8. [Recursos — proveedor local](#8-recursos--proveedor-local)
9. [Funciones útiles](#9-funciones-útiles)
10. [Módulos](#10-módulos)
11. [Práctica completa — DevForge](#11-práctica-completa--devforge)
    - [Parte 1 — Un fichero, un recurso](#parte-1--un-fichero-un-recurso)
    - [Parte 2 — Variables y outputs](#parte-2--variables-y-outputs)
    - [Parte 3 — Múltiples recursos y locals](#parte-3--múltiples-recursos-y-locals)
    - [Parte 4 — Módulos](#parte-4--módulos)
12. [Errores comunes](#12-errores-comunes)

---

## 1. Conceptos fundamentales

Terraform es una herramienta de **Infraestructura como Código (IaC)**. Declaras el estado deseado en ficheros `.tf` y Terraform calcula qué crear, modificar o destruir para llegar a ese estado.

### Jerarquía de componentes

```
Proyecto Terraform
├── Ficheros .tf          → código declarativo
├── terraform.tfstate     → estado actual (qué existe realmente)
├── .terraform/           → binarios de providers descargados
└── .terraform.lock.hcl  → versiones fijadas de providers
```

### Conceptos clave

| Concepto | Descripción |
|---|---|
| **Provider** | Plugin que sabe hablar con un sistema (local, AWS, GCP…) |
| **Resource** | Elemento de infraestructura que Terraform gestiona |
| **Variable** | Parámetro de entrada configurable |
| **Local** | Valor calculado internamente, no expuesto al exterior |
| **Output** | Valor que Terraform muestra al terminar y expone a otros módulos |
| **Módulo** | Conjunto reutilizable de recursos agrupados en un directorio |
| **State** | Fichero JSON donde Terraform guarda qué ha creado |

### Flujo de trabajo

```
Escribir .tf → terraform init → terraform plan → terraform apply → terraform destroy
```

---

## 2. Comandos esenciales

```bash
terraform init        # Descarga providers declarados en required_providers
terraform fmt         # Formatea el código automáticamente
terraform validate    # Valida la sintaxis sin contactar APIs
terraform plan        # Muestra qué va a crear/modificar/destruir (dry-run)
terraform apply       # Aplica los cambios (pide confirmación)
terraform apply -auto-approve   # Aplica sin pedir confirmación
terraform destroy     # Destruye todos los recursos del estado
terraform show        # Muestra el estado actual en formato legible
terraform output      # Muestra los valores de los outputs
```

### Pasar variables por línea de comandos

```bash
terraform apply -var="author=Ana García"
terraform apply -var="author=Ana García" -var="project_name=api-gateway"
```

### Cuándo volver a ejecutar `terraform init`

Siempre que:
- Añadas un nuevo provider
- Añadas o cambies la ruta de un módulo (`source = "./modules/project"`)

---

## 3. Estructura de ficheros

### Proyecto básico

```
devforge/
├── main.tf        ← recursos principales
├── variables.tf   ← declaración de variables de entrada
└── outputs.tf     ← valores que Terraform expone al terminar
```

### Proyecto con módulos

```
devforge/
├── main.tf            ← root module: llama a los módulos
├── variables.tf
├── outputs.tf
└── modules/
    └── project/
        ├── main.tf    ← lógica del módulo
        ├── variables.tf
        └── outputs.tf
```

> Cada directorio con ficheros `.tf` es un módulo. El directorio raíz es el **root module**.

---

## 4. Bloque terraform y provider

```hcl
# main.tf (o versions.tf)
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"   # cualquier 2.x pero no 3.x
    }
  }
}
```

El proveedor `local` no necesita bloque `provider {}` adicional — no requiere credenciales ni configuración. Solo el bloque `terraform {}` es suficiente.

**Operadores de versión:**

| Operador | Significado |
|---|---|
| `"~> 2.0"` | `>= 2.0` y `< 3.0` (minor libre, major fijo) |
| `">= 2.0"` | Cualquier versión igual o superior |
| `"= 2.4.0"` | Exactamente esa versión |

---

## 5. Variables

### Declaración (`variables.tf`)

```hcl
variable "project_name" {
  type        = string
  description = "Nombre del proyecto"
  default     = "my-project"   # Opcional. Sin default → obligatorio pasarla
}

variable "author" {
  type        = string
  description = "Nombre del autor o autora"
  # Sin default → Terraform la pedirá interactivamente o hay que pasarla con -var
}

variable "environment" {
  type        = string
  description = "Entorno de despliegue"
  default     = "dev"
}
```

### Tipos disponibles

```hcl
type = string
type = number
type = bool
type = list(string)
type = map(string)
type = object({ name = string, port = number })
```

### Uso en el código

```hcl
var.project_name
var.author
var.environment
```

### Formas de pasar el valor

```bash
# 1. Por línea de comandos
terraform apply -var="author=Ana García"

# 2. Fichero .tfvars
# terraform.tfvars (se carga automáticamente)
author       = "Ana García"
project_name = "api-gateway"

# 3. Interactivamente — Terraform lo pregunta si no hay default ni -var
```

---

## 6. Locals

Los locals son valores calculados internamente. No son inputs (como variables) ni outputs — son como constantes privadas del módulo.

```hcl
locals {
  project_dir = "${path.module}/output/${var.project_name}"

  gitignore_content = <<-EOT
    .env
    node_modules/
    *.log
  EOT
}
```

### Uso

```hcl
local.project_dir
local.gitignore_content
```

### `path.module` vs `path.root`

| Variable | Valor |
|---|---|
| `path.module` | Ruta del directorio donde está el fichero `.tf` actual |
| `path.root` | Ruta del root module (el directorio desde donde se ejecuta Terraform) |

En el **root module** son iguales. En un módulo anidado, `path.module` apunta al directorio del módulo y `path.root` al raíz del proyecto.

### Strings multilínea (`<<-EOT ... EOT`)

```hcl
content = <<-EOT
  Línea 1
  Línea 2
  Línea 3
EOT
```

El `<<-` (con guión) elimina la indentación común del bloque. Sin guión (`<<EOT`) el texto se toma literalmente con toda la indentación.

---

## 7. Outputs

```hcl
# outputs.tf
output "readme_path" {
  description = "Ruta del fichero generado"
  value       = local_file.readme.filename
}

output "directions" {
  value = {
    backend  = module.backend.project_path
    frontend = module.frontend.project_path
  }
}
```

- Se muestran en la terminal al terminar `terraform apply`.
- Los módulos los exponen para que el root module pueda referenciarlos.
- Se consultan con `terraform output`.

### Referenciar el output de un módulo

```hcl
module.backend.project_path     # output "project_path" del módulo "backend"
module.frontend.project_path
```

---

## 8. Recursos — proveedor local

El proveedor `hashicorp/local` crea y gestiona ficheros en el sistema de ficheros local.

### `local_file` — crear un fichero

```hcl
resource "local_file" "readme" {
  filename = "${path.module}/output/README.md"
  content  = "# Mi primer proyecto Terraform\n\nGenerado automáticamente.\n"
}
```

**Atributos principales:**

| Atributo | Descripción |
|---|---|
| `filename` | Ruta completa del fichero a crear. Terraform crea los directorios intermedios automáticamente. |
| `content` | Contenido del fichero (string) |
| `content_base64` | Contenido en base64 (para ficheros binarios) |

**Atributos exportados (para usar en otros recursos/outputs):**

```hcl
local_file.readme.filename   # La ruta del fichero
local_file.readme.id         # Hash SHA1 del contenido
```

### Fichero vacío (para que git reconozca el directorio)

```hcl
resource "local_file" "src_keep" {
  filename = "${local.project_dir}/src/.keep"
  content  = ""
}
```

### Referencia de recursos

```
resource "<tipo>" "<nombre_local>" { ... }
```

Se referencia como `<tipo>.<nombre_local>.<atributo>`:

```hcl
local_file.readme.filename
local_file.gitignore.id
```

---

## 9. Funciones útiles

```hcl
upper("dev")           # → "DEV"
lower("DEV")           # → "dev"
trimspace("  hola  ")  # → "hola"

# Interpolación de variables en strings
"${var.project_name}-service"
"${path.module}/output/${var.project_name}"

# Interpolación de locals
"${local.project_dir}/README.md"
```

---

## 10. Módulos

Un módulo es un directorio con ficheros `.tf`. Permite reutilizar lógica declarando varias instancias con distintos parámetros.

### Llamar a un módulo

```hcl
# main.tf (root module)
module "backend" {
  source       = "./modules/project"   # Ruta al directorio del módulo

  # Variables del módulo (deben estar declaradas en modules/project/variables.tf)
  project_name = "backend-api"
  author       = "Fernando"
  environment  = "dev"
  base_dir     = path.module
}

module "frontend" {
  source       = "./modules/project"
  project_name = "frontend-app"
  author       = "Fernando"
  environment  = "dev"
  base_dir     = path.module
}
```

> **Importante:** Tras añadir un módulo nuevo hay que volver a ejecutar `terraform init`.

### Variables del módulo (`modules/project/variables.tf`)

```hcl
variable "project_name" {
  type        = string
  description = "Nombre del proyecto"
  default     = "my-project"
}

variable "author" {
  type        = string
  description = "Nombre del autor o autora"
}

variable "environment" {
  type        = string
  description = "Entorno de despliegue"
  default     = "dev"
}

variable "base_dir" {
  type        = string
  description = "Directorio base donde crear el proyecto"
}
```

### Outputs del módulo (`modules/project/outputs.tf`)

```hcl
output "project_path" {
  description = "Ruta del directorio del proyecto"
  value       = local.project_dir
}
```

### Usar el output del módulo en el root

```hcl
# outputs.tf (root)
output "directions" {
  value = {
    backend  = module.backend.project_path
    frontend = module.frontend.project_path
  }
}
```

---

## 11. Práctica completa — DevForge

### Parte 1 — Un fichero, un recurso

**Objetivo:** crear el primer fichero con Terraform.

**`main.tf`:**

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "readme" {
  filename = "${path.module}/output/README.md"
  content  = "# Mi primer proyecto Terraform\n\nGenerado automáticamente.\n"
}
```

**Comandos:**

```bash
terraform init     # Descarga el provider hashicorp/local
terraform plan     # Muestra: +1 resource to add
terraform apply    # Crea output/README.md
terraform show     # Muestra el estado actual
```

**Resultado:** se crea `output/README.md` con el contenido indicado. Terraform crea el directorio `output/` automáticamente.

---

### Parte 2 — Variables y outputs

**Objetivo:** parametrizar la configuración.

**Estructura:**
```
devforge/
├── main.tf
├── variables.tf
└── outputs.tf
```

**`variables.tf`:**

```hcl
variable "project_name" {
  type        = string
  description = "Nombre del proyecto"
  default     = "my-project"
}

variable "author" {
  type        = string
  description = "Nombre del autor o autora"
  # Sin default → obligatorio pasarla
}

variable "environment" {
  type        = string
  description = "Entorno de despliegue"
  default     = "dev"
}
```

**`main.tf`:**

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "readme" {
  filename = "${path.module}/output/README.md"
  content  = <<-EOT
    # ${var.project_name}

    Autor: ${var.author}
    Entorno: ${upper(var.environment)}

    Generado automáticamente.
  EOT
}
```

**`outputs.tf`:**

```hcl
output "readme_path" {
  description = "Ruta del fichero generado"
  value       = local_file.readme.filename
}
```

**Comandos:**

```bash
terraform apply -var="author=Ana García" -var="project_name=api-gateway"
```

---

### Parte 3 — Múltiples recursos y locals

**Objetivo:** generar la estructura completa de un proyecto usando `locals`.

**Estructura generada:**
```
output/<project_name>/
├── README.md
├── .gitignore
├── src/.keep
├── test/.keep
└── docs/index.md
```

**`main.tf`:**

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

locals {
  project_dir = "${path.module}/output/${var.project_name}"

  gitignore_content = <<-EOT
    .env
    node_modules/
    *.log
  EOT
}

resource "local_file" "readme" {
  filename = "${local.project_dir}/README.md"
  content  = <<-EOT
    # ${var.project_name}

    Autor: ${var.author}
    Entorno: ${upper(var.environment)}

    Generado automáticamente.
  EOT
}

resource "local_file" "gitignore" {
  filename = "${local.project_dir}/.gitignore"
  content  = local.gitignore_content
}

resource "local_file" "src_keep" {
  filename = "${local.project_dir}/src/.keep"
  content  = ""
}

resource "local_file" "test_keep" {
  filename = "${local.project_dir}/test/.keep"
  content  = ""
}

resource "local_file" "index" {
  filename = "${local.project_dir}/docs/index.md"
  content  = <<-EOT
    # Documentación de ${var.project_name}
  EOT
}
```

---

### Parte 4 — Módulos

**Objetivo:** extraer la lógica a un módulo reutilizable e instanciarlo dos veces.

**Estructura final:**
```
devforge/
├── main.tf            ← root module
├── variables.tf
├── outputs.tf
└── modules/
    └── project/
        ├── main.tf    ← lógica del módulo
        ├── variables.tf
        └── outputs.tf
```

**`modules/project/variables.tf`:**

```hcl
variable "project_name" {
  type        = string
  description = "Nombre del proyecto"
  default     = "my-project"
}

variable "author" {
  type        = string
  description = "Nombre del autor o autora"
}

variable "environment" {
  type        = string
  description = "Entorno de despliegue"
  default     = "dev"
}

variable "base_dir" {
  type        = string
  description = "Directorio base donde crear el proyecto"
}
```

**`modules/project/main.tf`:**

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

locals {
  project_dir = "${var.base_dir}/output/${var.project_name}"

  gitignore_content = <<-EOT
    .env
    node_modules/
    *.log
  EOT
}

resource "local_file" "readme" {
  filename = "${local.project_dir}/README.md"
  content  = <<-EOT
    # ${var.project_name}
    Autor: ${var.author}
    Entorno: ${upper(var.environment)}
    Generado automáticamente.
  EOT
}

resource "local_file" "gitignore" {
  filename = "${local.project_dir}/.gitignore"
  content  = local.gitignore_content
}

resource "local_file" "src_keep" {
  filename = "${local.project_dir}/src/.keep"
  content  = ""
}

resource "local_file" "test_keep" {
  filename = "${local.project_dir}/test/.keep"
  content  = ""
}

resource "local_file" "index" {
  filename = "${local.project_dir}/docs/index.md"
  content  = <<-EOT
    # Documentación de ${var.project_name}
  EOT
}
```

**`modules/project/outputs.tf`:**

```hcl
output "project_path" {
  description = "Ruta del directorio del proyecto"
  value       = local.project_dir
}
```

**`main.tf` (root module — solo llama a módulos):**

```hcl
module "backend" {
  source       = "./modules/project"
  project_name = "backend-api"
  author       = "Fernando"
  environment  = "dev"
  base_dir     = path.module
}

module "frontend" {
  source       = "./modules/project"
  project_name = "frontend-app"
  author       = "Fernando"
  environment  = "dev"
  base_dir     = path.module
}
```

**`outputs.tf` (root module):**

```hcl
output "directions" {
  value = {
    backend  = module.backend.project_path
    frontend = module.frontend.project_path
  }
}
```

**Comandos:**

```bash
terraform init     # Obligatorio tras añadir el módulo
terraform plan
terraform apply
```

**Resultado en `output/`:**
```
output/
├── backend-api/
│   ├── README.md       → Entorno: DEV, Autor: Fernando
│   ├── .gitignore      → .env / node_modules/ / *.log
│   ├── src/.keep
│   ├── test/.keep
│   └── docs/index.md
└── frontend-app/
    ├── README.md
    ├── .gitignore
    ├── src/.keep
    ├── test/.keep
    └── docs/index.md
```

---

## 12. Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `No such file or directory` al `init` | El directorio `modules/project` no existe | Crear el directorio antes de ejecutar `init` |
| `Error: Module not installed` | Se añadió un módulo sin volver a hacer `init` | Ejecutar `terraform init` de nuevo |
| `var.author is required` | Variable sin default no pasada | Usar `-var="author=Nombre"` o añadir default |
| `Error: Reference to undeclared input variable` | Se usa `var.X` pero no está declarada en `variables.tf` | Añadir la declaración de la variable |
| `Error: Unsupported argument` | Se pasa un argumento al módulo que no está declarado en su `variables.tf` | Añadir la variable al módulo o eliminar el argumento |
| Ficheros no se eliminan al `destroy` | Comportamiento esperado con `local_file` en algunos casos | Verificar con `terraform show` antes de destroy |
| Output vacío tras `apply` | El output referencia un módulo/recurso incorrecto | Revisar los nombres exactos de módulo y output |
| `path.module` apunta al directorio incorrecto | Se usa en el módulo esperando la ruta raíz | Pasar `base_dir = path.module` desde el root y usar `var.base_dir` en el módulo |
