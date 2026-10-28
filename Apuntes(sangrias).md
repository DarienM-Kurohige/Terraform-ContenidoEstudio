## 1. Módulo VPC
Empecé por la VPC porque las EC2 y los NLB necesitan una red donde ubicarse. En el módulo declaré la VPC y usé "count" para crear las dos subredes sin repetir el bloque "aws_subnet". Además, cada subred toma el ID de la VPC creada en el mismo módulo, por lo que no tuve que escribir ese identificador manualmente.

### "modulos/vpc/variables.tf"

Definí cuatro entradas: un nombre base, el CIDR de la VPC, una lista de CIDR de subredes y una lista de zonas de disponibilidad. Los valores concretos los entrega el archivo raíz al llamar al módulo.

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

### "modulos/vpc/main.tf"

"local.nombre_vpc" construye el nombre del recurso. "count.index" recorre las listas y asocia cada subred a su CIDR y zona correspondiente. La referencia "aws_vpc.principal.id" establece la dependencia interna entre subred y VPC.

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

### "modulos/vpc/outputs.tf"

Expuse el ID de la VPC y los IDs de todas las subredes. El output "subnet_ids" es el que después utilicé en EC2 y NLB desde el módulo raíz.

```hcl
output "vpc_id" {
  value = aws_vpc.principal.id
}

output "subnet_ids" {
  value = aws_subnet.subredes[*].id
}

## 2. Módulo EC2

Después creé el módulo EC2. Utilicé "count" para levantar dos instancias "t3.micro" con la misma AMI y distribuí una en cada subred. Los IDs no están escritos en este módulo: llegan desde el output de VPC a través del "main.tf" principal. De esta forma, los nombres de variables del módulo EC2 coinciden con los argumentos que recibe y sus outputs quedan disponibles para consultar las instancias creadas.
```

### "modulos/ec2/variables.tf"

El módulo recibe el nombre base, la AMI, el tipo de instancia, la cantidad y la lista de subredes.

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

### "modulos/ec2/main.tf"

El operador "%" permite alternar los índices de las subredes. Con dos instancias y dos subredes, la primera usa el índice "0" y la segunda el "1". Desactivé la IP pública porque esta formativa solamente pedía crear las instancias dentro de la VPC, no habilitar acceso desde Internet.

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

### "modulos/ec2/outputs.tf"

Dejé como salidas los IDs y las IP privadas de las dos instancias.

```hcl
output "instance_ids" {
  value = aws_instance.servidores[*].id
}

output "private_ips" {
  value = aws_instance.servidores[*].private_ip
}

## 3. Módulo NLB

El módulo NLB sigue la misma idea que EC2: recibe las subredes desde VPC y usa "count" para crear dos balanceadores. Elegí "network" porque el enunciado pedía Network Load Balancer, e "internal = true" porque no se solicitó publicarlos en Internet. Cada balanceador utiliza una de las dos subredes.
```

### "modulos/nlb/variables.tf"

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

### "modulos/nlb/main.tf"

El "local" limita la base del nombre y el recurso añade el sufijo correspondiente a cada balanceador. La expresión de "subnets" selecciona una subred según el índice de "count".

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

### "modulos/nlb/outputs.tf"

Expuse los nombres DNS y los ARN que AWS asignó a los NLB.


```hcl
output "nlb_dns" {
  value = aws_lb.balanceadores[*].dns_name
}

output "nlb_arns" {
  value = aws_lb.balanceadores[*].arn
}

## 4. Módulo S3

En S3 utilicé el nombre generado desde el root y "count" para crear dos buckets con nombres distintos. Aparte declaré "aws_s3_bucket_public_access_block" para establecer las cuatro opciones de bloqueo de acceso público. A diferencia de EC2 y NLB, S3 no necesita recibir subredes de la VPC.
```

### "modulos/s3/variables.tf"

```hcl
variable "nombre" {
  type = string
}

variable "cantidad" {
  type = number
}
```

### "modulos/s3/main.tf"

"lower()" deja el nombre en minúsculas y "count.index + 1" diferencia los dos buckets. El segundo recurso toma el ID del bucket correspondiente para asociar la configuración de privacidad.

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

### "modulos/s3/outputs.tf"

El output permite consultar los nombres definitivos de ambos buckets.

```hcl
output "bucket_names" {
  value = aws_s3_bucket.buckets[*].bucket
}

## 5. Archivos del módulo raíz

El módulo raíz no repite los recursos de los hijos. Su función es definir los valores generales, llamar a VPC, EC2, NLB y S3, y pasar los outputs donde hacen falta. La dependencia más importante es "module.vpc.subnet_ids": desde ahí reciben sus subredes EC2 y NLB.
```

### "main.tf"

Usé "random_string" para obtener seis caracteres y un "local" para juntarlos con el nombre del proyecto. Ese nombre llega a los cuatro módulos. Los argumentos escritos en cada bloque "module" coinciden con las variables declaradas dentro del módulo hijo.

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

module "vpc" {
  source = "./modulos/vpc"

  nombre       = local.nombre
  cidr         = var.vpc_cidr
  subnet_cidrs = var.subnet_cidrs
  azs          = var.azs
}

module "ec2" {
  source = "./modulos/ec2"

  nombre         = local.nombre
  ami_id         = var.ami_id
  tipo_instancia = var.instance_type
  cantidad       = var.ec2_count

  subnet_ids = module.vpc.subnet_ids
}

module "nlb" {
  source = "./modulos/nlb"

  nombre     = local.nombre
  cantidad   = var.nlb_count
  subnet_ids = module.vpc.subnet_ids
}

module "s3" {
  source = "./modulos/s3"

  nombre   = local.nombre
  cantidad = var.s3_count
}
```

### "outputs.tf"

Estos outputs permiten consultar los resultados del proyecto completo sin entrar a cada subcarpeta. Los valores provienen de los outputs de los módulos hijos y del nombre calculado en el root.

```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "subnet_ids" {
  value = module.vpc.subnet_ids
}

output "nombre_generado" {
  value = local.nombre
}

output "ec2_ids" {
  value = module.ec2.instance_ids
}

output "ec2_ips_privadas" {
  value = module.ec2.private_ips
}

output "nlb_dns" {
  value = module.nlb.nlb_dns
}

output "s3_nombres" {
  value = module.s3.bucket_names
}
```

### "providers.tf"

Declaré los providers de AWS y Random. Configuré AWS en "us-east-1", la misma región utilizada por las subredes, las EC2, los NLB y los buckets.

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

### "variables.tf"

En este archivo dejé los parámetros generales: direccionamiento, zonas de disponibilidad, AMI, tipo de instancia y cantidades. Los valores por defecto permiten reproducir el ejercicio; "ami_id" se entrega aparte, porque tiene que corresponder a una imagen real de la región.

```hcl
variable "nombre_proyecto" {
  type    = string
  default = "iac-formativa"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_cidrs" {
  type = list(string)

  default = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]
}

variable "azs" {
  type = list(string)

  default = [
    "us-east-1a",
    "us-east-1b"
  ]
}
variable "ami_id" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "ec2_count" {
  type    = number
  default = 2
}

variable "nlb_count" {
  type    = number
  default = 2
}

variable "s3_count" {
  type    = number
  default = 2
}
```

### "terraform.tfvars"

En mi ejecución asigné una AMI de Amazon Linux 2023 de arquitectura x86, utilizada por las dos instancias.

```hcl
ami_id = "ami-04b938437b69a2c9c"
```

Este ID corresponde al entorno utilizado en la práctica; si cambio de región o la imagen deja de estar disponible, debo obtener una AMI compatible antes de ejecutar Terraform.

## 6. Comandos utilizados y comprobaciones

Trabajé en la terminal PowerShell de VS Code desde la carpeta raíz "eva-formativa-02". Primero confirmé que Terraform estuviera disponible e inicié el laboratorio de AWS Academy.

```powershell
terraform -version
```

Las credenciales temporales las cargué en la sesión actual de PowerShell. No las escribí dentro de los archivos ".tf".

```powershell
$env:AWS_ACCESS_KEY_ID="VALOR_DEL_LAB"
$env:AWS_SECRET_ACCESS_KEY="VALOR_DEL_LAB"
$env:AWS_SESSION_TOKEN="VALOR_DEL_LAB"
$env:AWS_DEFAULT_REGION="us-east-1"
```

Después seguí este orden. Repetí "init" al incorporar módulos nuevos y revisé el plan antes de aprobar cada despliegue.

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
terraform output
terraform state list
```

"init" inicializa providers y módulos; "fmt -recursive" formatea los archivos ".tf" de la raíz y las subcarpetas; "validate" revisa la configuración. "plan" muestra los cambios propuestos y normalmente consulta AWS para actualizar el estado. "apply" ejecuta los cambios aprobados; "output" muestra las salidas del módulo raíz y "state list" los recursos registrados.

## 7. Debugging

Para el requisito de debugging activé el nivel "DEBUG" y definí un archivo para guardar el registro:

```powershell
$env:TF_LOG="DEBUG"
$env:TF_LOG_PATH="$PWD\terraform-debug.log"
terraform validate
Test-Path .\terraform-debug.log
Get-Item .\terraform-debug.log | Select-Object Name, Length, LastWriteTime
```

"Test-Path" devolvió "True" y el archivo tenía 35.194 bytes, por lo que comprobé que no estaba vacío. Después desactivé las variables:

```powershell
Remove-Item Env:TF_LOG
Remove-Item Env:TF_LOG_PATH
```
