# wordpress-aws

# Projeto de Implantação de WordPress na AWS

## Descrição do Projeto
Este projeto tem como objetivo a criação de instâncias EC2 privadas na AWS utilizando um container com a imagem do WordPress. Para isso, é necessário atender a algumas requisições, como:

- Conexão ao serviço RDS da Amazon.
- Utilização do EFS para armazenamento de arquivos estáticos.
- Criação e configuração da VPC, alocando instâncias e serviços nas redes privadas e públicas.
- Criação de um Load Balancer para a conexão externa das instâncias privadas.
- Implementação de um Auto Scaling Group para garantir a escalabilidade e segurança do sistema.

## 1. Criação da VPC
### Nome da VPC: my-vpc-wp
- **Tipo de Recurso**: VPC apenas
- **Tag de Nome**: Nomeie sua VPC
- **Bloco de Endereço IPv4**: 10.0.0.0/16 (evita sobreposição com outras redes)
- **Tenancy**: Default
- **Tags**:
  - **KEY**: Name
  - **VALUE**: Nome da VPC

### Configuração das Subnets
#### Subnet 1: subnet-publica-1
- **Availability Zone**: us-east-1a
- **CIDR Block**: 10.0.0.x

#### Subnet 2: subnet-publica-2
- **Availability Zone**: us-east-1b
- **CIDR Block**: 10.0.1.x

#### Subnet 3: subnet-privada-1
- **Availability Zone**: us-east-1a
- **CIDR Block**: 10.0.2.x

#### Subnet 4: subnet-privada-2
- **Availability Zone**: us-east-1b
- **CIDR Block**: 10.0.3.x

## 2. Criação das Route Tables
### Route Table Pública: rota-publica
- **Associada à VPC**: my-vpc-wp
- **Tag de Nome**: rota-publica

### Route Table Privada: rota-privada
- **Associada à VPC**: my-vpc-wp
- **Tag de Nome**: rota-privada

## 3. Configuração do Internet Gateway
### Nome do IGW: my-internet-gateway-wp
- **Associar à VPC**: my-vpc-wp

## 4. Criação do NAT Gateway
### Nome do NAT Gateway: my-nat-gateway-wp
- **Associar à Subnet Pública**: subnet-publica-1
- **Alocar Elastic IP**: Sim

## 5. Configuração das Tabelas de Rota
### Tabela de Rota Privada: rota-privada
- associe as subnets privadas a tabela de rotas privadas
- **Rota**: 0.0.0.0/0 - my-nat-gateway-wp

### Tabela de Rota Pública: rota-publica
- associe as subnets publicas a tabela de rotas publica
- **Rota**: 0.0.0.0/0 - my-internet-gateway-wp

## 6. Grupos de Segurança (Security Groups)

### ec2-sg

Regras de entrada:

SSH: Porta 22 (somente do bastion-host).

HTTP: Porta 80 (apenas do load-balancer-sg).

HTTPS: Porta 443 (apenas do load-balancer-sg).

Custom TCP: Porta 8080 (apenas do load-balancer-sg).

NFS: Porta 2049 (apenas do efs-sg).

MySQL/Aurora: Porta 3306 (apenas do rds-sg).

### load-balancer-sg

Regras de entrada:

HTTP: Porta 80 (0.0.0.0/0).

HTTPS: Porta 443 (0.0.0.0/0).

Custom TCP: Porta 8080 (0.0.0.0/0).

### efs-sg

Regras de entrada:

NFS: Porta 2049 (somente do ec2-sg).

### rds-sg

Regras de entrada:

MySQL/Aurora: Porta 3306 (somente do ec2-sg).

## 7. EFS - Elastic File System
### Nome do EFS: EFS-WP
- **Associado à VPC**: my-vpc-wp
- **Security Group**: efs-sg

## 8. Lançamento da Instância EC2 Privada
### Configurações da Instância
- **AMI**: Ubuntu
- **Tipo de Instância**: t2.micro
- **Subnet**: subnet-privada
- **Auto-assign Public IP**: Desabilitado
- **Security Group**: ec2-sg

### User Data

   ```bash

#!/bin/bash

# Atualiza e instala dependências

sudo apt update -y && sudo apt upgrade -y

sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

#Instala o Docker

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y

sudo apt install -y docker-ce docker-ce-cli containerd.io

# Inicia e habilita o serviço Docker

sudo systemctl start docker

sudo systemctl enable docker

sudo usermod -aG docker ubuntu

# Instala o cliente MySQL


sudo apt install -y mysql-client-core-8.0

# Baixa a imagem do WordPress

sudo docker pull wordpress
# Monta o EFS

sudo mkdir -p /efs

sudo apt-get install -y nfs-common

sudo mount -t nfs4 -o nfsvers=4.1 (EFS-DNS):/ /efs

# Configura o Docker Compose

sudo mkdir -p /caminho/para/wp

sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

#Cria o arquivo docker-compose.yaml

cat > /caminho/para/wp/docker-compose.yaml <<EOL
services:
     wordpress:
      image: wordpress:latest
      restart: always
      ports:
      - "80:80"
      environment:
          WORDPRESS_DB_HOST: seuhostRDS
          WORDPRESS_DB_NAME: dbname
          WORDPRESS_DB_USER: username
          WORDPRESS_DB_PASSWORD: suasenha
      volumes:
          - /efs/efs_wordpress:/var/www/html
EOL
Inicia o container do WordPress
cd /caminho/para/wp
sudo docker compose up -d
```
## 9. Configuração do Load Balancer
### Nome do Load Balancer: LB-WP
- **Tipo**: Classic Load Balancer
- **Configuração Básica**: Internet-facing
- **Protocolos e Portas**:
  - **Listener Protocol**: HTTP
  - **Listener Port**: 80
  - **Health Check**: HTTP, Port: 80, Path: /
  - **Security Group**: load-balance-sg
    
## 10. Configuração do WordPress
### URLs
As configurações siteurl e home precisam apontar para o DNS público do Load Balancer.

Via Painel Administrativo

Acesse Configurações > Geral.

Configure:
- **Endereço do WordPress (URL)**: http://seu-load-balancer-dns
- **Endereço do site (URL)**: http://seu-load-balancer-dns

### wp-config.php
Via wp-config.php (se não puder acessar o painel) Edite o arquivo wp-config.php na pasta do WordPress e adicione as linhas abaixo:

```php
define('WP_HOME', 'http://seu-load-balancer-dns');
define('WP_SITEURL', 'http://seu-load-balancer-dns');
```

## 11. Configuração do Auto Scaling Group
Selecione um Launch Template existente ou crie um novo.
Esse template define configurações essenciais para as instâncias, incluindo:
Tipo de instância (por exemplo, t3.micro, t2.medium).
Imagem de Máquina Amazon (AMI), como Ubuntu ou outra distribuição.
Configurações de rede, grupos de segurança e permissões IAM.
Subnets privadas para alocação de rede.

- **Capacidade Desejada**: 2
- **Capacidade Mínima**: 2
- **Capacidade Máxima**: 4
- **Políticas de Escalabilidade**: Baseadas em métricas de CPU e requisições no Load Balancer.

## Conclusão
Este documento fornece um guia abrangente para a configuração de um ambiente WordPress na AWS, garantindo segurança, escalabilidade e alta disponibilidade.
