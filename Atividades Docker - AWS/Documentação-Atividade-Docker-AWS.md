# Atividade Prática Docker e AWS

## Requisitos Gerais da Atividade
Elaboração de arquitetura AWS com Auto Scaling Group e Load Balancer para gerenciamento de instâncias EC2, de diferentes Zonas de Disponibilidade, com contêineres WordPress conectados a um banco de dados MySQL RDS, e um sistema de arquivos EFS configurado para compartilhar pastas estáticas da aplicação.

<br>
<br>


<p align="center">
<img src="https://github.com/Nicolas-S-M/Images/blob/main/Atividade-Docker-Template.png" width="500" height="400">
</p>
<br>
<br>

## Tópicos da Atividade:
- Instalação e configuração do Docker via script User Data;
- Implementação de contêiner com aplicação WordPress e banco de dados MySQL RDS;
- Configuração de um sistema de arquivos EFS para estáticos da aplicação do contêiner WordPress;
- Configuração do Load Balancer;
- Configuração do Auto Scaling Group;
<br>
<br>
<br>

### •  Instalação e Configuração do Docker via Script User Data
A instalação e configuração do Docker na instância é realizado durante sua inicialização. O script no User Data é executado durante este período realizando updates, instalações e execuções de outros comandos informados. O contêiner é gerado com base em uma imagem gerada de um dockerfile, que configura as váriaveis de ambiente, o client para conexão com o database MySQL RDs, a porta de exposição do contêiner e as permissões necessárias para o funcionamento do servidor.
```bash
#!/bin/bash
sudo yum update
# sudo yum install -y nfs-utils
# sudo systemctl start nfs
# sudo systemctl enable nfs
sudo amazon-linux-extras install docker -y
sudo service docker start
sudo systemctl enable docker
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=fs-0c64cd46913ded989.efs.us-east-1.amazonaws.com,rw,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
    --opt device=:/ efs
mkdir -p /Dockerfile
echo "FROM wordpress:latest
RUN apt-get update && apt-get install -y default-mysql-client
ENV WORDPRESS_DB_HOST=database-1.chj99ulhygmu.us-east-1.rds.amazonaws.com
ENV WORDPRESS_DB_USER=admin
ENV WORDPRESS_DB_PASSWORD=adminadmin
ENV WORDPRESS_DB_NAME=WPDatabase
WORKDIR /var/www/html
EXPOSE 80
RUN chown -R www-data:www-data /var/www/html" >> /Dockerfile/dockerfile
docker build -t mywordpress /Dockerfile
docker run -d \
  --name wordpress \
  -v  efs:/var/www/html/ \
  -p 80:80 \
  --restart always \
  mywordpress 
```
<br>
<br>
<br>

### •	 Implementar contêiner com aplicação WordPress e banco de dados MySQL RDS
 Antes de realizar a criação da instância, é importante iniciar uma instância RDS com banco de dados MYSQL. Para isso é necessário :
 - No Console da AWS acessar o recurso **RDS** e selecionar a opção **Create Database**;
 - Selecione o banco de dados **MySQL**;
 - Selecione o **Free Tier**;
 - Informe o **DB cluster identifier**, o **Master Username**, a **Master User Password** e um **Nome para Database**;
 - Selecionar o tamanho mínimo : **20 GB**;
 - Selecionar um **Security Group** com :
   - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = MYSQL/Aurora&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 3306;
 - Selecionar **Create Database**;
<br>

Para possiblitar a conexão da database MySQL com a aplicação WordPress, é importante criar variáveis de ambiente no contêiner como  :
- **WORDPRESS_DB_HOST= 'Database Endpoint'**;
- **WORDPRESS_DB_USER= 'Master Username'**;
- **WORDPRESS_DB_PASSWORD= 'Master Password'**;
- **WORDPRESS_DB_NAME= 'Database Name'**;

Também é necessário instalar o **default-mysql-client** no contêiner para permitir a conexão com a database;

A configuração destas variáveis foi feita pelo Script User Data :
```bash
#!/bin/bash
# sudo yum update
# sudo yum install -y nfs-utils
# sudo systemctl start nfs
# sudo systemctl enable nfs
# sudo amazon-linux-extras install docker -y
# sudo service docker start
# sudo systemctl enable docker
# docker volume create \
#     --driver local \
#     --opt type=nfs \
#     --opt o=addr=fs-0c64cd46913ded989.efs.us-east-1.amazonaws.com,rw,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
#     --opt device=:/ efs
# mkdir -p /Dockerfile
# echo "FROM wordpress:latest
RUN apt-get update && apt-get install -y default-mysql-client
ENV WORDPRESS_DB_HOST=database-1.chj99ulhygmu.us-east-1.rds.amazonaws.com
ENV WORDPRESS_DB_USER=admin
ENV WORDPRESS_DB_PASSWORD=adminadmin
ENV WORDPRESS_DB_NAME=WPDatabase
# WORKDIR /var/www/html
# EXPOSE 80
# RUN chown -R www-data:www-data /var/www/html" >> /Dockerfile/dockerfile
# docker build -t mywordpress /Dockerfile
# docker run -d \
#   --name wordpress \
#   -v  efs:/var/www/html/ \
#   -p 80:80 \
#   --restart always \
#   mywordpress 
```
<br>
<br>
<br>

### •	 Configuração de um sistema de arquivos EFS para estáticos da aplicação do contêiner WordPress
Primeiro deve-se criar um sistema de arquivos EFS :
- Na Console da AWS acessar o recurso **EFS** e selecionar **Create File System**;
- Informe um nome e selecione **Create**;
- Selecione um Security Group com :
  - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = NFS&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 2049
<br>

Em seguida, a montagem do EFS no contêiner compartilhando o arquivo "/var/www/html/" é feito pelo Script User Data :
- Primeiramente, é criado um volume docker com o EFS criado anteriormente :
  - docker volume create \
        --driver local \
        --opt type=nfs \
        --opt o=addr='**DNS EFS**',rw,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
        --opt device=:/ '**Nome do volume**'
- Em seguida, na criação do contêiner é informado o volume gerado e o local onde será montado no contêiner :

   -v volume:/caminho/do/dir/a/compartilhar;
  
   -v  efs:/var/www/html/;

```bash
#!/bin/bash
# sudo yum update
# sudo yum install -y nfs-utils
# sudo systemctl start nfs
# sudo systemctl enable nfs
# sudo amazon-linux-extras install docker -y
# sudo service docker start
# sudo systemctl enable docker
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=fs-0c64cd46913ded989.efs.us-east-1.amazonaws.com,rw,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
    --opt device=:/ efs
# mkdir -p /Dockerfile
# echo "FROM wordpress:latest
# RUN apt-get update && apt-get install -y default-mysql-client
# ENV WORDPRESS_DB_HOST=database-1.chj99ulhygmu.us-east-1.rds.amazonaws.com
# ENV WORDPRESS_DB_USER=admin
# ENV WORDPRESS_DB_PASSWORD=adminadmin
# ENV WORDPRESS_DB_NAME=WPDatabase
# WORKDIR /var/www/html
# EXPOSE 80
# RUN chown -R www-data:www-data /var/www/html" >> /Dockerfile/dockerfile
# docker build -t mywordpress /Dockerfile
# docker run -d \
#  --name wordpress \
  -v  efs:/var/www/html/ \
#   -p 80:80 \
#   --restart always \
#   mywordpress 
```
<br>
<br>
<br>

### •	 Configuração do Load Balancer
O Load Balancer pode ser gerado durante a criação do **Auto Scaling Group** ou separado no **Load Balancers**, também um recurso **EC2** :
- No painel **EC2** selecione o recurso **Load Balancers** e **Create Load Balancer**;
- Selecionar um dos tipos de Load Balancer. A sugestão era **Classic Load Balancer**;
- Informar um nome para o Load Balancer, selecionar a opção **Internet-facing**;
- Informar :
  - **VPC** : padrão;
  - **Subnets** : subnets públicas no **us-east-1a** e **us-east-1b**;
- Selecionar um Security Group com : 
  - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = HTTP&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 80;
  - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = HTTPS&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 80;
- Criar um **Listener = Listener  HTTP:80&nbsp;&nbsp;&nbsp;&nbsp;Instance   HTTP:80**;
- Modificar o Ping Path do **Health Checks** para '**/wp-admin/install.php**';
- Selecionar **Create Load Balancer**;
> [!IMPORTANT]
> Após criado o **Classic Load Balancer**, é importante ativar a opção **Cookie stickiness** e adicionar um certo **período de tempo** nas **Configurações do Listener do Load Balacer**. Isso resolvera o problema de sessão persistente na página login da console do WordPress. 

<br>
<br>
<br>

### •	 Configuração do Auto Scaling Group
Para criar um Auto Scaling Group é necessário um template das instâncias que ira gerar :
- Na console da AWS nos recursos **EC2** selecionar **Launch Templates**;
- Selecionar **Create Launch Template**;
- Informar :
  - nome;
  - descrição;
  - AMI : Amazon Linux 2;
  - Tipo de instância : T2.micro;
  - Chaves de segurança;
  - grupos de segurança :
    - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = SSH&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 22;
    - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = MYSQL/Aurora&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 3306;
    - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = Custom TCP&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 111;
    - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = Custom UDP&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = UDP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 111;
    - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = NFS&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 2049;

- Inserir no User Data o Script para preparação do ambiente da instância :
```bash
#!/bin/bash
sudo yum update
sudo yum install -y nfs-utils
sudo systemctl start nfs
sudo systemctl enable nfs
sudo amazon-linux-extras install docker -y
sudo service docker start
sudo systemctl enable docker
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=fs-0c64cd46913ded989.efs.us-east-1.amazonaws.com,rw,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
    --opt device=:/ efs
mkdir -p /Dockerfile
echo "FROM wordpress:latest
RUN apt-get update && apt-get install -y default-mysql-client
ENV WORDPRESS_DB_HOST=database-1.chj99ulhygmu.us-east-1.rds.amazonaws.com
ENV WORDPRESS_DB_USER=admin
ENV WORDPRESS_DB_PASSWORD=adminadmin
ENV WORDPRESS_DB_NAME=WPDatabase
WORKDIR /var/www/html
EXPOSE 80
RUN chown -R www-data:www-data /var/www/html" >> /Dockerfile/dockerfile
docker build -t mywordpress /Dockerfile
docker run -d \
  --name wordpress \
  -v  efs:/var/www/html/ \
  -p 80:80 \
  --restart always \
  mywordpress 
```  
 > [!NOTE]
> Para permitir acesso ao WordPress somente via Load Balancer, o Security Group do Launch Template deve conter :
> - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = HTTP&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 80&nbsp;&nbsp;&nbsp;&nbsp; **Sourcer**= 'Id do Security Group do Load Balancer';
> - **Inbound Rule** &nbsp;&nbsp;&nbsp;&nbsp;: &nbsp;&nbsp;&nbsp;&nbsp;**Type** = HTTPS&nbsp;&nbsp;&nbsp;&nbsp;**Protocol** = TCP&nbsp;&nbsp;&nbsp;&nbsp; **Port** = 80&nbsp;&nbsp;&nbsp;&nbsp; **Sourcer**= 'Id do Security Group do Load Balancer';

<br>

Para criar o **Auto Scaling Group** :
- Na console da AWS nos recursos **EC2** selecionar **Auto Scaling Groups**;
- Selecionar **Create Auto Scaling Group**;
- Inserir nome para o Auto Scaling e informar o **Launch Template** criado;
- Informar :
  - **VPC** : padrão;
  - **Subnets** : subnets públicas no **us-east-1a** e **us-east-1b**;
- Selecionar o **Load Balancer** criado anteriormente;
- Informa a quantidade de instâncias a ser criada. No caso 1 em cada AZ diferentes :

  **Desired capacity** = 2&nbsp;&nbsp;&nbsp;&nbsp; **Minimum capacity** = 2&nbsp;&nbsp;&nbsp;&nbsp; **Maximum capacity** = 2;
- Selecionar **Create Auto Scaling Group**;
<br>

Após configurar todo a arquitetura, o Auto Scaling ira gerar as instâncias de acordo com o Launch Template informado.
**É importante usar o DNS do Load Balancer na primeira configuração do Usuário da console WordPress na web, pois assim a database ira salvar como URL do site o DNS do Load Balancer.**
