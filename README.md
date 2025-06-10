# WordPress com Docker e AWS (Windows/WSL, EC2 AWS, EFS, RDS, ALB, User Data, VPC, Subnet)

Este trabalho unifica, detalha e explica todos os cenários modernos para rodar *WordPress* via *Docker* e *AWS*:
- *Ambiente local* usando Windows/WSL
- *Deploy em nuvem* usando EC2 da AWS com banco MySQL em container ou RDS
- *Amazon EFS* para armazenamento compartilhado dos uploads do WordPress
- *Application Load Balancer (ALB)* para distribuir o tráfego entre múltiplas EC2
- *Automação via User Data* para provisionamento rápido e padronizado
- *VPC, Subnet, Internet Gateway, Route Table e Security Group* explicados passo a passo
- *Explicações detalhadas, comandos de diagnóstico, checklist e resolução de problemas*

Ideal para desenvolvimento, testes, aprendizado e produção escalável em cloud.

---

## Explicação dos Componentes

- *WordPress*: É um sistema de gerenciamento de conteúdo (CMS) popular para criação de sites e blogs. Ele precisa de um servidor web (Apache/Nginx), PHP e um banco de dados MySQL/MariaDB.
- *Docker*: Plataforma de containers que permite empacotar a aplicação e suas dependências, garantindo portabilidade e facilidade de deploy.
- *Docker Compose*: Ferramenta para orquestrar múltiplos containers (ex: WordPress + MySQL) via um arquivo YAML, facilitando o gerenciamento dos serviços.
- *Windows/WSL*: O Windows Subsystem for Linux permite rodar comandos Linux no Windows, facilitando o uso do Docker e integração de sistemas.
- *EC2 (AWS)*: Instâncias de máquinas virtuais escaláveis na AWS, onde hospedamos os containers Docker.
- *RDS (Relational Database Service)*: Serviço gerenciado de banco de dados, que oferece alta disponibilidade, backup e performance, ideal para produção.
- *Amazon EFS (Elastic File System)*: Sistema de arquivos em rede escalável e compartilhado, permitindo que múltiplas instâncias EC2 acessem os mesmos arquivos (ideal para uploads do WordPress).
- *ALB (Application Load Balancer)*: Distribui o tráfego HTTP/HTTPS entre múltiplas instâncias EC2, garantindo alta disponibilidade e escalabilidade.
- *User Data*: Script de inicialização automática para configurar a instância EC2 ao boot, automatizando toda a instalação e configuração.
- *Security Group (SG)*: Firewall virtual da AWS para controlar o tráfego de rede das instâncias e serviços.
- *VPC (Virtual Private Cloud)*: Rede virtual isolada na AWS, onde ficam os recursos como EC2, RDS e EFS.
- *Subnet*: Segmento da VPC, define o intervalo de IPs e a disponibilidade (pública ou privada) das máquinas.
- *Internet Gateway (IGW)*: Permite que recursos da VPC acessem a internet.
- *Route Table*: Define como o tráfego é roteado dentro da VPC.

---

## Sumário

1. [VPC, Subnet, Internet Gateway, Route Table e Security Group: Guia Prático na AWS](#vpc-subnet-internet-gateway-route-table-e-security-group-guia-prático-na-aws)
2. [WordPress com Docker no Windows/WSL](#wordpress-com-docker-no-windowswsl)
3. [WordPress em EC2 (AWS) com Docker Compose (MySQL em container)](#wordpress-em-ec2-aws-com-docker-compose-mysql-em-container)
4. [WordPress na EC2 com RDS MySQL (AWS) + User Data](#wordpress-na-ec2-com-rds-mysql-aws--user-data)
5. [Amazon EFS (Elastic File System) – Armazenamento Compartilhado](#amazon-efs-elastic-file-system--armazenamento-compartilhado)
6. [Application Load Balancer (ALB) e Target Group](#application-load-balancer-alb-e-target-group)
7. [Diagnóstico e Comandos Úteis](#diagnóstico-e-comandos-úteis)
8. [Resolução de erros comuns](#resolução-de-erros-comuns)
9. [Checklist de Infraestrutura](#checklist-de-infraestrutura)
10. [Resumo: Por que EFS e ALB com WordPress?](#resumo-por-que-efs-e-alb-com-wordpress)
11. [Dicas de Segurança e Produção](#dicas-de-segurança-e-produção)
12. [Acesso ao WordPress](#acesso-ao-wordpress)
13. [Referências úteis](#referências-úteis)
14. [Observações](#observações)

---

# VPC, Subnet, Internet Gateway, Route Table e Security Group: Guia Prático na AWS

## O que é VPC?
*VPC (Virtual Private Cloud)* é uma rede isolada logicamente dentro da AWS onde você pode lançar recursos (EC2, RDS, EFS, etc.) em um ambiente seguro, controlando endereçamento IP, sub-redes e rotas.

## O que é Subnet?
*Subnet* é uma subdivisão da VPC, permitindo separar recursos em diferentes zonas de disponibilidade (AZs) ou controlar o acesso deles (pública/privada).

- *Subnet Pública:* Tem rota para a internet via IGW. Usada para recursos que precisam ser acessados da internet (ex: EC2 do WordPress, ALB).
- *Subnet Privada:* Sem rota direta para a internet. Usada para recursos internos (ex: bancos RDS, servidores de aplicação internos).

## O que é Internet Gateway (IGW)?
Dispositivo virtual anexado à VPC que permite comunicação entre recursos da VPC e a internet.

## O que é Route Table?
Tabela de rotas define como o tráfego de rede é direcionado dentro da VPC (por exemplo, para a internet, para outras subnets, etc.).

## O que é Security Group?
Firewall virtual para controlar o tráfego de entrada e saída de recursos.

---

## Passo a Passo – Infraestrutura de Rede AWS

### 1. Criar a VPC

- Console AWS > VPC > Criar VPC
- Nome: wordpress-vpc
- IPv4 CIDR block: 10.0.0.0/16 (padrão)
- Marque “Enable DNS hostnames” para facilitar uso de EFS/RDS

### 2. Criar Subnets

- *Subnet Pública:*  
  - VPC: wordpress-vpc
  - Nome: public-subnet-1
  - AZ: us-east-1a (por exemplo)
  - IPv4 CIDR block: 10.0.1.0/24

- (Opcional) *Subnet Privada:*  
  - Nome: private-subnet-1
  - AZ: us-east-1a
  - IPv4 CIDR block: 10.0.2.0/24

### 3. Criar Internet Gateway

- VPC > Internet Gateways > Criar IGW
- Nome: wordpress-igw
- Anexar à sua VPC

### 4. Criar Route Table

- VPC > Route Tables > Criar Route Table
- Nome: public-rt
- VPC: wordpress-vpc
- Adicione rota:
  - Destination: 0.0.0.0/0
  - Target: Internet Gateway criado (wordpress-igw)
- Associe esta Route Table à Subnet Pública

### 5. Security Groups

- EC2 > Security Groups > Criar
- Nome: wordpress-sg
- Regras de entrada:
  - Porta 22 (SSH) do seu IP
  - Porta 80 (HTTP) de 0.0.0.0/0
  - Porta 443 (HTTPS) de 0.0.0.0/0
- Regras de saída: padrão (tudo liberado)
- Associe este SG à sua instância EC2 (e ALB, se usar)

### 6. Checklist Visual (Resumo):


VPC (10.0.0.0/16)
 ├── Subnet Pública (10.0.1.0/24) [public-subnet-1]
 │    └── EC2, ALB
 ├── Subnet Privada (10.0.2.0/24) [private-subnet-1]
 │    └── RDS, EFS (opcional)
 └── Internet Gateway [wordpress-igw]
      └── Route Table [public-rt] (0.0.0.0/0 → IGW)


### 7. Observações Importantes

- Para acessar a internet, EC2 deve estar em subnet pública, associada à route table que redireciona tráfego para o IGW.
- O RDS pode (e deve, em produção) ficar em subnet privada, acessível apenas pela EC2.
- O EFS pode ser acessado por ambas, mas sempre liberar porta 2049/TCP no SG do EFS para o SG das EC2.

---

# WordPress com Docker no Windows/WSL

## O que é?
Rodar WordPress localmente em containers Docker usando o Windows, aproveitando a integração com o WSL2 (subsistema Linux), facilita o desenvolvimento e testes sem comprometer a máquina principal.

## Requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado e rodando no Windows  
  Permite criar e gerenciar containers de maneira simples.
- Integração do Docker com o WSL2 ativada  
  Melhora desempenho, compatibilidade com comandos Linux e caminhos de arquivos.
- Pasta do WordPress extraída em D:\wordpress (ou seu caminho preferido)  
  Facilita mapear arquivos do host para o container, garantindo persistência e facilidade de edição de arquivos.

## Estrutura do Projeto


D:\
└── wordpress\
    ├── wp-admin\
    ├── wp-content\
    ├── wp-includes\
    └── ...

*Essas pastas são essenciais para o funcionamento do WordPress, especialmente wp-content para plugins, temas e uploads.*

## Subindo os containers

### 1. Suba o container do MySQL

bash
docker run --name some-mysql \
  -e MYSQL_ROOT_PASSWORD=wordpress \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wordpress \
  -e MYSQL_PASSWORD=wordpress \
  -d mysql:5.7

- *Explicação:* Cria um banco MySQL chamado wordpress, usuário e senha padrão para facilitar testes locais.

### 2. Suba o container do WordPress

No Ubuntu/WSL:

bash
cd /mnt/d/wordpress
docker run --name some-wordpress \
  --link some-mysql:mysql \
  -p 8080:80 \
  -d \
  -v /mnt/d/wordpress:/var/www/html \
  wordpress

- *Explicação:*  
  - --link some-mysql:mysql: conecta o container do WordPress ao banco MySQL.
  - -p 8080:80: expõe a porta 80 do container na porta 8080 do host.
  - -v /mnt/d/wordpress:/var/www/html: mapeia a pasta local para persistir arquivos do WordPress.

## Acessando o WordPress

- [http://localhost:8080](http://localhost:8080)
- Use:
    - *Banco:* wordpress
    - *Usuário:* wordpress
    - *Senha:* wordpress
    - *Servidor:* mysql

## Comandos úteis

- Parar/remover containers:
  bash
  docker stop some-wordpress some-mysql
  docker rm some-wordpress some-mysql
  
- Logs: docker logs some-wordpress
- Containers rodando: docker ps

## Dicas e Solução de problemas

- *Permissões:* Ajuste permissões em D:\wordpress se necessário.
- *Persistência:* Use volumes para /var/lib/mysql.
- *Falha de conexão:* Verifique porta, containers rodando, nome do banco = mysql.
- *Permissão para criar arquivos:* sudo chmod -R 777 /mnt/d/wordpress
- *Reiniciar tudo:* Remova containers e dados, recomece.

---

# WordPress em EC2 (AWS) com Docker Compose (MySQL em container)

## O que é?
Executa o WordPress em uma instância EC2 na AWS, com Docker Compose para orquestrar tanto o WordPress quanto o banco MySQL em containers. Ideal para testar deploys em cloud.

## Passo a Passo no Ambiente AWS

> *Importante:* Antes de criar a instância EC2, é necessário já ter configurado a VPC, Subnet, IGW, Route Table e Security Groups conforme a seção anterior.

### 1. Criação e Acesso à Instância EC2

- Escolha uma AMI Ubuntu, coloque na subnet pública, associe o SG, gere uma chave SSH.
- Corrija permissões do .pem e conecte:
  bash
  chmod 400 ~/teste1pb.pem
  ssh -i ~/teste1pb.pem ubuntu@IP_PUBLICO_DA_EC2
  

### 2. Instalação do Docker e Docker Compose

bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER

- Instala e ativa o Docker, adiciona o usuário ao grupo docker para dispensar sudo.

### 3. Deploy do WordPress com Docker Compose

Crie o docker-compose.yml e suba:

yaml
version: '3.1'
services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_ROOT_PASSWORD: example
    volumes:
      - db_data:/var/lib/mysql

volumes:
  wordpress_data:
  db_data:

- Define dois serviços (WordPress e banco), cada um com seu volume para persistência.

bash
docker-compose up -d

- Sobe os containers em segundo plano.

---

# WordPress na EC2 com RDS MySQL (AWS) + User Data

## O que é?
Utiliza o RDS (banco de dados gerenciado) para separar o banco do WordPress, tornando o ambiente mais escalável e resiliente. O script User Data automatiza toda a configuração da instância ao inicializar.

## Passo a Passo no Ambiente AWS

> *Importante:* Garanta que RDS esteja na mesma VPC/Subnet (privada de preferência) e liberado no SG para a EC2.

### 1. Criar o RDS MySQL

- Console AWS > RDS > Criar banco de dados
- Engine: MySQL, defina usuário/senha, anote endpoint
- Security Group do RDS: porta 3306 aberta para SG da EC2
- Banco de dados pode ser criado via User Data ou manualmente

### 2. Criar e Configurar Instância EC2

- AMI Ubuntu, subnet pública, SG configurado (porta 80 para web, 22 para seu IP), chave SSH criada
- No campo *User Data* da EC2, cole o script abaixo:

bash
#!/bin/bash
apt-get update -y
apt-get install -y docker.io mysql-client
systemctl start docker
systemctl enable docker
docker pull wordpress:latest
sleep 10
DB_HOST="database-1.ce3yu8ycol1v.us-east-1.rds.amazonaws.com"
DB_USER="admin"
DB_PASS="admin1234"
DB_NAME="wordpress"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -e "CREATE DATABASE IF NOT EXISTS $DB_NAME;"
docker rm -f wordpress || true
docker run -d --name wordpress \
  -p 80:80 \
  -e WORDPRESS_DB_HOST=$DB_HOST:3306 \
  -e WORDPRESS_DB_USER=$DB_USER \
  -e WORDPRESS_DB_PASSWORD=$DB_PASS \
  -e WORDPRESS_DB_NAME=$DB_NAME \
  wordpress:latest

- Este script instala Docker, puxa a imagem do WordPress, garante que o banco existe e sobe o container já apontando para o RDS.

- Acesse via SSH (opcional) e monitore logs com:
  bash
  tail -f /var/log/cloud-init-output.log
  

---

# Amazon EFS (Elastic File System) – Armazenamento Compartilhado

## O que é?
O EFS é um sistema de arquivos em rede, compartilhável entre múltiplas EC2, ideal para armazenar uploads do WordPress em ambientes escaláveis (clusters).

## Passo a Passo no Ambiente AWS

1. *Criar EFS:*  
   Console AWS > EFS > Criar sistema de arquivos (mesma VPC das EC2)
2. *Mount Target:*  
   Adicione na subnet/AZ das EC2, SG do EFS libera 2049/TCP para SG das EC2
3. *Configuração de DNS na VPC:*  
   Habilite resolução DNS e nomes de host DNS
4. *Instale NFS na EC2:*  
   bash
   sudo apt update
   sudo apt install -y nfs-common
   
5. *Monte o EFS:*  
   bash
   sudo mkdir -p /mnt/efs
   sudo mount -t nfs4 -o nfsvers=4.1 fs-XXXXXXXX.efs.REGIAO.amazonaws.com:/ /mnt/efs
   
6. **Adicione ao /etc/fstab para automount**
7. *No Docker:*  
   Monte /mnt/efs em /var/www/html/wp-content/uploads para persistência:

   bash
   docker run -d --name wordpress \
     ... \
     -v /mnt/efs:/var/www/html/wp-content/uploads \
     wordpress:latest
   
- Assim, todos os uploads do WordPress ficam persistentes e acessíveis por múltiplas instâncias.

---

# Application Load Balancer (ALB) e Target Group

## O que é?
O ALB distribui o tráfego HTTP/HTTPS entre várias instâncias de WordPress, garantindo alta disponibilidade, failover e auto scaling.

## Passo a Passo no Ambiente AWS

> *Importante:* O ALB deve ser criado na mesma VPC e em subnets públicas.

### 1. Criar o Application Load Balancer (ALB):

- Console AWS > EC2 > Load Balancers > Criar Load Balancer
- Escolha Application Load Balancer
- Nomeie, escolha esquema Internet-facing, selecione VPC e subnets públicas
- Configure Security Group permitindo porta 80

### 2. Criar o Target Group:

- Tipo Instance, HTTP:80, health check /
- Adicione as EC2 rodando WordPress
- Health check path: / (ou /health)

### 3. Associar Instâncias ao Target Group:

- Selecione as EC2, adicione ao Target Group

### 4. Teste e monitore:

- Aguarde o status do ALB ficar “active”
- Verifique se instâncias estão “healthy” no Target Group
- Copie o DNS do ALB e acesse pelo navegador

### 5. Segurança:

- SG das EC2 deve permitir tráfego do ALB na porta 80

---

# Diagnóstico e Comandos Úteis

- Checar DNS do EFS:  
  host fs-XXXXXXXX.efs.REGIAO.amazonaws.com
- Testar porta NFS:  
  nc -zv <IP_MOUNT_TARGET> 2049
- Montagem EFS:  
  df -h | grep efs, mount | grep efs
- Testar arquivo:  
  sudo touch /mnt/efs/teste.txt
- Health check ALB:  
  Verifique status das instâncias no Target Group

---

# Resolução de erros comuns

## WordPress
- *"Error establishing a database connection":*  
  Verifique variáveis, bancos, SGs
- *Permissão Docker:*  
  Adicione usuário ao grupo docker
- *Porta 80 não liberada:*  
  Ajuste SG da EC2

## EFS
- *DNS do EFS não resolve:*  
  Habilite DNS na VPC
- *Porta 2049 bloqueada:*  
  Revise SG do EFS
- *Permissão negada ao montar:*  
  Confirme SGs e IPs

## ALB
- *Instâncias "unhealthy":*  
  Health check falhou. Revise path, serviços, SG entre ALB e EC2
- *503 Service Unavailable:*  
  Nenhum target healthy

---

# Checklist de Infraestrutura

- [x] VPC criada e configurada
- [x] Subnet pública associada ao IGW
- [x] Internet Gateway criado e anexado
- [x] Route Table pública com rota para IGW
- [x] Security Groups configurados (EC2, RDS, EFS, ALB)
- [x] EFS criado e disponível
- [x] Mount Target na mesma subnet/AZ da EC2
- [x] DNS habilitado na VPC
- [x] Security Group do EFS com 2049 liberada para EC2
- [x] EC2 com Docker, NFS e cliente MySQL
- [x] Banco criado no RDS
- [x] ALB ativo, Target Group saudável

---

# Resumo: Por que EFS e ALB com WordPress?

- *EFS:* uploads compartilhados, múltiplas instâncias, escalabilidade  
  Permite que todas as instâncias tenham acesso aos mesmos arquivos de upload, essencial para ambientes com várias EC2.
- *ALB:* balanceamento, failover, auto scaling, alta disponibilidade  
  Garante que o tráfego seja distribuído e o sistema continue acessível mesmo se uma instância falhar.

---

# Dicas de Segurança e Produção

- Nunca deixe portas sensíveis (3306, 22, 2049) abertas para o mundo
- Use variáveis de ambiente seguras (AWS Secrets Manager)
- Prefira RDS/EFS para produção; banco local só para testes
- Use backup e monitore logs:
  bash
  tail -f /var/log/cloud-init-output.log
  
- Use HTTPS e configure domínio

---

# Acesso ao WordPress

- Aguarde instâncias EC2 “2/2 verificações”
- No navegador, acesse:
  - http://<DNS-do-ALB>
  - ou http://<IP-PUBLICO-DA-EC2> (caso sem ALB)
- Siga instalação do WordPress

---

# Referências úteis

- [Documentação EFS AWS](https://docs.aws.amazon.com/efs/latest/ug/)
- [WordPress Docker Hub](https://hub.docker.com/_/wordpress)
- [Documentação RDS MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MySQL.html)
- [Docker para Windows/WSL](https://docs.docker.com/desktop/wsl/)
- [Documentação ALB AWS](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [Documentação VPC AWS](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [AWS Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html)
- [Internet Gateway AWS](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)

---

# Observações

- Todo o procedimento pode ser feito sem SSH, apenas pelo Console AWS (usando User Data)
- As credenciais do banco estão expostas no script User Data. Em produção, utilize mecanismos seguros
- Dimensione EC2/RDS/EFS conforme sua carga
- O EFS permite compartilhamento de dados entre múltiplas EC2
- *VPC, Subnet, IGW, Route Table e Security Groups* são essenciais para segurança e funcionamento da sua arquitetura AWS.

---
