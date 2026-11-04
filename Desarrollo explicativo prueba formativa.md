# Terraform — Formativa de Infraestructura como Código I

Este proyecto corresponde a la formativa de Terraform trabajada en AWS Academy. Los archivos están separados por servicio y conectados desde la raíz. La organización permite revisar cada parte sin mezclar la configuración de red, las instancias, los balanceadores y el almacenamiento.

La configuración incluye una VPC `10.0.0.0/16`, dos subredes (`10.0.1.0/24` y `10.0.2.0/24`), dos EC2, dos NLB internos y dos buckets S3 con bloqueo de acceso público. También utiliza Random para generar un sufijo de seis caracteres, además de `locals`, `count`, variables y outputs. La configuración de subredes públicas con Internet Gateway y rutas corresponde a otra práctica y no está incluida aquí.

## Organización del proyecto

Cada carpeta constituye un módulo. Terraform lee sus archivos `.tf` en conjunto: `variables.tf` reúne las entradas, `main.tf` define los recursos y `outputs.tf` deja disponibles los datos que otros módulos pueden utilizar. Los bloques `variable`, `resource`, `module` y `output` son parte de la sintaxis de Terraform; los nombres que van entre comillas los elegimos según su función.

Por ejemplo, la llamada al módulo EC2 se escribe en el `main.tf` raíz. La etiqueta `computo` identifica esa llamada, mientras que `source` señala la carpeta que contiene el módulo:

```hcl
module "computo" {
  source = "./modulos/ec2"

  nombre         = local.nombre
  ami_id         = var.ami_ec2
  tipo_instancia = var.tipo_ec2
  cantidad       = var.cantidad_ec2
  subnet_ids     = module.red.subnet_ids
}
```

`ami_id` es el nombre de una entrada del módulo EC2, mientras que `var.ami_ec2` es una variable declarada en la raíz. La misma relación aparece en `cantidad = var.cantidad_ec2`. Los IDs de subred llegan del output `subnet_ids` de VPC, no de valores escritos manualmente.

### Archivos y carpetas

```text
proyecto-formativa/
  providers.tf
  variables.tf
  terraform.tfvars
  main.tf
  outputs.tf
  modulos/
    vpc/  variables.tf, main.tf, outputs.tf
    ec2/  variables.tf, main.tf, outputs.tf
    nlb/  variables.tf, main.tf, outputs.tf
    s3/   variables.tf, main.tf, outputs.tf
```

Trabajo desde `proyecto-formativa/`, la carpeta que contiene las llamadas a los cuatro módulos. Dentro de `modulos/`, VPC administra la red, EC2 las instancias, NLB los balanceadores y S3 los buckets. Cada carpeta conserva sus entradas, recursos y salidas. Terraform se ejecuta desde la raíz para incluir todo el proyecto.

## Módulo raíz — `proyecto-formativa/`

En la raíz se configura el provider, se declaran los parámetros generales y se conectan los módulos. `terraform.tfvars` reúne los valores concretos; `main.tf` contiene las llamadas y `outputs.tf` permite consultar los resultados del conjunto. Los recursos de AWS quedan definidos en las subcarpetas correspondientes.

### `providers.tf`

`providers.tf` declara AWS y Random. AWS permite administrar los servicios del ejercicio y Random genera el sufijo utilizado en los nombres. Configuro `us-east-1` en la raíz para que los módulos usen la misma región sin repetir el bloque `provider` en cada carpeta.

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

`source = "hashicorp/aws"` indica de dónde se obtiene el provider; `version = "~> 6.0"` permite versiones de la serie 6.x. La región `us-east-1` debe ser coherente con la AMI y con las zonas de disponibilidad. Aquí no coloco claves de AWS: las credenciales temporales se cargan en la sesión PowerShell del laboratorio.

### `variables.tf`

`variables.tf` contiene los parámetros generales: CIDR de la VPC, rangos de subredes, zonas de disponibilidad, AMI, tipo de EC2 y cantidades. Los nombres indican qué parte del proyecto se modifica. En `main.tf`, estos valores pasan a las entradas de cada módulo hijo, que pueden tener nombres más breves.

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

Los valores se utilizan de la siguiente manera:

`nombre_proyecto` da el prefijo común. El sufijo de Random se agregará en `main.tf`. `cidr_vpc` es el rango total de la red y `cidrs_subredes` contiene dos rangos que caben dentro de él y no se superponen. `zonas_disponibilidad` guarda las zonas correspondientes en el mismo orden. Si agrego una tercera subred, debo tener también una tercera zona; de lo contrario, el código intentará acceder a una posición inexistente.

`ami_ec2` identifica la imagen del sistema operativo. No tiene `default`, así que debe recibir un valor antes del plan; en este caso lo escribiré en `terraform.tfvars`. `tipo_ec2` determina el tipo de instancia. Las tres variables `cantidad_*` controlan las repeticiones que ejecutan los hijos mediante `count`.

Un `default` es un valor inicial, no una obligación: puedo sobrescribirlo desde `terraform.tfvars`. Por ejemplo, para tres EC2 basta con escribir `cantidad_ec2 = 3` en ese archivo. No es necesario duplicar un bloque `resource`.

### `terraform.tfvars`

En `terraform.tfvars` asigno la AMI a `ami_ec2`, que está declarada sin `default` en la raíz. También puedo utilizar este archivo para cambiar una cantidad o un tipo de instancia sin modificar su declaración en `variables.tf` ni el recurso de EC2.

```hcl
ami_ec2 = "ami-04b938437b69a2c9c"
```

La AMI de la formativa corresponde a Amazon Linux 2023 en `us-east-1`. En otro laboratorio debo confirmar que exista y sea compatible con el tipo elegido. No todas las AMI sirven en todas las regiones.

### `main.tf`

`main.tf` reúne las llamadas a los cuatro módulos y genera el nombre común con Random. Desde aquí se pasan los CIDR y zonas a VPC, y se recuperan los IDs de sus subredes para EC2 y NLB. Los bloques `aws_vpc`, `aws_instance` y `aws_lb` permanecen en los módulos hijos; la raíz solo les entrega las entradas necesarias.

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

Primero se calcula `local.nombre`: toma `var.nombre_proyecto` y concatena el sufijo obtenido de `random_string.sufijo.result`. Ese nombre llega a los cuatro hijos.

En `module "red"`, el argumento `cidr` recibe `var.cidr_vpc`. El nombre `cidr` debe existir como entrada en el hijo VPC; en cambio, `cidr_vpc` solo tiene que existir en la raíz. Esta es una de las relaciones de nombres que más conviene comprobar.

La conexión central está en `subnet_ids = module.red.subnet_ids`. El módulo VPC expone los IDs de las subredes; la raíz toma ese output y se lo entrega tanto a `computo` como a `balanceo`. No escribo manualmente ningún `subnet-...`: esos identificadores los asigna AWS y Terraform los obtiene mediante referencias.

El módulo S3 no recibe las subredes porque un bucket no se crea dentro de una VPC. Mantener esa separación permite comprender qué depende realmente de la red y qué no.

### `outputs.tf`

`outputs.tf` reúne los valores que consultaré desde la raíz: VPC, subredes, EC2, NLB y buckets. Cada resultado procede de un output del módulo correspondiente. Este archivo no crea recursos; solo expone sus atributos mediante referencias.

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

`module.computo.instance_ids` significa: buscar el output `instance_ids` del módulo llamado `computo`. No significa buscar un archivo con ese nombre ni consultar una variable del root. `module.red.subnet_ids` se interpreta de la misma forma. No hace falta que el output de la raíz se llame igual al del hijo.

## Módulos hijos — `modulos/`

Cada módulo hijo mantiene sus propios parámetros, recursos y resultados. Aunque VPC y EC2 tengan una entrada llamada `nombre`, cada `var.nombre` pertenece únicamente a su carpeta. Para compartir valores entre módulos se utilizan las llamadas y los outputs de la raíz; no existe una variable global que los comunique automáticamente.

### VPC — `modulos/vpc/`

La subcarpeta `vpc/` reúne únicamente los recursos de red. La mantengo aparte porque EC2 y NLB necesitan las subredes, pero no deben encargarse de construirlas. En la raíz, el bloque `module "red"` apunta a esta carpeta mediante `source = "./modulos/vpc"`. La raíz define los CIDR y las zonas; VPC crea los recursos y devuelve sus IDs para que puedan utilizarse en los otros módulos.

#### `modulos/vpc/variables.tf`

Las entradas de VPC son el nombre, el CIDR general, los CIDR de las subredes y las zonas. La llamada `module "red"` de la raíz entrega estos valores. `cidr` es el argumento que espera el hijo; `var.cidr_vpc` identifica el valor que proporciona la raíz.

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

`nombre` viene de `local.nombre` en el root. `cidr` recibe `var.cidr_vpc`. `subnet_cidrs` y `azs` son listas de texto: sus posiciones se corresponden. Las variables del hijo no incluyen valores fijos para que pueda reutilizar la misma carpeta con otra red.

#### `modulos/vpc/main.tf`

Con las entradas declaradas, este archivo realiza el trabajo propio de VPC: crea una red y dos subredes dentro de ella. El CIDR de la red llega por `var.cidr`; las subredes y sus zonas llegan como listas. `count` evita escribir dos bloques casi iguales. No agrego EC2 aquí, porque VPC solo debe construir la red. La conexión con las instancias se hará más adelante desde el `main.tf` raíz, usando los IDs que este módulo devuelve.

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

El recurso `aws_vpc.principal` crea la VPC con `cidr_block = var.cidr`. Las subredes usan `vpc_id = aws_vpc.principal.id`: así Terraform conoce tanto la VPC a la que pertenecen como la dependencia necesaria para crearlas.

`count = length(var.subnet_cidrs)` crea una subred por cada CIDR de la lista. Con `count.index = 0` toma el primer CIDR y la primera zona; con `count.index = 1` toma el segundo par. Los nombres se diferencian mediante `count.index + 1`. El `local.nombre_vpc` solo prepara la etiqueta del recurso y no necesita salir del módulo.

Este módulo no agrega Internet Gateway ni rutas públicas, porque no forman parte de los recursos exigidos en la formativa. Crear subredes por sí solo no implica que tengan salida a Internet.

#### `modulos/vpc/outputs.tf`

Aquí se define qué información de la red puede utilizar el resto del proyecto. Para la formativa necesitamos los IDs que AWS asigne a las subredes, porque ni EC2 ni NLB pueden ubicarse sin conocerlas. El output `subnet_ids` devuelve una lista y la raíz lo consulta como `module.red.subnet_ids`. También expongo el ID de VPC para poder identificarla desde la raíz. Sin estos outputs, los otros módulos no tendrían acceso a esos atributos mediante las llamadas actuales.

```hcl
output "vpc_id" {
  value = aws_vpc.principal.id
}

output "subnet_ids" {
  value = aws_subnet.subredes[*].id
}
```

`vpc_id` devuelve el identificador de la VPC. `subnet_ids` devuelve **una lista** con los IDs de todas las subredes, gracias a `[*].id`. Esto es lo que luego aparece en la raíz como `module.red.subnet_ids`. EC2 y NLB reciben la lista sin necesitar conocer los nombres internos `aws_subnet.subredes`.

### EC2 — `modulos/ec2/`

La subcarpeta `ec2/` concentra la creación de instancias y sus resultados. La separo de VPC para que el código EC2 se pueda reutilizar en otra red: en lugar de construir subredes, recibe sus IDs. En el `main.tf` de la raíz se invoca como `module "computo"`, cuya ruta es `./modulos/ec2`. Su entrada más importante para la conexión entre módulos es `subnet_ids = module.red.subnet_ids`.

#### `modulos/ec2/variables.tf`

EC2 recibe nombre, AMI, tipo de instancia, cantidad e IDs de subred. La raíz entrega estos valores mediante `module "computo"`. `subnet_ids` contiene identificadores asignados por AWS, no rangos CIDR. Si cambia la red, se puede conservar el módulo EC2 y cambiar la lista que recibe.

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

`nombre` sirve para las etiquetas. `ami_id` recibe el identificador de la imagen. `tipo_instancia` recibe `t3.micro` desde `var.tipo_ec2` de la raíz. `cantidad` determina cuántas EC2 tendrá este módulo. `subnet_ids` es una lista porque en la formativa hay dos subredes y queremos distribuir las instancias entre ellas.

Aunque el root use `cantidad_ec2`, la entrada del hijo es `cantidad`: la relación la establece la línea `cantidad = var.cantidad_ec2` del `main.tf` raíz. Si escribo otro argumento en el root, debo declarar el nombre correspondiente aquí.

#### `modulos/ec2/main.tf`

Este es el archivo que construye realmente las EC2. Toma los parámetros recibidos y los asigna a los argumentos exigidos por `aws_instance`: `ami`, `instance_type` y `subnet_id`. Como la formativa requiere dos instancias, `count` repite el recurso y la expresión con `%` selecciona las subredes de la lista. El `local` prepara el nombre de cada instancia. Observa que aquí se usa `var.subnet_ids`, no `module.red.subnet_ids`: EC2 no conoce las llamadas hechas por la raíz, solo los datos que recibe.

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

`ami` e `instance_type` son argumentos propios del recurso AWS, mientras que `var.ami_id` y `var.tipo_instancia` son las entradas que definimos en el hijo. No tienen que llamarse exactamente igual.

`count = var.cantidad` crea los recursos `[0]` y `[1]`. La selección de subred usa `count.index % length(var.subnet_ids)`: con dos EC2 y dos subredes, cada instancia toma una distinta; si hay tres EC2, la tercera vuelve a la primera. La lista de subredes no debe estar vacía.

`associate_public_ip_address = false` mantiene las instancias sin IP pública. No cambia la subred donde están; solo la asignación de dirección pública. La etiqueta `Name` combina el nombre base y el índice para reconocer cada servidor en AWS.

#### `modulos/ec2/outputs.tf`

Después de crear las instancias, AWS les asigna identificadores e IP privadas. Este archivo los devuelve para que el módulo raíz pueda consultarlos. `instance_ids` y `private_ips` son outputs internos del hijo; en la raíz los leo mediante `module.computo.instance_ids` y `module.computo.private_ips`. No tengo que escribir manualmente los IDs ni intentar adivinar las direcciones: Terraform recupera los atributos de los recursos que acaba de administrar.

```hcl
output "instance_ids" {
  value = aws_instance.servidores[*].id
}

output "private_ips" {
  value = aws_instance.servidores[*].private_ip
}
```

Los IDs y las IP privadas son valores asignados por AWS después de crear las EC2. `[*].id` obtiene un ID por instancia y `[*].private_ip` obtiene sus IP privadas. La raíz los puede leer como `module.computo.instance_ids` y `module.computo.private_ips`.

### NLB — `modulos/nlb/`

Esta subcarpeta existe para mantener la configuración de los balanceadores separada tanto de la red como de las instancias. El `main.tf` raíz la incorpora mediante `module "balanceo"`, con `source = "./modulos/nlb"`, y le entrega las subredes que creó VPC. Así NLB puede cambiar su cantidad o sus características sin obligarme a editar el módulo VPC ni copiar recursos `aws_lb` en la raíz.

#### `modulos/nlb/variables.tf`

NLB recibe el nombre base, la cantidad y los IDs de subred. Se llama desde la raíz como `balanceo` y utiliza las subredes publicadas por VPC. La AMI o el tipo de EC2 no tienen relación con su configuración.

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

El nombre base llega desde la raíz; `cantidad` controla cuántos NLB se crean y `subnet_ids` recibe las dos subredes de VPC. No necesito otra variable para la VPC, ya que cada subred tiene una VPC asociada.

#### `modulos/nlb/main.tf`

Con esas entradas, el recurso `aws_lb` crea dos Network Load Balancers internos. Utiliza `count` para repetirlos y selecciona una subred por balanceador de la lista que recibe. El nombre se prepara con `locals` para mantenerlo dentro del límite del servicio. En esta formativa el objetivo es crear los NLB, por eso este archivo no incluye listeners ni grupos de destino: añadirlos supondría una actividad diferente. La raíz solo coordina la llamada; no contiene los bloques `aws_lb`.

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

`load_balancer_type = "network"` determina el tipo NLB; `internal = true` lo deja interno. El argumento `subnets` espera una colección, incluso cuando cada balanceador de esta práctica recibe una sola subred. Por eso aparece entre corchetes `[...]`.

`substr(var.nombre, 0, 20)` limita el prefijo antes de agregar `-nlb-1` o `-nlb-2`, porque AWS tiene un límite para el nombre del balanceador. `count` y el índice permiten crear los dos NLB sin escribir dos recursos completos.

#### `modulos/nlb/outputs.tf`

Cuando AWS crea cada NLB, le asigna un nombre DNS y un ARN. Este archivo expone ambos atributos para que la raíz pueda utilizarlos sin buscar los valores manualmente. En la versión actual, el `outputs.tf` de la raíz recupera `module.balanceo.nlb_dns`. El ARN queda disponible si más adelante necesito mostrarlo. Estos outputs sirven para identificar recursos; por sí solos no prueban que haya tráfico balanceado.

```hcl
output "nlb_dns" {
  value = aws_lb.balanceadores[*].dns_name
}

output "nlb_arns" {
  value = aws_lb.balanceadores[*].arn
}
```

El DNS permite identificar el nombre asignado por AWS al balanceador. El ARN es el identificador completo del recurso. En la raíz exponemos el DNS; si necesito también los ARN, puedo agregar allí otro output que use `module.balanceo.nlb_arns`.

### S3 — `modulos/s3/`

S3 queda en su propia subcarpeta porque administra almacenamiento de objetos, no servidores ni subredes. La raíz lo incorpora mediante `module "almacenamiento"` y la ruta `./modulos/s3`. En esta formativa solo necesita un nombre base y una cantidad para crear dos buckets con bloqueo público. Mantenerlo independiente también evita una dependencia falsa con VPC: sus recursos no necesitan un `subnet_id`.

#### `modulos/s3/variables.tf`

S3 recibe un nombre base y la cantidad de buckets. La raíz lo llama como `almacenamiento`; no hay una entrada de subred porque los buckets no se crean dentro de la VPC.

```hcl
variable "nombre" {
  type = string
}

variable "cantidad" {
  type = number
}
```

El root entrega `nombre = local.nombre` y `cantidad = var.cantidad_s3`. No agrego AMI, VPC ni zona porque no se usan para crear el bucket.

#### `modulos/s3/main.tf`

En este archivo se crean los dos buckets y sus bloques de acceso público. El primer recurso utiliza `count` para generar nombres distintos a partir del prefijo recibido. El segundo recurso usa el mismo índice para aplicar el bloqueo al bucket correspondiente. Esa referencia es importante porque relaciona cada configuración de seguridad con su bucket sin tener que escribir el nombre otra vez. Todo ocurre dentro de S3; el módulo raíz únicamente le indica cuántos crear y qué nombre base utilizar.

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

`lower()` convierte el nombre a minúsculas. El índice añade `-1` y `-2` para distinguir los buckets. El segundo recurso, `aws_s3_bucket_public_access_block`, toma el ID del bucket del mismo índice. Esa referencia es la que vincula cada bloqueo con el bucket que corresponde.

Las cuatro opciones en `true` bloquean las formas habituales de acceso público administradas por este recurso. No son equivalentes a prohibir todo acceso: las políticas IAM siguen siendo otro aspecto de seguridad.

#### `modulos/s3/outputs.tf`

El último archivo del módulo S3 devuelve los nombres que quedaron asignados a los buckets. Como se crearon con `count`, el resultado es una lista. La raíz la consulta mediante `module.almacenamiento.bucket_names` y la expone como `nombres_buckets`. Así puedo obtener los nombres desde `terraform output` después del despliegue, sin copiar información de la consola AWS.

```hcl
output "bucket_names" {
  value = aws_s3_bucket.buckets[*].bucket
}
```

El output `bucket_names` devuelve los nombres reales de los dos buckets. En el root se presenta como `nombres_buckets`, usando `module.almacenamiento.bucket_names`.

## Referencias entre archivos

La AMI se configura en `terraform.tfvars` como `ami_ec2`. Desde `main.tf` raíz, `ami_id = var.ami_ec2` la entrega al módulo EC2, que la recibe mediante `variable "ami_id"` y la utiliza en `ami = var.ami_id`.

VPC devuelve los IDs de las subredes mediante `subnet_ids`. La raíz los recupera como `module.red.subnet_ids` y los pasa a `computo`. El hijo EC2 los utiliza con `var.subnet_ids`. No se consulta `module.red` dentro de EC2, porque esa llamada está declarada en la raíz.

El módulo EC2 devuelve los IDs como `instance_ids`. La raíz consulta `module.computo.instance_ids`, lo expone como `ids_ec2` y permite leerlo con `terraform output ids_ec2`.

Ante un error de referencia, reviso dónde se declara el valor. `Unsupported attribute` suele señalar un atributo u output que no existe en ese contexto; `Unsupported argument` puede indicar que la llamada envía un argumento que el módulo hijo no declaró.
