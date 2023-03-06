<h1>Atividade Prática Sobre Docker e AWS - PB COMPASS.UOL</h1>

> Status da Atividade: :heavy_check_mark: (Concluída)

## Tópicos

* [Descrição da atividade](#descrição-da-atividade)

* [Criação do user-data](#criação-do-user-data)

* [Criação do arquivo Docker Compose para execução dos containers](#criação-do-arquivo-docker-compose-para-execução-dos-containers)

* [Criação de uma Bastion Host](#criação-de-uma-bastion-host)

* [Conclusão](#conclusão)

## Descrição da atividade

A atividade consiste na criação de uma instância EC2, onde será feita a instalação do Docker e do Docker Compose para efetuar o Deploy de uma aplicação WordPress, utilizando o serviço de Load Balancer para realizar o acesso na internet. A seguir uma breve descrição dos passos necessários para execução da atividade:

1. Instalação e configuração do Docker e do Docker-compose no host EC2 com a utilização de um script(user-data.sh);

2. Efetuar o Deploy de uma aplicação WordPress com um container de aplicação e um container database MySQL;

3. Configurar a utilização do serviço EFS AWS para armazenar os estáticos do container da aplicação WordPress;

4. Configurar o serviço de Load Balancer AWS para a aplicação WordPress.

* Observação: Não utilizar IP público para saída do serviço WordPress. O acesso a aplicação WordPress só pode ser realizado por meio do DNS do Load Balancer.

## Criação do user-data

Ao executar uma nova instância no Amazon EC2, existe a opção de passar dados de usuário que podem realizar tarefas de configuração comuns automatizadas e até mesmo executar scripts após a inicialização da instância. Essas informações são inseridas em um campo chamado "user data" localizado dentro das configurações detalhadas da instância (Advanced Details").

Para realização da atividade, o user data terá script Bash que automatiza a instalação e configuração do Docker e do Docker Compose em uma instância EC2 da AWS e também cria um volume EFS e inicia um conjunto de contêineres com base em um arquivo docker-compose.yml. Aqui está uma explicação detalhada das etapas realizadas pelo script:

```
#!bin/bash
# Indica que o interpretador do script é o bash.

yum update -y
# Atualiza todos os pacotes instalados na instância.

yum install -y amazon-efs-utils
# Instala o utilitário EFS na instância.

yum install -y docker
# Instala o Docker na instância.

systemctl start docker
# Inicia o serviço do Docker.

systemctl enable docker
# Configura o Docker para ser iniciado automaticamente quando a instância é iniciada.

usermod -aG docker ec2-user
# Adiciona o usuário "ec2-user" ao grupo "docker".

chkconfig docker on
# Configura o Docker para ser iniciado automaticamente quando a instância é iniciada.

curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
# Baixa a última versão do Docker Compose.

chmod +x /usr/local/bin/docker-compose
# Dá permissão de execução ao arquivo baixado.

mv /usr/local/bin/docker-compose /bin/docker-compose
# Move o arquivo baixado para a pasta /bin.

curl -sL https://raw.githubusercontent.com/Vandielson/Atividade_Docker_AWS_PB_COMPASS/main/Docker-compose.yml --output /home/ec2-user/docker-compose.yml
# Baixa o arquivo docker-compose.yml de um repositório no GitHub e salva na pasta /home/ec2-user.

mkdir -p /mnt/efs/vandielson/var/www/html
# Cria uma pasta para armazenar os arquivos do WordPress no volume EFS.

mount -t efs fs-06b0d9af54c842dd6.efs.us-east-1.amazonaws.com:/ /mnt/efs
# Monta o volume EFS na instância.

chown ec2-user:ec2-user /mnt/efs
# Altera o proprietário da pasta /mnt/efs para o usuário "ec2-user".

echo "fs-06b0d9af54c842dd6.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs defaults 0 0" >> /etc/fstab
# Adiciona uma linha ao arquivo /etc/fstab para que o volume EFS seja montado automaticamente na inicialização da instância.

docker-compose -f /home/ec2-user/docker-compose.yml up -d
# Inicia um conjunto de contêineres com base nas instruções do arquivo docker-compose.yml.
```

## Criação do arquivo Docker Compose para execução dos containers

Este é um arquivo de configuração do Docker Compose, que é usado para definir e executar aplicativos Docker em vários contêineres. Para está atividade será utilizado um script para executar uma aplicação WordPress e um banco de dados MySQL em dois containers.

Script utilizado para criação dos Containers:

```
version: '3.3'
services:
  db:
    image: mysql:latest
    restart: always
    environment:
      TZ: America/Recife
      MYSQL_ROOT_PASSWORD: teste
      MYSQL_USER: teste
      MYSQL_PASSWORD: teste
      MYSQL_DATABASE: wordpress
    ports:
      - "3306:3306"
    networks:
      - wordpress-network
  
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "80:80"
    restart: always
    volumes:
      - /mnt/efs/vandielson/var/www/html:/var/www/html
    environment:
      TZ: America/Recife
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: teste
      WORDPRESS_DB_PASSWORD: teste
    networks:
      - wordpress-network

networks:
  wordpress-network:
    driver: bridge
```

Aqui está uma explicação de cada parte do script:

* version: '3.3': especifica a versão do formato do arquivo de configuração do Docker Compose que está sendo usado. Neste caso, é a versão 3.3.

* services: é uma seção onde são definidos os serviços do aplicativo. Cada serviço representa um contêiner Docker. 

* db: é o nome do primeiro serviço. Este serviço usa a imagem mais recente do MySQL disponível no Docker Hub, define algumas variáveis de ambiente para configurar o banco de dados, expõe a porta 3306 do contêiner e se conecta à rede "wordpress-network".

* image: define a imagem do contêiner que será usada para o serviço. O Docker irá baixar a imagem, se ela ainda não estiver disponível localmente.

* restart: é uma opção que define a política de reinicialização do contêiner. Neste caso, o valor "always" indica que o contêiner será sempre reiniciado, independentemente do motivo da parada.

* environment: é uma opção que define variáveis de ambiente para o contêiner.

* TZ: define o fuso horário para o contêiner.

* YSQL_ROOT_PASSWORD: define a senha do usuário root do MySQL.

* MYSQL_USER: define o nome de usuário do MySQL.

* MYSQL_PASSWORD: define a senha do usuário do MySQL.

* MYSQL_DATABASE: define o nome do banco de dados MySQL.

* WORDPRESS_DB_HOST: define o host do banco de dados do WordPress.

* WORDPRESS_DB_NAME: define o nome do banco de dados do WordPress.

* WORDPRESS_DB_USER: define o nome de usuário do banco de dados do WordPress.

* WORDPRESS_DB_PASSWORD: define a senha do usuário do banco de dados do WordPress.

* ports: é uma opção que mapeia as portas do contêiner para as portas do host.

* wordpress: é o nome do segundo serviço. Este serviço usa a imagem mais recente do WordPress disponível no Docker Hub, define algumas variáveis de ambiente para se conectar ao banco de dados, expõe a porta 80 do contêiner, cria um volume para armazenar os arquivos do WordPress e se conecta à rede "wordpress-network". Este serviço também depende do serviço "db".

* depends_on: é uma opção que define uma dependência entre serviços. Neste caso, o serviço "wordpress" depende do serviço "db", o que significa que o serviço "db" será iniciado antes do serviço "wordpress".

* Valumes: é uma opção que cria um volume para armazenar os arquivos da aplicação.

* /mnt/efs/vandielson/var/www/html:/var/www/html: é o volume que será criado para armazenar os arquivos do WordPress. Ele mapeia o diretório /mnt/efs/vandielson/var/www/html no host para o diretório /var/www/html no contêiner.

* networks: é uma opção que define as redes em que o contêiner estará conectado.

* wordpress-network: é o nome da rede criada para o aplicativo.

* driver: define o driver da rede. Neste caso, o valor "bridge" indica que o Docker usará a rede padrão do Docker para o aplicativo.


## Criação de uma Bastion Host

Como um dos requisitos da atividade é não utilizar IP público na instância e somente realizar o acesso a aplicação WordPress por meio de um Load Balancer, então a solução mais adequada é a criação de um Bastion Host na AWS (Amazon Web Services) com o uso de um Load Balancer para acessar a aplicação na instância privada. A Bastion Host é um servidor seguro que atua como um ponto de entrada para acessar sua rede privada de dentro da nuvem AWS. A seguir serão descritos os passos necessários para criação do Bastion Host.

### Passo 1: Configurar a VPC

1. Para começar, é necessário criar uma VPC (Virtual Private Cloud) na AWS. Acesse o console da AWS e clique em "VPC" no menu principal. Em seguida, clique em "Criar VPC".

2. Preencha o nome da sua VPC e o CIDR (Classless Inter-Domain Routing) para a sua rede privada. Clique em "Criar" para criar a sua VPC.

## Passo 2: Criar as Sub-redes

Após a criação da VPC, é necessário criar as sub-redes: duas públicas e duas privadas.

* Observação: As sub-redes precisam ser criadas em zonas de disponibilidade diferentes para aumentar a disponibilidade e a resiliência da sua aplicação. Você pode selecionar a zona de disponibilidade desejada ao criar a sub-rede.

Para criar as sub-redes, siga os passos abaixo:

1. Acesse o console da AWS e clique em "VPC" no menu principal.

2. Clique em "Sub-redes" no menu lateral esquerdo.

3. Clique em "Criar Sub-rede".

3. Preencha o nome da sub-rede, selecione a VPC que você criou anteriormente e informe o CIDR para a sua sub-rede.

4. Clique em "Criar" para criar a sub-rede.

6. Repita o processo para criar as demais sub-redes.

* Importante: Lembre-se de que as sub-redes criadas devem estar na mesma VPC, mas em diferentes zonas de disponibilidade. Isso garante que as instâncias sejam distribuídas em diferentes zonas de disponibilidade para evitar possíveis interrupções devido a falhas em uma única zona de disponibilidade.

### Passo 3: Criar as tabelas de roteamento para as sub-redes

As tabelas de roteamento são componentes fundamentais da infraestrutura de rede da Amazon Web Services (AWS). Elas permitem que você controle o fluxo de tráfego em sua rede, especificando o destino de cada pacote de dados. Cada tabela de roteamento contém uma série de regras de roteamento que determinam para onde o tráfego deve ser encaminhado.

As regras de roteamento geralmente especificam o tipo de tráfego (por exemplo, tráfego da Internet ou tráfego interno da VPC), a sub-rede de origem e a sub-rede de destino do tráfego. Com base nessas informações, a tabela de roteamento determina a melhor rota para encaminhar o tráfego.

Serão necessárias duas tabelas de roteamento, uma para as sub-redes públicas e outra para as privadas. Para criar as tabelas de roteamento na AWS, siga os passos abaixo:

1. Ainda dentro da aba VPC, clique em "Tabelas de roteamento" no menu lateral esquerdo.

2. Clique em "Criar tabela de roteamento".

3. Preencha o nome da tabela de roteamento e selecione a VPC que você criou anteriormente.

4. Clique em "Criar".

5. Selecione a tabela de roteamento que você acabou de criar e clique em "Editar rotas".

6. Adicione uma regra de roteamento para cada sub-rede que você criou anteriormente. Cada regra de roteamento deve especificar a sub-rede de destino e o gateway de internet (se o destino for a Internet) ou um gateway NAT (NAT – Conversão de endereços de rede). Você pode usar um gateway NAT para que as instâncias em uma sub-rede privada possam se conectar a serviços fora da VPC, mas os serviços externos não podem iniciar uma conexão com essas instâncias. 

7. Clique em "Salvar rotas".

* Observação: Para a tabela de rotas com as sub-redes públicas utilize o gateway de internet e para a tabela com as privadas utilize gateway NAT.

Ao criar as tabelas de roteamento, você está especificando como o tráfego de rede é encaminhado em sua VPC. Isso permite que você controle o acesso de rede em sua infraestrutura, aumente a segurança e a eficiência da rede e atenda aos requisitos de suas aplicações.

### Passo 4: Criar a Bastion Host

1. Agora é preciso criar a Bastion Host. Clique em "Instâncias" no menu principal e em seguida, clique em "Launch Instance".

2. Escolha uma AMI (Amazon Machine Image) e selecione o tipo de instância que você deseja criar. Depois escolha a VPC e a sub-rede pública que você criou anteriormente.

3. Selecione a opção de criar um grupo de segurança e em seguida, adicione uma regra para permitir o tráfego SSH por meio da porta 22 da sua máquina local para a Bastion Host. Por fim, clique em "Launch" para iniciar a instância.

### Passo 5: Criar a Instância Privada

Agora vamos criar a Instância Privada:

1. Clique em "Instâncias" no menu principal e em seguida, clique em "Launch Instance".

2. Escolha uma AMI (Amazon Machine Image) e selecione o tipo de instância que você deseja criar. Na próxima tela, escolha a VPC e a sub-rede privada que você criou anteriormente.

3. Selecione a opção de criar um grupo de segurança e insira todas as regras necessárias para o pleno funcionamento da aplicação WordPress e o banco de dados MySQL. Realizar a liberação das portas: 22/TCP, 111/TCP e UDP, 2049/TCP e UDP, 80/TCP, 443/TCP, 3306/TCP.

* Atenção: Todas as portas podem estar liberadas para qualquer IP, porém a porta 80 deve ser liberada apenas para o IP da sua Bastion Host, deixando-a ainda mais segura.

### Passo 6: Configurar o Load Balancer

Crie um Application Load Balancer para distribuir o tráfego entre a instância bastion e a instância privada. Para isso, siga os passos abaixo:

1. Acesse o console da AWS e clique em "EC2" no menu principal.

2. Clique em "Load Balancers" no menu lateral esquerdo.

3. Clique em "Criar Load Balancer".

4. Selecione "Application Load Balancer" e clique em "Criar".

5. Preencha as configurações básicas do load balancer, como nome, esquema, endereços IP, etc.

6. Em "Configurar segurança", selecione ou crie um grupo de segurança para o load balancer. Só é necessário que tenha soment apenas a porta 80. Este grupo de segurança será usado para controlar o tráfego para o load balancer.

7. Em "Selecionar sub-redes", selecione as sub-redes públicas que você criou anteriormente. Certifique-se de selecionar sub-redes em pelo menos duas zonas de disponibilidade diferentes.

8. Em "Configurar grupos de destino", crie um novo grupo de destino e selecione a instância que você criou anteriormente (a instância privada). Certifique-se de atribuir pesos diferentes para cada instância, a fim de equilibrar a carga de trabalho.

9. lique em "Criar" para criar o Application Load Balancer.

Ao criar o Application Load Balancer, você está configurando um serviço de balanceamento de carga que distribui o tráfego de rede entre as instâncias bastion e privada. Isso aumenta a disponibilidade e a escalabilidade da sua aplicação, além de melhorar o desempenho e a segurança da rede.

### Passo 7: Acessar a aplicação

Agora você pode acessar a aplicação na Instância Privada através do Load Balancer. Use o DNS do Load Balancer para acessar a aplicação. Se você precisar acessar a Bastion Host, use o SSH para se conectar à Bastion Host primeiro e, em seguida, use o SSH para se conectar à Instância Privada.

## Conclusão

Ao realizar todos os passos da maneira correta, você terá acesso a aplicação por meio do DNS do Load Balancer. Exitem outras coisas que podem ser feitas para melhorar ainda mais a segurança do instância da aplicação. Porém, com todos os passos realizados as exigências da atividade estão cumpridas.