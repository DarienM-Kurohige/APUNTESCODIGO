# Terraform — Unidad 2 | Apuntes de laboratorio

## Entorno

En AWS Academy inicio el laboratorio y uso las credenciales de la sesión en la terminal PowerShell de VS Code. Abro la carpeta raíz del proyecto, no la carpeta del módulo EC2. El provider AWS es quien hace las solicitudes a AWS; VS Code se utiliza para editar y ejecutar Terraform.

```powershell
terraform -version
$env:AWS_ACCESS_KEY_ID="VALOR_DEL_LAB"
$env:AWS_SECRET_ACCESS_KEY="VALOR_DEL_LAB"
$env:AWS_SESSION_TOKEN="VALOR_DEL_LAB"
$env:AWS_DEFAULT_REGION="us-east-1"
```

Los tres valores de credenciales se reemplazan por los del laboratorio. La AMI y las zonas también tienen que corresponder a `us-east-1`.

```text
proyecto-formativa/
  providers.tf       variables.tf       terraform.tfvars
  main.tf            outputs.tf
  modulos/
    vpc/  variables.tf  main.tf  outputs.tf
    ec2/  variables.tf  main.tf  outputs.tf
    nlb/  variables.tf  main.tf  outputs.tf
    s3/   variables.tf  main.tf  outputs.tf
```

Los `.tf` de cada carpeta se leen en conjunto. En la raíz declaro valores y llamo a los hijos. Cada hijo administra sus recursos y deja outputs para que la raíz los utilice.

## Proyecto de la formativa

El ejercicio que ya trabajamos utiliza una VPC `10.0.0.0/16`, dos subredes `10.0.1.0/24` y `10.0.2.0/24`, dos EC2, dos NLB internos y dos buckets S3 con bloqueo de acceso público. Esta configuración es la base; las modificaciones de la clase están más abajo.

### Módulo raíz — `providers.tf`

AWS administra los servicios y Random genera un sufijo para los nombres. Defino el provider en la raíz; no tengo que repetirlo en los hijos.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Módulo raíz — `variables.tf`

Estas variables pertenecen al proyecto completo. `ami_ec2` es un texto obligatorio; `cantidad_ec2`, `cantidad_nlb` y `cantidad_s3` permiten modificar cuántos recursos crea cada hijo. Los CIDR y las zonas están en listas porque el módulo de red genera dos subredes.

```hcl
variable "nombre_proyecto" {
  type    = string
  default = "iac-formativa"
}

variable "cidr_vpc" {
  type    = string
  default = "10.0.0.0/16"
}

variable "cidrs_subredes" {
  type = list(string)

  default = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]
}

variable "zonas_disponibilidad" {
  type = list(string)

  default = [
    "us-east-1a",
    "us-east-1b"
  ]
}

variable "ami_ec2" {
  type = string
}

variable "tipo_ec2" {
  type    = string
  default = "t3.micro"
}

variable "cantidad_ec2" {
  type    = number
  default = 2
}

variable "cantidad_nlb" {
  type    = number
  default = 2
}

variable "cantidad_s3" {
  type    = number
  default = 2
}
```

### Módulo raíz — `terraform.tfvars`

Aquí asigno el ID de la AMI. Cuando necesito cambiar un valor que ya tiene `default`, también puedo especificarlo aquí sin editar los recursos: por ejemplo, `cantidad_ec2 = 3`.

```hcl
ami_ec2 = "ami-04b938437b69a2c9c"
```

### Módulo raíz — `main.tf`

Random construye el sufijo y `local.nombre` combina ese resultado con `nombre_proyecto`. Las llamadas `red`, `computo`, `balanceo` y `almacenamiento` corresponden a carpetas hijas. El nombre de la llamada no tiene por qué ser igual al de la carpeta.

`subnet_ids = module.red.subnet_ids` es la unión entre VPC y EC2/NLB: el output de red se entrega como entrada a los otros módulos. S3 no requiere una subred.

```hcl
resource "random_string" "sufijo" {
  length  = 6
  upper   = false
  lower   = true
  numeric = true
  special = false
}

locals {
  nombre = "${var.nombre_proyecto}-${random_string.sufijo.result}"
}

module "red" {
  source = "./modulos/vpc"

  nombre       = local.nombre
  cidr         = var.cidr_vpc
  subnet_cidrs = var.cidrs_subredes
  azs          = var.zonas_disponibilidad
}

module "computo" {
  source = "./modulos/ec2"

  nombre         = local.nombre
  ami_id         = var.ami_ec2
  tipo_instancia = var.tipo_ec2
  cantidad       = var.cantidad_ec2
  subnet_ids     = module.red.subnet_ids
}

module "balanceo" {
  source = "./modulos/nlb"

  nombre     = local.nombre
  cantidad   = var.cantidad_nlb
  subnet_ids = module.red.subnet_ids
}

module "almacenamiento" {
  source = "./modulos/s3"

  nombre   = local.nombre
  cantidad = var.cantidad_s3
}
```

### Módulo raíz — `outputs.tf`

Los módulos hijos devuelven datos; en la raíz elijo cuáles consultar con `terraform output`. Por ejemplo, `module.computo.instance_ids` toma el output `instance_ids` del hijo EC2.

```hcl
output "id_vpc" {
  value = module.red.vpc_id
}

output "ids_subredes" {
  value = module.red.subnet_ids
}

output "nombre_generado" {
  value = local.nombre
}

output "ids_ec2" {
  value = module.computo.instance_ids
}

output "ips_privadas_ec2" {
  value = module.computo.private_ips
}

output "dns_nlb" {
  value = module.balanceo.nlb_dns
}

output "nombres_buckets" {
  value = module.almacenamiento.bucket_names
}
```

## Módulo EC2 — `modulos/ec2/`

EC2 no crea subredes: las recibe. Así puedo reutilizar la misma carpeta si cambia la VPC. `variables.tf` describe lo que necesita; `main.tf` crea las instancias; `outputs.tf` devuelve los IDs e IP privadas.

### `variables.tf`

`ami_id` es una AMI (`string`), `tipo_instancia` es el tamaño, `cantidad` es un número y `subnet_ids` es una lista de IDs. En el root, `ami_id = var.ami_ec2` y `cantidad = var.cantidad_ec2` entregan esos valores.

```hcl
variable "nombre" {
  type = string
}

variable "ami_id" {
  type = string
}

variable "tipo_instancia" {
  type = string
}

variable "cantidad" {
  type = number
}

variable "subnet_ids" {
  type = list(string)
}
```

### `main.tf`

`count` crea las dos EC2 sin repetir el recurso. `count.index` comienza en cero; `% length(var.subnet_ids)` alterna entre las subredes disponibles. El `local` se ocupa solamente de preparar la etiqueta `Name`. La IP pública queda desactivada.

```hcl
locals {
  nombre_ec2 = "${var.nombre}-ec2"
}

resource "aws_instance" "servidores" {
  count = var.cantidad

  ami           = var.ami_id
  instance_type = var.tipo_instancia

  subnet_id = var.subnet_ids[
    count.index % length(var.subnet_ids)
  ]

  associate_public_ip_address = false

  tags = {
    Name = "${local.nombre_ec2}-${count.index + 1}"
  }
}
```

### `outputs.tf`

El operador `[*]` reúne el atributo indicado de todas las instancias creadas con `count`. La raíz lee estos resultados como `module.computo.instance_ids` y `module.computo.private_ips`.

```hcl
output "instance_ids" {
  value = aws_instance.servidores[*].id
}

output "private_ips" {
  value = aws_instance.servidores[*].private_ip
}
```

## Módulo VPC — `modulos/vpc/`

Esta carpeta crea la red y entrega los IDs de las subredes que EC2 y NLB necesitan. En esta versión no hay Internet Gateway: no lo pedía la formativa.

### `variables.tf`

`cidr` es el rango de la VPC; `subnet_cidrs` son los rangos de las subredes y `azs` sus zonas. Las dos listas deben mantener el mismo orden y tener suficientes elementos.

```hcl
variable "nombre" {
  type = string
}

variable "cidr" {
  type = string
}

variable "subnet_cidrs" {
  type = list(string)
}

variable "azs" {
  type = list(string)
}
```

### `main.tf`

`vpc_id = aws_vpc.principal.id` indica a qué VPC pertenece cada subred. `count = length(var.subnet_cidrs)` genera tantas subredes como CIDR haya en la lista. Si solo piden una, basta con una entrada en `cidrs_subredes` y una en `zonas_disponibilidad` de la raíz.

```hcl
locals {
  nombre_vpc = "${var.nombre}-vpc"
}

resource "aws_vpc" "principal" {
  cidr_block = var.cidr

  tags = {
    Name = local.nombre_vpc
  }
}

resource "aws_subnet" "subredes" {
  count = length(var.subnet_cidrs)

  vpc_id            = aws_vpc.principal.id
  cidr_block        = var.subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]

  tags = {
    Name = "${var.nombre}-subnet-${count.index + 1}"
  }
}
```

### `outputs.tf`

`subnet_ids` entrega la lista de IDs, no los CIDR. En el root se consulta mediante `module.red.subnet_ids`.

```hcl
output "vpc_id" {
  value = aws_vpc.principal.id
}

output "subnet_ids" {
  value = aws_subnet.subredes[*].id
}
```

## Módulo NLB — `modulos/nlb/`

Se utiliza si piden balanceadores. En la formativa se crearon dos internos, cada uno en una subred. No tienen listeners ni target groups, así que estar creados no significa que distribuyan tráfico.

### `variables.tf`

El hijo recibe el nombre base, la cantidad y las subredes entregadas por VPC.

```hcl
variable "nombre" {
  type = string
}

variable "cantidad" {
  type = number
}

variable "subnet_ids" {
  type = list(string)
}
```

### `main.tf`

`load_balancer_type = "network"` define el NLB y `internal = true` lo mantiene interno. `subnets` es una lista incluso cuando cada NLB utiliza una sola subred. `substr` acorta el nombre para respetar el límite de AWS.

```hcl
locals {
  nombre_nlb = substr(var.nombre, 0, 20)
}

resource "aws_lb" "balanceadores" {
  count = var.cantidad

  name = "${local.nombre_nlb}-nlb-${count.index + 1}"

  internal           = true
  load_balancer_type = "network"

  subnets = [
    var.subnet_ids[count.index % length(var.subnet_ids)]
  ]

  enable_deletion_protection = false
}
```

### `outputs.tf`

El DNS y el ARN permiten identificar los NLB creados. La raíz publica el DNS.

```hcl
output "nlb_dns" {
  value = aws_lb.balanceadores[*].dns_name
}

output "nlb_arns" {
  value = aws_lb.balanceadores[*].arn
}
```

## Módulo S3 — `modulos/s3/`

Los buckets no se ubican en subredes. Este módulo funciona de forma independiente de VPC; recibe únicamente un nombre base y una cantidad. La formativa exigía bloqueo de acceso público.

### `variables.tf`

No necesita variables de red. `cantidad` controla el número de buckets.

```hcl
variable "nombre" {
  type = string
}

variable "cantidad" {
  type = number
}
```

### `main.tf`

El primer recurso crea los buckets y el segundo aplica las cuatro opciones de Public Access Block. Ambos utilizan el mismo índice para que cada bloqueo apunte a su bucket. `lower()` mantiene nombres en minúsculas.

```hcl
locals {
  nombre_bucket = lower("${var.nombre}-s3")
}

resource "aws_s3_bucket" "buckets" {
  count = var.cantidad

  bucket = "${local.nombre_bucket}-${count.index + 1}"

  tags = {
    Name = "${local.nombre_bucket}-${count.index + 1}"
  }
}

resource "aws_s3_bucket_public_access_block" "privados" {
  count = var.cantidad

  bucket = aws_s3_bucket.buckets[count.index].id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### `outputs.tf`

Devuelve los nombres reales que luego publica la raíz.

```hcl
output "bucket_names" {
  value = aws_s3_bucket.buckets[*].bucket
}
```

## Cambios según el ejercicio

### Empezar escribiendo EC2

Puedo preparar primero `modulos/ec2` y dejar la conexión de red para después. Si en esa primera etapa debo **desplegarla**, necesito una subred que ya exista en AWS. Para utilizar el hijo EC2 de la formativa sin reescribirlo, una sola subred se entrega como lista de un elemento y `cantidad = 1`.

**`main.tf` raíz, cuando solo se trabaja EC2** (sustituye las cuatro llamadas de la formativa por esta):

```hcl
module "computo" {
  source = "./modulos/ec2"

  nombre         = var.nombre_proyecto
  ami_id         = var.ami_ec2
  tipo_instancia = var.tipo_ec2
  cantidad       = 1
  subnet_ids     = [var.subred_existente]
}
```

**`variables.tf` raíz:** conservo las variables `nombre_proyecto`, `ami_ec2` y `tipo_ec2`; añado la siguiente y retiro las variables de VPC/NLB/S3 que no se usan.

```hcl
variable "subred_existente" {
  type = string
}
```

**`terraform.tfvars`:** junto a `ami_ec2`, asigno `subred_existente = "subnet-ID_REAL"` con el ID del laboratorio. **`outputs.tf` raíz:** conservo los outputs EC2 y retiro los que refieren a módulos que no llamé. `providers.tf` puede mantenerse sin cambios.

Cuando incorporo VPC en un proyecto nuevo, dejo de usar `subred_existente` y paso `subnet_ids = module.red.subnet_ids`. Eso exige agregar la llamada `module "red"` y las variables de red. Si EC2 ya fue desplegada en otra subred, el cambio puede requerir reemplazarla; lo reviso en `plan`.

### VPC sin salida pública o con salida pública

**Solo VPC y subredes:** se mantienen `aws_vpc` y `aws_subnet` del módulo de red. No agrego Internet Gateway, rutas ni asociaciones. AWS asigna una tabla principal a la VPC y las subredes sin asociación explícita utilizan esa tabla.

**Si piden subredes públicas con salida a Internet:** agrego al final de `modulos/vpc/main.tf` estos recursos. Reutilizan la VPC y las subredes ya declaradas. No se necesitan variables nuevas para esta ruta fija.

```hcl
resource "aws_internet_gateway" "salida" {
  vpc_id = aws_vpc.principal.id
}

resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id
}

resource "aws_route" "hacia_internet" {
  route_table_id         = aws_route_table.publica.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.salida.id
}

resource "aws_route_table_association" "publicas" {
  count = length(var.subnet_cidrs)

  subnet_id      = aws_subnet.subredes[count.index].id
  route_table_id = aws_route_table.publica.id
}
```

La tabla `publica` es una tabla personalizada. Llamar `main` a un recurso no lo convierte en la tabla principal de AWS. La ruta hacia el gateway define el carácter público de la subred; para una EC2 con conectividad IPv4 a Internet también se necesita una IP pública y reglas de seguridad apropiadas.

**Si solo piden una subred:** en `variables.tf` raíz dejo un CIDR y una zona en las respectivas listas; `count` generará una sola subred. Si piden **solo VPC, sin ninguna subred**, conservo `aws_vpc.principal` y retiro `aws_subnet.subredes`, las variables `subnet_cidrs` y `azs` del hijo, sus argumentos en `module "red"`, el output `subnet_ids` y las variables y outputs de subred de la raíz. También retiro las llamadas a EC2/NLB si requieren subredes que ya no existirán.

Si el profesor pide que la subred **asigne IP pública automáticamente**, agrego `map_public_ip_on_launch = true` dentro de `aws_subnet`. Es una opción distinta de la ruta pública: la ruta se configura en la tabla y la asignación de IP se configura en la subred o en la EC2. En la formativa, EC2 tiene `associate_public_ip_address = false`, así que no solicita una IP pública aunque la subred tenga esa opción.

### Varias EC2 y `for_each`

En la formativa `cantidad_ec2 = 2` genera dos instancias. Para tres, puedo sobrescribir `cantidad_ec2 = 3` en `terraform.tfvars`; la tercera vuelve a la primera subred.

Si las instancias se identifican por nombres (`web`, `app`) en vez de índices, considero `for_each`. No se combina con `count` en un mismo recurso. Esta es una **alternativa** al bloque `aws_instance` del hijo, no un bloque adicional:

```hcl
resource "aws_instance" "servidores" {
  for_each = var.subred_por_servidor

  ami                         = var.ami_id
  instance_type               = var.tipo_instancia
  subnet_id                   = each.value
  associate_public_ip_address = false

  tags = {
    Name = "${var.nombre}-${each.key}"
  }
}
```

En `modulos/ec2/variables.tf` se reemplazan `cantidad` y `subnet_ids` por:

```hcl
variable "subred_por_servidor" {
  type = map(string)
}
```

En la llamada `module "computo"` de la raíz se reemplazan los argumentos `cantidad` y `subnet_ids` por:

```hcl
subred_por_servidor = {
  web = module.red.subnet_ids[0]
  app = module.red.subnet_ids[1]
}
```

Los outputs del hijo pasan a usar mapas, porque ahora las instancias se identifican por clave:

```hcl
output "instance_ids" {
  value = { for nombre, ec2 in aws_instance.servidores : nombre => ec2.id }
}

output "private_ips" {
  value = { for nombre, ec2 in aws_instance.servidores : nombre => ec2.private_ip }
}
```

La raíz puede conservar `module.computo.instance_ids` y `module.computo.private_ips`, pero recibe mapas en lugar de listas. Cambiar de `count` a `for_each` sobre infraestructura ya aplicada modifica las direcciones del estado.

### Dejar fuera NLB o S3

Si solo piden VPC y EC2, elimino de `main.tf` raíz las llamadas `module "balanceo"` y `module "almacenamiento"`; retiro también `cantidad_nlb` y `cantidad_s3` de las variables raíz y sus outputs. Las carpetas de esos hijos pueden permanecer guardadas: Terraform no administra un módulo que la raíz no invoca.

Para agregarlos después, restauro sus llamadas del `main.tf` raíz. NLB recibe `module.red.subnet_ids`; S3 no necesita entradas de VPC. Evito crear una dependencia entre ambos si el ejercicio no la requiere.

## `locals`, provisioners y debugging

`locals` sirve para reutilizar un valor calculado dentro del mismo módulo. Por ejemplo, en la raíz `local.nombre` une el prefijo con el sufijo de Random. No es una variable global: cada módulo tiene sus propios `locals`.

`local-exec` ejecuta en el equipo donde está funcionando Terraform. Si se agrega este bloque **dentro de `aws_instance`**, registra el ID al crear una nueva instancia en Windows:

```hcl
provisioner "local-exec" {
  interpreter = ["PowerShell", "-Command"]
  command     = "Add-Content -Path ec2-creadas.txt -Value ${self.id}"
}
```

`remote-exec` ejecuta comandos en la EC2 a través de SSH. También se coloca dentro de `aws_instance`, acompañado de `connection`:

```hcl
connection {
  type        = "ssh"
  host        = self.public_ip
  user        = "ec2-user"
  private_key = file(var.ruta_clave_privada)
  timeout     = "5m"
}

provisioner "remote-exec" {
  inline = [
    "sudo dnf install -y httpd",
    "sudo systemctl enable --now httpd"
  ]
}
```

Este último ejemplo **no funciona directamente en nuestra EC2 privada**. Necesita acceso SSH desde Windows, un Key Pair asociado a la instancia, un Security Group que permita TCP 22 desde un origen autorizado y rutas apropiadas. Además, hay que declarar el dato que utiliza `file()`:

**Hijo EC2, `variables.tf`:**

```hcl
variable "ruta_clave_privada" {
  type = string
}
```

**Raíz, `variables.tf`:**

```hcl
variable "ruta_pem" {
  type = string
}
```

**Raíz, en `module "computo"`:** `ruta_clave_privada = var.ruta_pem`. En `terraform.tfvars`, `ruta_pem` recibe la ruta real al `.pem` en Windows, sin subir ese archivo al repositorio. También debo configurar el `key_name` correcto en EC2; la ruta del archivo privado por sí sola no asocia la clave con la instancia. Los provisioners suelen ejecutarse durante la creación, no en cada `apply` sin cambios.

En PowerShell, desde la raíz, activo el registro cuando necesito más información:

```powershell
$env:TF_LOG="DEBUG"
$env:TF_LOG_PATH="$PWD\terraform-debug.log"
terraform plan
Test-Path .\terraform-debug.log
Remove-Item Env:TF_LOG
Remove-Item Env:TF_LOG_PATH
```

## Ejecución y comprobación

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

`validate` comprueba la configuración; `plan` presenta creaciones, modificaciones y reemplazos. Si el plan corresponde al ejercicio, ejecuto `terraform apply`, consulto `terraform output` y comparo en AWS Console los recursos efectivamente creados. `terraform state list` indica lo que está registrado en el estado, no si un servicio funciona.

En la formativa anterior, `ExpiredToken` se resolvió renovando las credenciales. S3 llegó a crear buckets, aunque AWS Academy bloqueó una lectura por SCP. El `plan -refresh=false` no reemplaza la comprobación de recursos reales.

Al subir apuntes a GitHub, excluyo `.terraform/`, `terraform.tfstate`, sus respaldos, logs, claves y credenciales. El estado puede contener información sensible.
