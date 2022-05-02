---
layout: post
title:  "Deployando uma Aplicação Web com Docker Swarm na AWS"
date:   2022-04-19 11:11:00
categories: 
---

Vou demonstrar como fazer o deploy de uma aplicação web Wordpress em um cluster de Docker Swarm na AWS.
Para alcançar esse nível de arquitetura, usarei as seguintes tecnologias:

- Docker (modo Swarm) 

As versões atuais do Docker incluem o modo swarm para gerenciar nativamente um cluster de Docker Engines chamado "swarm". Apenas com a CLI do Docker é possível criar um swarm e implantar serviços (containers) em vários hosts docker que trabalham em cluster. Dessa forma, iremos usar cluster swarm para manter os containers da aplicação em alta disponibilidade, assim garantindo tolerência a falhas até mesmo se um dos hosts vier a cair.

- Amazon Elastic Compute Cloud (EC2) 

O EC2 é um serviço web que disponibiliza capacidade computacional segura e redimensionável na nuvem. Em nosso sistema, teremos 3 instâncias EC2 e cada uma será um nó do cluster Swarm.

- Amazon Elastic File System (EFS) 

O EFS é um sistema de arquivos elástico simples, sem servidor, compatível com o protocolo Network File System versão 4 (NFSv4.1 e NFSv4.0). Várias instâncias, incluindo Amazon EC2, Amazon ECS e AWS Lambda, podem acessar um sistema de arquivos do Amazon EFS ao mesmo tempo. Usaremos o EFS para armazenar os dados do nosso volume Docker.

- Amazon Relational Database Service (RDS)  

O RDS é uma coleção de serviços gerenciados que facilita a configuração, operação e escalabilidade de bancos de dados na nuvem.Então disponível várias engines para uso no RDS, desde Amazon Aurora, MySQL, MariaDB, PostgreSQL, Oracle e SQL Server. Neste lab usarei a engine do MySQL 8.0.11, ao qual o Wordpress se conectará para uso da base da dados.

- Traefik 

O Traefik é um Edge Router, pode utuar como uma espécie de proxy, que torna a publicação de serviços uma experiência fácil. Ele recebe solicitações em nome do seu sistema e descobre quais componentes são responsáveis por tratá-las. O Traefik também é nativo para uso com Docker, e estará presente em cada um dos nós do Swarm. Dessa forma será possível servir nossa aplicação indenpende de qual nó do Swarm executa o serviço. 

- Wordpress 

WordPress é uma ferramenta open source de web sites e blogs e um Sistema de Gerenciamento de Conteúdo (CMS). Neste lab o WordPress será usado para exemplificar o funcionamento de uma aplicação web real.

##### Arquitetura da Aplicação Web

![My helpful screenshot](/assets/images/architecture.png)


##### Configuração das instâncias AWS EC2

O cluster do Swarm será composto por 3 instâncias do EC2, como sistema operacional base de cada instância 
utilizei o Ubuntu 20.04.3 LTS.

Instance Name                                   
- node03-swarm.us-east-1.prod.aws.godric.com
- node02-swarm.us-east-1.prod.aws.godric.com
- node01-swarm.us-east-1.prod.aws.godric.com

Private IPv4 
- node01 - 172.31.88.147
- node02 - 172.31.29.135
- node03 - 172.31.26.68

As três instâncias estão na mesma VPC (Virtual Private Cloud) sob o CIDR 172.31.0.0/16.
Além disso, atrelado a VPC onde estão os hosts EC2, criei um Security Group para realizar as devidas liberações das portas dos serviços e protocolos.


##### Configuração do Docker Engine

Vou realizar a instalação do Docker Engine conforme a [documentação][docker_install] ofical do docker para Ubuntu Server. Essa configuração é válida para os três hosts EC2 que irão compor nosso swarm.

Existem muitas maneiras de instalar o docker, mas usarei o metódo via repositórios do Docker e instalar a partir deles, para facilitar as tarefas de instalação e atualização. Esta é a abordagem recomendada.

`1. Atualize o índice de pacotes do apt e instale os pacotes para permitir que o apt use um repositório sobre HTTPS:`

```bash
$ sudo apt-get update
```

```bash
$ sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

`2. Adicione a chave GPG oficial do Docker:`

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

`3. Use o comando a seguir para configurar o repositório estável. Para adicionar o repositório nightly ou test, adicione a palavra nightly ou test (ou ambos) após a palavra stable nos comandos abaixo.`

```bash
$  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
`4. Instalando o docker.`
```bash
$ sudo apt-get update
```

```bash
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

`5. Verifique a versão do docker instalada.`

```bash
$ docker --version
```

Realize a mesma configuração nas demais instâncias EC2.

[docker_install]: https://docs.docker.com/engine/install/ubuntu/

##### Configuração do Docker Swarm

No Swarm cada host pode assumir a função de worker, a entidade que executa as cargas de trabalho gerais ou manager, a entidade que gerencia o cluster swarm e executa cargas de trabalho relacionadas a gerência do mesmo. 

No nosso cenário, temos apenas 3 nós EC2 e dessa forma todos terão função de manager. Existem uma série de boas práticas a serem seguidas na criação de um cluster swarm, mas não irei abordar nesse artigo.

Os próximos passos devem ser aplicados no node01:

`1. Ativando modo swarm no host.`

```bash
$ docker swarm init --advertise-addr 172.31.88.147 # IP node01
```

A saída do comando terá as instruções para adicionar novos nós no swarm.
Para que haja comunicação entre os nós do swarm libere as seguintes portas no Security Group da VPC:

- `2377/tcp`
- `7946/tcp`
- `7946/udp`
- `4789/udp`

`2. Obtendo token para adicionar o node02 e node03 como nós do tipo manager.`

```bash
$ docker swarm join-token manager
```

Use o token para incluir os demais hosts no swarm.

`3. Para validar se os três hosts estão no swarm, use o comando:`

```bash
$ docker node ls
```

A saída deve ser algo parecido com:
```bash
ID          HOSTNAME           STATUS    AVAILABILITY   STATUS   
fpi8hbnk    ip-172-31-26-68    Ready     Active         Reachable        
qjp3jwr2    ip-172-31-29-135   Ready     Active         Reachable        
lb8wsbvq *  ip-172-31-88-147   Ready     Active         Leader           
```

Perceba que um dos hosts está com uma marcação `*`, isso além do próprio status, indica que esse nó é o líder do swarm. No momento que houve uma falha, o swarm automaticamente irá eleger um dos dois outros nós como líder.

Nosso cluster já está pronto! Agora temos que configurar a partição onde ficará os dados do volume docker no AWS EFS e base da dados da aplicação no AWS RDS.

`4. Crie uma rede docker com o drive overlay para a aplicação no Swarm`

```bash
docker network create --driver overlay webapp-network
```

##### Criando o Elastic File System (EFS)

No menu stogare, no painel da AWS, crie um novo Elastic File System. Lembre de atribuir ao EFS a mesma VPC e Security Group onde foram criadas as instâncias do EC2 se conectem no sistema de arquivos.

Name: `app-data-vol`

Antes de conectar no EFS, instale o pacote cliente do NFS nos hosts, dessa forma eles poderão conectar no sistema de arquivos. 

```bash
$ apt install nfs-common
```

Primeiramente, crie nos três hosts do swarm um diretório com caminho `/mnt/app-data-vol/`. Copie o comando no painel do EFS para fazer o attach do sistema de arquivos nas instâncias EC2.

O comando será algo parecido com:

```bash
$ sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 172.31.19.25:/ /mnt/app-data-vol/ # No comando tem o IP do EFS 
```

Provalmente, ocorrerá um erro de timeout na conexão da instância EC2 com o EFS, isso se deu devido não termos liberado a porta `2049` do protocolo NFSv4 no Security Group. Após esse ajuste, a conexão deve acontecer sem problemas.

Ainda temos que resolver mais um detalhe, o comando mount que usamos, de fato, montará o sistema de arquivos em `/mnt/app-data-vol/`, porém a configuração só funcionará até que as intâncias forem reiniciadas. Para tornamos permante a montagem do EFS, devemos adicionar a configuração no arquivo `/etc/fstab` em cada um dos hosts.

Veja o meu exemplo:

```bash
$ vim /etc/fstab
```
```bash
172.31.19.25:/ /mnt/app-data-vol   nfs4    nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport     0 0
```

##### Configuração do Banco de Dados (RDS)

Crie uma base de dados no menu do RDS. Nesse lab será utilizado a engine do MySQL. Seguirei a seguinte configuração:

```bash
Endpoint: wordpressdb.example.rds.amazonaws.com
Engine: MySQL 8.0.11
Port: 3306
Database: wordpress
User: admin
Password: *****************************
```

##### Deployando a Aplicação na AWS

As imagens oficiais do Traefik e WordPress estão disponíveis no Docker Hub. 

O deploy será feito com base no arquivo de Docker Compose, o arquivo será passado como argumento do comando de inicialização da stack do Docker Swarm, conforme o exemplo:

Obs.: Como boa prática, execute o comando de deploy no host EC2 que é o líder do cluster swarm.

```bash
$ docker stack deploy -c docker-compose.yaml webapplication 
# Sintaxe: docker stack deploy [-c arquivo-compose] [nome-da-stack]
```

Para visualizar o status dos serviços no Swarm use o comando:



```bash
$ docker service ls
```

A saída deve ser algo parecido com:

```bash
ID       NAME                       MODE         REPLICAS   IMAGE              PORTS
rn4***   webapplication_traefik     global       3/3        traefik:v2.2.1     *:80->80/tcp, *:843->443/tcp
hc2***   webapplication_wordpress   replicated   1/1        wordpress:latest]
```

Para melhor visualização do arquivo `docker-compose.yaml`, adicionei-o no meu [GitHub][github_kessix] no repositório [webapp-dockerswarm-aws][repo], confere lá!

[github_kessix]: https://github.com/kessix
[repo]: https://github.com/kessix/webapp-dockerswarm-aws/blob/main/docker-compose-wordpress-swarm.yaml

Note que no docker-compose, eu defini dois serviços: `traefik` e `wordpress`. 

O primeiro é o serviço do traefik, que será replicado de modo `global`, em outras palavras isso significa que cada nó do swarm terá um container do traefik, dessa forma faremos a descoberta do serviço em todas as EC2.

```bash
services:
  traefik:
    image: traefik:v2.2.1
    deploy:
      mode: global
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s
      placement:
        constraints:
          - node.role == manager
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.traefik.rule=Host(`traefik.godric.local`)"
        - "traefik.http.services.justAdummyService.loadbalancer.server.port=1337"
        - "traefik.http.routers.traefik.service=api@internal"
    ports:
      - "80:80"
      - "843:443"
    networks:
      - "webapp-network"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - "--api=true"
      - "--log.level=DEBUG"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.file.directory=/etc/traefik/dynamic"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
```

Observe que não fizemos exposição de portas para o serviço do WordPress, isso graças ao `labels` que define a URL que será resolvida pela nosso proxy Traefik!

O segundo serviço definido no arquivo compose, faz o deploy a aplicação web WordPress, note que a âncora `template-deploy` fará com que o serviço do WordPress tenha apenas uma réplica, isso faz total sentido, uma que vez que será necessário apenas um container da aplicação em execução por vez em cada host EC2. Logo, se ocorrer uma falha no container atual, o swarm subirá um novo container no "melhor" host do cluster. 

```bash
services:
 [...]
  wordpress:
    image: wordpress
    environment:
      WORDPRESS_DB_HOST: wordpressdb.example.us-east-1.rds.amazonaws.com
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: ***********************
      WORDPRESS_DB_NAME: wordpress
    networks:
      - "webapp-network"
    volumes:
      - /mnt/app-data-vol:/var/www/html
    deploy: 
      <<: *template-deploy
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.wordpress.entrypoints=web"
        - "traefik.http.routers.wordpress.rule=Host(`wordpress.godric.local`)"
        - "traefik.http.services.wordpress.loadbalancer.server.port=80"
```

O serviço do WordPress, possui um volume do tipo `bind mount` apontando para nossa área do AWS EFS, logo os arquivos da nossa aplicação web serão armazenados em um volume no EFS.

Por fim, definidas as variáveis de ambiente no arquivo compose, conectamos a aplicação Web na base de dados criada no AWS RDS, o campo mais importante é endpoint que deve ser exatamente igual ao criado na instância RDS. Caso a VPC das instâncias EC2 seja diferente da VPC onde está a instância RDS, será necessário realizar as devidas liberações na configuração da instância RDS para que seja possível conectar na base de dados.

Para testar o funcionamento da aplicação adicione as hostnames configurados no `label` do Traefik em sua zona de DNS. Neste lab não abordarei a configuração da entradas DNS e nem o serviço da AWS Route53. Como dica, para testar o acesso à aplicação web de forma simples, adicione as entredas em seu arquivo hosts que está presente em máquinas com sistemas Linux ou Windows. No meu Linux, ficaria algo parecido com:

```bash
# wordpress-in-swarm
[ip-public-EC2] traefik.godric.local
[ip-public-EC2] wordpress.godric.local
```

Com todos esses passos alcançamos o objetivo e agora temos uma sistema web em alta disponibilidade na AWS, desde a  aplicaçao Web com arquivos no AWS EFS, até a base de dados gerencida no AWS RDS. 

Infelizmente não é possível abordar todos os detalhes relacionados a configuração do ambiente, devido o nível de detalhes que seria necessário para tal. Dessa forma, me coloco a disposição para exclarecer dúvidas refentes ao ambiente aqui provisionado.

Até a próxima! :D


