# Terraform — EC2 y conexión con VPC

En la clase trabajamos una EC2 por separado y después incorporamos una VPC propia. Las dos configuraciones están en este archivo porque permiten ver qué cambia al pasar de una subred existente a una creada mediante Terraform. Más adelante se utiliza `count` para distribuir dos instancias y se muestran ajustes puntuales sobre el mismo módulo.

Cada versión tiene sus archivos correspondientes. Cuando cambia una variable de entrada, también hay que revisar la llamada del `main.tf` raíz; por eso las configuraciones se presentan completas donde corresponde.

## Antes de configurar los módulos

El trabajo se hace en VS Code, desde la terminal PowerShell de la carpeta raíz. Para operar con AWS Academy inicio el laboratorio y cargo sus credenciales temporales en esa terminal. No es una conexión directa entre VS Code y AWS: Terraform utiliza las credenciales de la sesión para que el provider AWS haga las solicitudes.

```powershell
terraform -version
$env:AWS_ACCESS_KEY_ID="VALOR_DEL_LAB"
$env:AWS_SECRET_ACCESS_KEY="VALOR_DEL_LAB"
$env:AWS_SESSION_TOKEN="VALOR_DEL_LAB"
$env:AWS_DEFAULT_REGION="us-east-1"
```

La región del provider y los identificadores utilizados en el proyecto deben corresponder al laboratorio activo. Si se renuevan las credenciales, basta con actualizar las variables de entorno en PowerShell. La configuración de los archivos viene después; la ejecución de `init`, `plan` y `apply` se revisa al final.

## 1. EC2 en una subred existente

Para comenzar utilicé una sola EC2. Su módulo recibe el nombre, la AMI, el tipo de instancia y el ID de una subred que ya exista en AWS. El bloque `aws_instance` queda dentro de `modulos/ec2`, mientras que en la raíz se definen los valores. Con una instancia no hace falta utilizar `count`.

En la carpeta principal están `providers.tf`, `variables.tf`, `terraform.tfvars`, `main.tf` y `outputs.tf`. El hijo `modulos/ec2/` contiene sus propios `variables.tf`, `main.tf` y `outputs.tf`; ahí queda el recurso de la instancia, no en la raíz.

### Módulo raíz — `providers.tf`

La región se configura en el root, en este caso `us-east-1`. Como EC2 usa el mismo provider AWS, no hace falta declarar otro dentro de `modulos/ec2`.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Módulo raíz — `variables.tf`

Aquí quedan los valores generales: nombre del proyecto, AMI, tipo de instancia y subred elegida. De esta forma el código de `aws_instance` no depende de un identificador escrito directamente en el módulo hijo.

```hcl
variable "nombre_proyecto" {
  type    = string
  default = "iac-practica"
}

variable "ami_ec2" {
  type = string
}

variable "tipo_ec2" {
  type    = string
  default = "t3.micro"
}

variable "subred_ec2_inicial" {
  type = string
}
```

`subred_ec2_inicial` guarda un único ID, por eso es `string`. En la llamada de EC2 ese dato pasa a llamarse `subnet_id`, tal como está declarado en el `variables.tf` del hijo.

### Módulo raíz — `terraform.tfvars`

Los valores concretos de AMI y subred quedan en `terraform.tfvars`. La subred debe existir en la cuenta donde esté activo el laboratorio.

```hcl
ami_ec2            = "ami-04b938437b69a2c9c"
subred_ec2_inicial = "subnet-REEMPLAZAR_POR_ID_REAL"
```

El texto `subnet-REEMPLAZAR_POR_ID_REAL` ocupa el lugar del ID verdadero. Lo obtengo en AWS Console, VPC → Subnets. Antes del plan también compruebo que la AMI corresponda a `us-east-1`.

### Módulo raíz — `main.tf`

En `main.tf` raíz se indica dónde está el módulo EC2 y qué valores recibe. `source` apunta a la carpeta `modulos/ec2`; el resto de las líneas corresponden a las entradas declaradas dentro de ese módulo.

```hcl
module "computo" {
  source = "./modulos/ec2"

  nombre         = var.nombre_proyecto
  ami_id         = var.ami_ec2
  tipo_instancia = var.tipo_ec2
  subnet_id      = var.subred_ec2_inicial
}
```

En `subnet_id = var.subred_ec2_inicial`, la izquierda es el argumento que recibe EC2 y la derecha es el valor que tiene la raíz. Los nombres pueden ser distintos; lo que debe coincidir es el argumento con la declaración del hijo.

### Módulo raíz — `outputs.tf`

Los outputs del root recuperan el ID y la IP privada que publica el módulo EC2. Son los valores que aparecerán al consultar `terraform output` después del despliegue.

```hcl
output "id_ec2" {
  value = module.computo.id_instancia
}

output "ip_privada_ec2" {
  value = module.computo.ip_privada
}
```

### Módulo EC2 — `modulos/ec2/variables.tf`

Estas son las entradas propias del hijo EC2. Los cuatro nombres deben corresponder con los argumentos utilizados en la llamada `module "computo"` de la raíz.

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

variable "subnet_id" {
  type = string
}
```

`subnet_id` es un ID individual, no una lista. Tampoco se necesita `cantidad` mientras el recurso se declare una sola vez. Ambos puntos cambiarán en el ejercicio de dos instancias.

### Módulo EC2 — `modulos/ec2/main.tf`

El bloque `aws_instance` define la EC2. La AMI, el tamaño y la subred llegan por variables. `local.etiqueta_ec2` prepara el nombre que quedará en la etiqueta `Name` dentro de AWS.

```hcl
locals {
  etiqueta_ec2 = "${var.nombre}-ec2"
}

resource "aws_instance" "servidor" {
  ami           = var.ami_id
  instance_type = var.tipo_instancia
  subnet_id     = var.subnet_id

  associate_public_ip_address = false

  tags = {
    Name = local.etiqueta_ec2
  }
}
```

La línea `subnet_id = var.subnet_id` utiliza el valor que entregó el root. Puede ser un ID consultado en AWS o uno que Terraform obtenga de otra VPC. En este ejemplo la instancia se deja sin IP pública (`false`).

### Módulo EC2 — `modulos/ec2/outputs.tf`

Con estos outputs recupero los datos que AWS asigna al crear la EC2: el ID y la dirección privada. La llamada `computo` permite leerlos desde el root.

```hcl
output "id_instancia" {
  value = aws_instance.servidor.id
}

output "ip_privada" {
  value = aws_instance.servidor.private_ip
}
```

Como solo hay una EC2, cada output entrega un valor. La referencia es `aws_instance.servidor.id`, sin índices.

## 2. EC2 conectada a la VPC del proyecto

Una vez definido EC2, agregué la carpeta `modulos/vpc`. El módulo EC2 mantiene sus tres archivos: la nueva subred se incorpora cambiando el valor que recibe en el `main.tf` raíz. En vez de escribir su ID en `terraform.tfvars`, se toma del output de VPC.

El esquema de clase considera una VPC con dos subredes, Internet Gateway, tabla pública y dos asociaciones. AWS crea una tabla de rutas principal por defecto; la tabla pública que aparece en el código es otra. Su etiqueta `Name` solo sirve para identificarla.

### Módulo VPC — `modulos/vpc/variables.tf`

En `variables.tf` del hijo VPC declaro el rango de red y los CIDR y zonas de las subredes. En esta versión se trabajan por separado, con los nombres `publica01` y `publica02` utilizados en clase.

```hcl
variable "nombre" {
  type = string
}

variable "cidr" {
  type = string
}

variable "cidr_publica_01" {
  type = string
}

variable "cidr_publica_02" {
  type = string
}

variable "zona_01" {
  type = string
}

variable "zona_02" {
  type = string
}
```

Cada CIDR está emparejado con una zona: `cidr_publica_01` con `zona_01`, y `cidr_publica_02` con `zona_02`. En la formativa completa estos valores se manejaban con listas y `count`; aquí quedan separados para identificar cada subred por su nombre.

### Módulo VPC — `modulos/vpc/main.tf`

El `main.tf` de VPC reúne los recursos de red. Las dos subredes apuntan a la misma VPC y después quedan asociadas a la tabla pública. Su ruta por defecto se dirige al Internet Gateway.

```hcl
locals {
  etiqueta_vpc = "${var.nombre}-vpc"
}

resource "aws_vpc" "principal" {
  cidr_block = var.cidr

  tags = {
    Name = local.etiqueta_vpc
  }
}

resource "aws_subnet" "publica01" {
  vpc_id            = aws_vpc.principal.id
  cidr_block        = var.cidr_publica_01
  availability_zone = var.zona_01

  tags = {
    Name = "${var.nombre}-publica01"
  }
}

resource "aws_subnet" "publica02" {
  vpc_id            = aws_vpc.principal.id
  cidr_block        = var.cidr_publica_02
  availability_zone = var.zona_02

  tags = {
    Name = "${var.nombre}-publica02"
  }
}

resource "aws_internet_gateway" "salida" {
  vpc_id = aws_vpc.principal.id

  tags = {
    Name = "${var.nombre}-igw"
  }
}

resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.salida.id
  }

  tags = {
    Name = "${var.nombre}-rt-publica"
  }
}

resource "aws_route_table_association" "publica01" {
  subnet_id      = aws_subnet.publica01.id
  route_table_id = aws_route_table.publica.id
}

resource "aws_route_table_association" "publica02" {
  subnet_id      = aws_subnet.publica02.id
  route_table_id = aws_route_table.publica.id
}
```

`vpc_id = aws_vpc.principal.id` enlaza cada subred con la VPC creada arriba. La ruta `0.0.0.0/0` indica salida hacia el Internet Gateway y cada asociación enlaza una subred con esa tabla. La ruta puede estar configurada aunque la EC2 no tenga IP pública.

### Módulo VPC — `modulos/vpc/outputs.tf`

En `outputs.tf` dejo publicados el ID de la VPC y los IDs de las dos subredes. Así el root puede seleccionar cuál entregar a EC2 sin escribir un valor fijo.

```hcl
output "id_vpc" {
  value = aws_vpc.principal.id
}

output "s_publica01_id" {
  value = aws_subnet.publica01.id
}

output "s_publica02_id" {
  value = aws_subnet.publica02.id
}
```

`s_publica01_id` corresponde a la primera subred y `s_publica02_id` a la segunda. Cada salida contiene un ID individual.

### Módulo raíz — `variables.tf` con VPC

El `variables.tf` raíz ahora incluye también el direccionamiento y las zonas de VPC. Las variables de AMI y tipo de instancia se mantienen; los CIDR de las subredes deben estar dentro de `10.0.0.0/16` y no superponerse.

```hcl
variable "nombre_proyecto" {
  type    = string
  default = "iac-practica"
}

variable "ami_ec2" {
  type = string
}

variable "tipo_ec2" {
  type    = string
  default = "t3.micro"
}

variable "cidr_vpc" {
  type    = string
  default = "10.0.0.0/16"
}

variable "cidr_subred_01" {
  type    = string
  default = "10.0.1.0/24"
}

variable "cidr_subred_02" {
  type    = string
  default = "10.0.2.0/24"
}

variable "zona_subred_01" {
  type    = string
  default = "us-east-1a"
}

variable "zona_subred_02" {
  type    = string
  default = "us-east-1b"
}
```

La variable `subred_ec2_inicial` se retira de esta versión, porque el ID ya lo entrega el módulo de red.

### Módulo raíz — `main.tf` con VPC y EC2

`module "red"` utiliza la carpeta `modulos/vpc`. En la llamada a `computo`, el argumento `subnet_id` recibe `module.red.s_publica01_id`: el resultado de la primera subred creada en VPC.

```hcl
module "red" {
  source = "./modulos/vpc"

  nombre          = var.nombre_proyecto
  cidr            = var.cidr_vpc
  cidr_publica_01 = var.cidr_subred_01
  cidr_publica_02 = var.cidr_subred_02
  zona_01         = var.zona_subred_01
  zona_02         = var.zona_subred_02
}

module "computo" {
  source = "./modulos/ec2"

  nombre         = var.nombre_proyecto
  ami_id         = var.ami_ec2
  tipo_instancia = var.tipo_ec2
  subnet_id      = module.red.s_publica01_id
}
```

EC2 conserva la entrada `subnet_id` y su recurso sigue igual. La referencia al output vincula la creación de EC2 con la de la subred; Terraform conoce ese orden por la referencia.

### Módulo raíz — `outputs.tf` con VPC y EC2

En los outputs del root quedan a la vista los datos de la red y de la instancia. Cada valor procede del módulo que lo creó.

```hcl
output "id_vpc" {
  value = module.red.id_vpc
}

output "subred_01" {
  value = module.red.s_publica01_id
}

output "subred_02" {
  value = module.red.s_publica02_id
}

output "id_ec2" {
  value = module.computo.id_instancia
}

output "ip_privada_ec2" {
  value = module.computo.ip_privada
}
```

### Módulo raíz — `terraform.tfvars` con VPC

En esta versión `terraform.tfvars` solo necesita la AMI. El direccionamiento y las zonas ya quedaron definidos como valores iniciales en `variables.tf` raíz.

```hcl
ami_ec2 = "ami-04b938437b69a2c9c"
```

Cambiar una EC2 ya creada a otra subred suele implicar reemplazarla; el `plan` permite comprobarlo antes de aplicar. Esto importa si se pasó de una subred existente a la VPC nueva en un mismo estado.

`providers.tf` permanece igual, al igual que los tres archivos de `modulos/ec2`.

## 3. Dos EC2 en subredes diferentes

Para distribuir dos EC2 entre las subredes, la entrada `subnet_id` se cambia por `subnet_ids` y se agrega `cantidad`. El hijo usa `count` para repetir el recurso. Se muestran completos los archivos que cambian; el resto de VPC se mantiene como estaba.

### Módulo VPC — `modulos/vpc/outputs.tf`

En VPC mantengo las dos salidas individuales y agrego `ids_subredes`, que entrega ambos IDs juntos.

```hcl
output "id_vpc" {
  value = aws_vpc.principal.id
}

output "s_publica01_id" {
  value = aws_subnet.publica01.id
}

output "s_publica02_id" {
  value = aws_subnet.publica02.id
}

output "ids_subredes" {
  value = [aws_subnet.publica01.id, aws_subnet.publica02.id]
}
```

El resultado de `ids_subredes` es una lista de dos cadenas, que EC2 puede recorrer para elegir la subred de cada instancia.

### Módulo EC2 — `modulos/ec2/variables.tf`

EC2 ahora recibe dos datos nuevos respecto de la versión de una instancia: `cantidad` y `subnet_ids`. La antigua entrada `subnet_id` se sustituye por una lista.

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

`module.red.ids_subredes` devuelve una lista; `module.red.s_publica01_id` devuelve un único ID. No conviene intercambiarlos: el tipo declarado en EC2 debe corresponder al dato entregado.

### Módulo EC2 — `modulos/ec2/main.tf`

Con `count` declaro las dos instancias en un solo bloque. La expresión de `subnet_id` toma una posición de la lista para cada una.

```hcl
locals {
  etiqueta_ec2 = "${var.nombre}-ec2"
}

resource "aws_instance" "servidor" {
  count = var.cantidad

  ami           = var.ami_id
  instance_type = var.tipo_instancia

  subnet_id = var.subnet_ids[
    count.index % length(var.subnet_ids)
  ]

  associate_public_ip_address = false

  tags = {
    Name = "${local.etiqueta_ec2}-${count.index + 1}"
  }
}
```

`count.index` empieza en cero. Al usar `% length(var.subnet_ids)`, la primera EC2 toma la primera subred, la segunda toma la siguiente y, si agrego otra instancia, vuelve a la primera. La lista debe contener al menos un ID.

### Módulo EC2 — `modulos/ec2/outputs.tf`

Al trabajar con `count`, `[*]` permite obtener los IDs y las IP privadas de todas las EC2.

```hcl
output "ids_instancias" {
  value = aws_instance.servidor[*].id
}

output "ips_privadas" {
  value = aws_instance.servidor[*].private_ip
}
```

Estas salidas ahora devuelven listas. En el ejemplo anterior había un único ID y una única IP privada.

### Raíz — `variables.tf` (cantidad de EC2)

En la raíz agrego `cantidad_ec2`, que controla cuántas instancias solicita el módulo `computo`.

Esta declaración se agrega al `variables.tf` raíz anterior:

```hcl
variable "cantidad_ec2" {
  type    = number
  default = 2
}
```

### Raíz — `main.tf` con dos EC2

La llamada a EC2 recibe dos argumentos nuevos: `cantidad` y `subnet_ids`. La VPC mantiene sus recursos; solo se amplió su output.

```hcl
module "red" {
  source = "./modulos/vpc"

  nombre          = var.nombre_proyecto
  cidr            = var.cidr_vpc
  cidr_publica_01 = var.cidr_subred_01
  cidr_publica_02 = var.cidr_subred_02
  zona_01         = var.zona_subred_01
  zona_02         = var.zona_subred_02
}

module "computo" {
  source = "./modulos/ec2"

  nombre         = var.nombre_proyecto
  ami_id         = var.ami_ec2
  tipo_instancia = var.tipo_ec2
  cantidad       = var.cantidad_ec2
  subnet_ids     = module.red.ids_subredes
}
```

`subnet_ids` coincide con la lista declarada en el hijo y `cantidad` con su variable numérica.

### Raíz — `outputs.tf` con dos EC2

Los resultados de EC2 ahora son listas, mientras que los outputs de VPC permanecen iguales.

```hcl
output "id_vpc" {
  value = module.red.id_vpc
}

output "subred_01" {
  value = module.red.s_publica01_id
}

output "subred_02" {
  value = module.red.s_publica02_id
}

output "ids_ec2" {
  value = module.computo.ids_instancias
}

output "ips_privadas_ec2" {
  value = module.computo.ips_privadas
}
```

La AMI sigue en `terraform.tfvars`. La cantidad inicial es dos, pero puedo establecer `cantidad_ec2 = 3`; en ese caso la tercera EC2 se asigna nuevamente a la primera subred.

---

## 4. Parámetro de IP pública

En el ejercicio base la asignación de IP pública quedó desactivada. Si necesito elegirla sin editar directamente `aws_instance`, puedo convertir esa opción en una variable.

El valor pasa de `false` fijo a `var.ip_publica`. La raíz define `ec2_ip_publica` y lo entrega al hijo como un dato booleano (`true` o `false`).

### Raíz — `variables.tf`

```hcl
variable "ec2_ip_publica" {
  type    = bool
  default = false
}
```

### Raíz — `main.tf`, dentro de `module "computo"`

```hcl
ip_publica = var.ec2_ip_publica
```

### EC2 — `modulos/ec2/variables.tf`

```hcl
variable "ip_publica" {
  type = bool
}
```

### EC2 — `modulos/ec2/main.tf`

```hcl
associate_public_ip_address = var.ip_publica
```

### Raíz — `terraform.tfvars` (valor opcional)

```hcl
ec2_ip_publica = true
```

`ec2_ip_publica` se entrega al hijo como `ip_publica`; luego `aws_instance` utiliza ese valor. Si es `true`, solicita una IP pública. El acceso SSH depende aparte de la red, las reglas de seguridad y las credenciales.

Si vuelvo al valor fijo de la formativa, retiro esta variable de raíz e hijo, quito el argumento de la llamada y restauro `associate_public_ip_address = false` en EC2. Los outputs no necesitan cambios.

## 5. Consultar también la zona de EC2

No hace falta crear una variable para saber en qué zona quedó una instancia: AWS ya entrega ese atributo. Solo agrego una salida en EC2 y otra en la raíz para consultarlo desde PowerShell.

### EC2 — `modulos/ec2/outputs.tf` (con `count`)

```hcl
output "zonas_instancias" {
  value = aws_instance.servidor[*].availability_zone
}
```

### Raíz — `outputs.tf`

```hcl
output "zonas_ec2" {
  value = module.computo.zonas_instancias
}
```

`zonas_instancias` está definido en el hijo y se consulta desde el root mediante `module.computo.zonas_instancias`. No cambia la creación de EC2: solo se muestra un atributo que ya existe.

En la formativa completa el recurso se llama `servidores` y aquí se llama `servidor`. Al trasladar el ejemplo hay que usar el nombre realmente declarado en el `main.tf` de EC2.

## 6. Identificar EC2 con `for_each`

En algunos ejercicios resulta más útil identificar las instancias por nombre. `for_each` permite trabajar con claves como `web` y `app` en lugar de `[0]` y `[1]`. Para utilizarlo cambio la llamada desde la raíz y las variables, el recurso y los outputs del hijo; no se mezcla con `count` en el mismo `aws_instance`.

### Raíz — `main.tf` (llamada EC2 para esta versión)

```hcl
module "computo" {
  source = "./modulos/ec2"

  nombre         = var.nombre_proyecto
  ami_id         = var.ami_ec2
  tipo_instancia = var.tipo_ec2
  instancias = {
    web = module.red.s_publica01_id
    app = module.red.s_publica02_id
  }
}
```

### EC2 — `modulos/ec2/variables.tf`

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

variable "instancias" {
  type = map(string)
}
```

### EC2 — `modulos/ec2/main.tf`

```hcl
locals {
  etiqueta_ec2 = "${var.nombre}-ec2"
}

resource "aws_instance" "servidor" {
  for_each = var.instancias

  ami           = var.ami_id
  instance_type = var.tipo_instancia
  subnet_id     = each.value

  associate_public_ip_address = false

  tags = {
    Name = "${local.etiqueta_ec2}-${each.key}"
  }
}
```

### EC2 — `modulos/ec2/outputs.tf`

```hcl
output "ids_instancias" {
  value = {
    for nombre, instancia in aws_instance.servidor :
    nombre => instancia.id
  }
}

output "ips_privadas" {
  value = {
    for nombre, instancia in aws_instance.servidor :
    nombre => instancia.private_ip
  }
}
```

En el mapa de instancias, `each.key` identifica `web` o `app` y `each.value` contiene el ID de la subred correspondiente. Los outputs ahora son mapas con esas claves, no listas. En la raíz pueden mantenerse los nombres `ids_instancias` e `ips_privadas` porque los outputs del hijo siguen llamándose igual.

Si reemplazo `for_each` por `count`, debo volver a la versión anterior en todos los archivos que dependen de `instancias`. Sobre recursos ya desplegados, este cambio modifica sus direcciones en el estado y hay que revisar `terraform plan` antes de aplicarlo.

## 7. Cambios frecuentes en EC2

**AMI y tamaño.** Si solo cambia la imagen o el tipo de instancia, puedo ajustar `ami_ec2` o `tipo_ec2` en `terraform.tfvars`. No hay que reescribir el hijo, porque esas entradas ya existen. Si el recurso está desplegado, el plan mostrará si el cambio exige reemplazo o puede realizarse de otra manera.

**Cantidad.** En la versión con `count`, `cantidad_ec2` controla cuántos servidores se crean. Al agregar una tercera instancia, la expresión de subred vuelve al primer elemento de la lista.

**Subred individual o lista de subredes.** Con una EC2 la entrada es `subnet_id` de tipo `string`. Al distribuir varias, utilizo `subnet_ids` de tipo `list(string)` y cambio también la llamada de la raíz y la selección de subred en `aws_instance`. Los ejemplos completos de ambas versiones quedaron arriba.

**IP pública y outputs nuevos.** Para hacer configurable la IP pública se necesita una entrada `bool` en EC2 y el argumento que la envía desde la raíz. Si solamente quiero consultar una zona o un ID nuevo, basta con exponer el atributo mediante outputs: no es necesario agregar una variable para un dato que el recurso ya entrega.

Cuando otro módulo necesite información de VPC, primero reviso si el hijo VPC ya tiene el output; después lo paso desde la raíz al módulo que lo utilizará. Es el mismo mecanismo que se utilizó con `subnet_id`.

---

## 8. Ejecución desde PowerShell

Con los archivos de la versión elegida guardados y las credenciales de AWS Academy cargadas, ejecuto Terraform desde la carpeta raíz del proyecto en VS Code:

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

`init` prepara los providers y módulos; `fmt -recursive` da formato a los archivos `.tf`; `validate` comprueba la configuración y `plan` presenta los cambios propuestos. Antes de aplicar reviso las cantidades y cualquier reemplazo, sobre todo si modifiqué una subred o cambié de `count` a `for_each`.

```powershell
terraform apply
terraform output
terraform state list
```

Con `output` consulto los valores definidos en la raíz y con `state list` reviso los recursos que Terraform registra. Luego contrasto lo desplegado en AWS Console. En este archivo trabajé con EC2 y VPC; los NLB y S3 están en los apuntes de la formativa de cuatro módulos.

## 9. Registro de depuración

Para revisar el detalle de una ejecución activo el log en la misma terminal PowerShell:

```powershell
$env:TF_LOG="DEBUG"
$env:TF_LOG_PATH="$PWD\terraform-debug.log"
terraform validate
Test-Path .\terraform-debug.log
Get-Item .\terraform-debug.log | Select-Object Name, Length, LastWriteTime
Remove-Item Env:TF_LOG
Remove-Item Env:TF_LOG_PATH
```

Con `Test-Path` comprobé que se generara el archivo y con `Get-Item` revisé su tamaño. Si el error aparece durante `plan`, puedo repetir ese comando con `TF_LOG` activo y revisar el registro correspondiente. El log queda fuera del repositorio.
