# Configuração de Infraestrutura AWS
## 1. Criação da VPC

1. Acesse o console da AWS e navegue até o serviço **VPC**.
2. Clique em **Criar VPC** e defina:
   - Click em `VPC and more` para ter uma vpc pré configurada.
   - Nome: `Projeto`
   - IPv4 CIDR Block: `10.0.0.0/16`
   - IPv6 CIDR Block: **Desativado** (ou conforme necessidade)
   - Tenancy : Default
   - Number of Availability Zones (AZs): 2
   - Number of public subnets: 2
   - Number of private subnets: 2
   - NAT gateways ($) : none
   - VPC endpoints: S3 Gateway
   - DNS options: Check enable DNS hostnames and enable DNS resolution

## 2. Criação e Configuração dos Security Groups (SGs)

1. No console da AWS, navegue até o serviço de EC2 e acesse **Security Groups**.
2. Crie os seguintes Security Groups:
   - **SG-EC2**: Permitir tráfego de entrada SSH (22), HTTP (80) e HTTPS (443).
   - **SG-EFS**: Permitir tráfego de entrada 2049 (NFS) para o SG-EC2.
   - **SG-RDS**: Permitir tráfego de entrada 3306 (MySQL) apenas do SG-EC2.

## 3. Criação do EFS

1. Acesse **EFS** e clique em **Create file systems**.
2. Defina o nome como `EFS-PROJETO` e escolha a VPC criada previamente.
3. Click em `Customize`.
 - File system type: Regional
 - Transition into Infrequent Access (IA): None
 - Transition into Archive: None
 - Throughput mode: Bursting
 - Security groups: SG-EFS

## 4. Criação do RDS
1. Click em **Subnet groups** e depois em **Create DB subnet group**.
     - Name: DB_Subnet_Group.
     - VPC: Selecione a VPC criada previamente.
     - Availability Zones: us-east-1a e us-east-1b.
     - Subnets: selecione todas as subnets.
       
3. Ainda no **RDS** clique em databases e **Create database**.
   - Choose a database creation method: Standard create
   - Engine options: MySQL
   - Templates: Free tier
   - Availability and durability: Single-AZ DB instance deployment (1 instance)
   - DB instance identifier: (nome da instancia)
   - Master username: (nome do usuario master)
   - Credentials management: Self managed
   - Master password: (senha do usuario master)
   - Tipo: `db.t3.micro`
   - Storage type: General Purpose SSD (gp3)
   - Armazenamento: 20GB SSD
   - Connectivity: Don’t connect to an EC2 compute resource
   - Conectividade: VPC criada anteriormente
   - DB subnet group: selecione o DB subnet group criado previamente
   - Public access: No
   - VPC security group (firewall): Choose existing
   - Existing VPC security groups: SG-RDS
   - Database authentication: Password authentication
   - Monitoring: Database Insights - Standard
   - Initial database name: nomeDB

## 5. Criação da Instância EC2

1. Acesse **EC2** e clique em **Launch instances**.
2. Escolha:
   - AMI: Amazon Linux 2023
   - Instance type: `t2.micro`
   - VPC e Subnet: conforme configurado
   - Security Group: `SG-EC2`
   - Key pair: Click em **Create key pair**
        * Escolha um nome para sua key pair e selecione o tipo RSA com formato .ppk.
   - VPC: selecione a VPC criada anteriormente.
   - Subnet: selecione uma subnet publica.
   - Auto-assign public IP: Enable
   - Firewall (security groups): Select existing security group
   - Common security groups: SG-EC2
   - Configure storage: 1x 8 GiB gp3
   - User data:
           
      ``` sh
      #!/bin/bash
      # Atualiza o sistema operacional
      sudo yum update -y
      
      # Instala o Docker
      sudo yum install -y docker
      sudo systemctl start docker
      sudo systemctl enable docker
      
      # Adiciona o usuário ec2-user ao grupo docker para executar comandos sem sudo
      sudo usermod -aG docker ec2-user
      
      # Aplica a mudança de grupo imediatamente sem necessidade de logout
      newgrp docker
      
      # Verifica se o Docker foi instalado corretamente
      docker --version
      
      # Baixa o Docker Compose na versão mais recente
      sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
      
      # Concede permissão de execução ao Docker Compose
      sudo chmod +x /usr/local/bin/docker-compose
      
      # Verifica se o Docker Compose foi instalado corretamente
      docker-compose version
      
      # Cria o diretório do projeto
      sudo mkdir -p /projeto
      
      # Cria o diretório onde o EFS será montado
      mkdir /efs
      
      # Instala o cliente do Amazon EFS
      sudo yum install -y amazon-efs-utils
      
      # Monta o sistema de arquivos EFS no diretório /efs
      sudo mount -t efs -o tls <ID_DO_RDS>:/ efs
      
      # Acessa o diretório do projeto
      cd /projeto
      
      # Cria o arquivo docker-compose.yml com as configurações do WordPress
      cat <<EOF > docker-compose.yml
      version: '3.8'
      
      services:
        wordpress:
          image: wordpress:latest
          container_name: wordpress_app
          restart: always
          ports:
            - "80:80"
          environment:
            WORDPRESS_DB_HOST: <EndPointBD>
            WORDPRESS_DB_NAME: <NomeBD>
            WORDPRESS_DB_USER: <UserDB>
            WORDPRESS_DB_PASSWORD: <PasswordDB>
          volumes:
            - /efs:/var/www/html  # Monta o EFS no diretório do WordPress
          networks:
            - wp_network
      
      volumes:
        wordpress_data:
      
      networks:
        wp_network:
      EOF
      
      # Inicia os containers em segundo plano
      docker-compose up -d
      ```
      
3. Conecte-se à instância via SSH e teste a aplicação.
