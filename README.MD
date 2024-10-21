# Terraform AWS Setup

Este repositório contém a configuração do Terraform para criar uma infraestrutura na AWS, que inclui uma instância EC2 rodando Debian e um servidor Nginx.

 ### Provedor AWS

```hc1
provider "aws" {
    region = "us-east-1"
}
```
- Configura o provedor aws para a região us-east-1, no caso Virgínia do Norte.



 ### Variáveis projeto e candidato

 ```hc1
variable "projeto" {
    description = "Nome do projeto"
    type        = string
    default     = "VExpenses"
}

variable "candidato" {
    description = "Nome do candidato"
    type        = string
    default     = "SeuNome"
}
```
- Define duas variáveis para o uso no codigo que irá vir.

 ### Chave SSH, tls_private_key e aws-key_pair

 ```hc1
 resource "tls_private_key" "ec2_key" {
        algorithm = "RSA"
        rsa_bits  = 2048
    }

resource "aws_key_pair" "ec2_key_pair" {
        key_name   = "${var.projeto}-${var.candidato}-key"
        public_key = tls_private_key.ec2_key.public_key_openssh
    }
```
- tls_private_key : Gera uma chave SSH privada RSA com 2048 bits.
- aws_key_pair : Cria um par de chaves AWS usando a chave pública gerada.

### VPC, aws_vpc

```hc1
resource "aws_vpc" "main_vpc" {
        cidr_block           = "10.0.0.0/16"
        enable_dns_support   = true
        enable_dns_hostnames = true

    tags = {
        Name = "${var.projeto}-${var.candidato}-vpc"
        }
    }
```
- Cria uma VPC com suporte a DNS e hostnames habilitados.

### Subnet, aws_subnet

```hc1
 resource "aws_subnet" "main_subnet" {
        vpc_id            = aws_vpc.main_vpc.id
        cidr_block        = "10.0.1.0/24"
        availability_zone = "us-east-1a"

    tags = {
        Name = "${var.projeto}-${var.candidato}-subnet"
        }
    }
```
- Cria uma subnet dentro da VPC utilizando o bloco 10.0.1.0/24 na zona us-east-1a.

### Internet Gateway, aws_internet_gateway

```hc1
 resource "aws_internet_gateway" "main_igw" {
        vpc_id = aws_vpc.main_vpc.id

    tags = {
        Name = "${var.projeto}-${var.candidato}-igw"
        }
    }
```
- Cria uma internet gateway para a VPC que se comunica com a internet.

### Tabela de Rotas e Associação, aws_route_table, aws_route_table_association

```hc1
resource "aws_route_table" "main_route_table" {
    vpc_id = aws_vpc.main_vpc.id

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.main_igw.id
    }

    tags = {
        Name = "${var.projeto}-${var.candidato}-route_table"
        }
    }

    resource "aws_route_table_association" "main_association" {
    subnet_id      = aws_subnet.main_subnet.id
    route_table_id = aws_route_table.main_route_table.id

    tags = {
        Name = "${var.projeto}-${var.candidato}-route_table_association"
        }
    }
```
- Define uma tabela de rotas associada à VPC, deixando o tráfego de saida para qualquer ip através do gateway. Associa a sub-rede já criada a tabela de rotas.

### Segurança, aws_security_group

```hc1
resource "aws_security_group" "main_sg" {
    name        = "${var.projeto}-${var.candidato}-sg"
    description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
    vpc_id      = aws_vpc.main_vpc.id

    ingress {
        description      = "Allow SSH from anywhere"
        from_port        = 22
        to_port          = 22
        protocol         = "tcp"
        cidr_blocks      = ["0.0.0.0/0"]
        ipv6_cidr_blocks = ["::/0"]
    }

    egress {
        description      = "Allow all outbound traffic"
        from_port        = 0
        to_port          = 0
        protocol         = "-1"
        cidr_blocks      = ["0.0.0.0/0"]
        ipv6_cidr_blocks = ["::/0"]
    }

    tags = {
        Name = "${var.projeto}-${var.candidato}-sg"
        }
    }
```
- Cria um grupo de segurança permitindo o acesso SSH(porta 22) de qualquer lugar, permitindo também todo tráfego de saída.

### AWS Debian, aws_ami e debian12

```hc1
data "aws_ami" "debian12" {
    most_recent = true

    filter {
        name   = "name"
        values = ["debian-12-amd64-*"]
    }

    filter {
        name   = "virtualization-type"
        values = ["hvm"]
    }

    owners = ["679593333241"]
    }
```
- Busca a imagem AMI mais recente do Debian 12, utilizando filtros para garantir que a imagem seja compativel com o tipo HVM.

### Instância EC2, aws_instance e debian_ec2

```hc1
 resource "aws_instance" "debian_ec2" {
    ami             = data.aws_ami.debian12.id
    instance_type   = "t2.micro"
    subnet_id       = aws_subnet.main_subnet.id
    key_name        = aws_key_pair.ec2_key_pair.key_name
    security_groups = [aws_security_group.main_sg.name]

    associate_public_ip_address = true

    root_block_device {
        volume_size           = 20
        volume_type           = "gp2"
        delete_on_termination = true
    }

    user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install -y nginx
              systemctl start nginx
              systemctl enable nginx
              EOF

    tags = {
        Name = "${var.projeto}-${var.candidato}-ec2"
        }
    }
```
- Cria uma instância de EC2 com o Debian 12 mais recente.
    - No momento da inicialização do script o user_data é executado :
      - Atualiza o sistema. 
      - Instala o servidor web Nginx.
      - Inicia o serviço Nginx para ser iniciado no boot.

    - É destinado 20gb de armazenamento nessa instância.

### Outputs, private_key e ec2_public_ip

```hc1
output "private_key" {
        description = "Chave privada para acessar a instância EC2"
        value       = tls_private_key.ec2_key.private_key_pem
        sensitive   = true
    }

    output "ec2_public_ip" {
        description = "Endereço IP público da instância EC2"
        value       = aws_instance.debian_ec2.public_ip
    }
```
- private_key : Chave privada para o acessar a instância EC2, sensível para nao aparecer em logs.
- ec2_public_ip : Exibe o ip público. 
