---
layout: post
title:  "Deployando Aplicação Web com Docker Swarm na AWS"
date:   2022-04-19 11:11:00
categories: 
---


#### Escopo

Vou demonstrar como fazer o deploy de uma aplicação web em cluster de Docker Swarm na AWS.
Para alcançar esse objetivo, usarei as seguintes tecnologias:

Docker (modo Swarm)

As versões atuais do Docker incluem o modo swarm para gerenciar nativamente um cluster de Docker Engines chamado "swarm". Apenas com a CLI do Docker é possível criar um swarm e implantar serviços (containers) em vários hosts docker que trabalham em cluster. Dessa forma, iremos usar cluster swarm para manter os containers da aplicação em alta disponibilidade, assim garantindo tolerência a falhas até mesmo se um dos hosts vier a cair.

Amazon Elastic Compute Cloud (EC2)

Amazon Elastic File System (EFS)

Amazon Relational Database Service (RDS) 

Traefik

##### Arquitetura

imagem


#### Hora de Deployar!

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

