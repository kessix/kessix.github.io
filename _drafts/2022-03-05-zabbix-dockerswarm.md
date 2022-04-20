---
layout: post
title:  "Como implantar o Zabbix com Docker Swarm em Alta Disponibilidade"
date:   2022-03-05 18:52:00
categories: 
---
<br>

#### Configurando o NFS Server

##### Primeiro passo corrigir o horário do servidor
##### Verificar o timezone atual

```bash
timedatectl status
```

Output:

```bash
               Local time: Wed 2020-06-03 05:03:25 -03
           Universal time: Wed 2020-06-03 08:03:25 UTC
                 RTC time: Wed 2020-06-03 08:03:24
                Time zone: America/New_York (-03, -0300)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

##### Listar timezone disponiveis

```bash
timedatectl list-timezones | grep Sao_Paulo
```

Output:

```bash
America/Fortaleza
```

##### Definir timezone

```bash
timedatectl set-timezone America/Fortaleza
```

##### Verifique status do NTP

```bash
timedatectl status
```

```bash
               Local time: Wed 2020-06-03 05:03:25 -03
           Universal time: Wed 2020-06-03 08:03:25 UTC
                 RTC time: Wed 2020-06-03 08:03:24
                Time zone: America/Fortaleza (-03, -0300)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

##### Ajustes de S.O após a instalação

##### Verificar se é necessário atualização do S.O

```bash
dnf clean all
dnf check-update
```

##### Instalar utilitários

```bash
dnf install -y net-tools vim nano epel-release wget curl tcpdump
```

###### NFS-Server

##### Criar diretório para dados que serão compartilhados entrem os nodes docker

```bash
mkdir /data/data-swarm/
```

##### Instalar NFS

```bash
dnf install nfs-utils -y
```

##### Configurando exports

```shell
vim /etc/exports
/data/data-swarm/ *(rw,sync,no_root_squash,no_subtree_check)
exportfs -ar
```

##### Inicializando o serviço

```bash
systemctl enable --now nfs-server
systemctl status nfs-server
```

##### Liberando Firewall

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --reload
```

#### NFS-Client no CentOS

`LEMBRE-SE DE VERIFICAR A DATA E HORA DOS SERVIDORES`

##### Instalar utilitários no client

```bash
dnf install -y net-tools vim nano epel-release wget curl tcpdump
```

##### Instalar NFS no client

```bash
dnf install nfs-utils -y
```

##### Criar diretório para o ponto de montagem

```shell
mkdir -p /mnt/data-swarm/
```

##### Validando o ponto de montagem

```bash
mount -v -t nfs 172.0.0.60:/data/data-swarm/ /mnt/data-swarm/
```

##### Verifique se funcionou

```bash
df -h
Filesystem         Type      Size  Used Avail Use% Mounted on
devtmpfs           devtmpfs  215M     0  215M   0% /dev
tmpfs              tmpfs     233M     0  233M   0% /dev/shm
tmpfs              tmpfs     233M  6.4M  227M   3% /run
tmpfs              tmpfs     233M     0  233M   0% /sys/fs/cgroup
/dev/sda1          xfs        10G  4.8G  5.3G  48% /
172.0.0.60:/data/data-swarm nfs4       10G  3.5G  6.6G  35% /mnt/data-swarm
```

##### Removendo ponto de montagem temporario

```bash
umount /mnt/data-swarm
```

##### Configurando ponto de montagem permanente

```bash
vim /etc/fstab
172.0.0.60:/data/data-swarm   /mnt/data-swarm    nfs    defaults    0 0 
```

##### Force o reload do fstab

```bash
sudo mount -av
/                        : ignored
/boot                    : already mounted
swap                     : ignored
/var/lib/docker          : already mounted
/mnt/data-swarm          : already mounted
```

#### Configurando Docker Engines

##### Primeiro passo corrigir o horário do servidor

##### Verificar o timezone atual

```bash
timedatectl status
```

Output:

```bash
               Local time: Wed 2020-06-03 05:03:25 -03
           Universal time: Wed 2020-06-03 08:03:25 UTC
                 RTC time: Wed 2020-06-03 08:03:24
                Time zone: America/New_York (-03, -0300)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

##### Listar timezone disponiveis

```bash
timedatectl list-timezones | grep Sao_Paulo
```

Output:

```bash
America/Fortaleza
```

##### Definir timezone

```bash
timedatectl set-timezone America/Fortaleza
```

##### Verifique status do NTP

```bash
timedatectl status
```

```bash
               Local time: Wed 2020-06-03 05:03:25 -03
           Universal time: Wed 2020-06-03 08:03:25 UTC
                 RTC time: Wed 2020-06-03 08:03:24
                Time zone: America/Fortaleza (-03, -0300)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

##### Ajustes de S.O após a instalação

##### Verificar se é necessário atualização do S.O

```bash
dnf clean all
dnf check-update
```

##### Instalar utilitários

```bash
dnf install -y net-tools vim nano epel-release wget curl tcpdump
```

##### Instalando o Docker

##### Instalando repositório do docker

```bash
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```

##### Removendo repositórios antigos

```bash
dnf clean all
```

##### Listando versões disponiveis do docker

```bash
dnf list docker-ce --showduplicates | sort -r
```

##### Instalando device mapper

```bash
dnf install -y device-mapper-persistent-data
```

##### Instalando a ultima versão do Docker

`Se precisar da ultima versão do docker, é necessário instalar manualmente a ultima versão do containerd.io`

```bash
dnf install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
dnf install -y docker-ce
```

`Se não, utilize o parâmetro --nobest na hora de instalação`

```bash
dnf install -y docker-ce --nobest
```

##### Inicializando o daemon do docker

```bash
systemctl enable --now docker
systemctl status docker
```

##### Verificando versão do docker

```bash
docker version
```

##### Habilitar roteamento nos containers

```bash
firewall-cmd --zone=public --add-masquerade --permanent
firewall-cmd --reload
```

##### Ativando modo swarm no host

```bash
docker swarm init --advertise-addr 172.0.0.61
```

##### Inspect de todas as redes do docker

```bash
for net in `docker network ls |grep -v NAME | awk '{print $2}'`;do ipam=`docker network inspect $net --format {{.IPAM}}` && echo $net - $ipam; done
```

##### Validando conflito de rede

```bash
ifconfig ens18
```

Output:

```bash
eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.0.0.61  netmask 255.255.255.0  broadcast 172.0.0.255
        inet6 fe80::a00:27ff:fe21:23b8  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:21:23:b8  txqueuelen 1000  (Ethernet)
        RX packets 14379  bytes 1905170 (1.8 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 14602  bytes 6202564 (5.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
###### Obs.: Caso haja conflito entre a rede a rede docker e a rede ingress do swarm, faça as seguintes alterações:
<br/>

###### Criando uma rede ingress com range diferente

##### Listando os nodes

```bash
docker node ls
```

##### Alterando disponibilidade do node no swarm

`NESTE MOMENTO ELE NÃO IRA INICIAR NOVOS SERVIÇOS`

`<NODE_NAME> É O NOME DO HOSTNAME QUE RETORNA NO COMANDO docker node ls`

```bash
docker node update --availability drain <NODE_NAME>
```

##### Removendo rede ingress atual

```bash
$ docker network rm ingress
```

Output:

```bash
WARNING! Before removing the routing-mesh network, make sure all the nodes
in your swarm run the same docker engine version. Otherwise, removal may not
be effective and functionality of newly created ingress networks will be
impaired.
Are you sure you want to continue? [y/N]
```

##### Criando nova rede ingress

```bash
docker network create \
--driver overlay \
--ingress \
--subnet=192.168.102.0/28 \
--gateway=192.168.102.2 \
--opt com.docker.network.driver.mtu=1200 \
ingress
```

##### Ativando a disponibilidade do node novamente

`<NODE_NAME> É O NOME DO HOSTNAME QUE RETORNA NO COMANDO docker node ls`

```bash
docker node update --availability active <NODE_NAME>
```

##### Criando rede para o nosso ambiente

```bash
docker network create --driver overlay monitoring-network
```

##### Inspect de todas as redes do docker novamente

```bash
for net in `docker network ls |grep -v NAME | awk '{print $2}'`;do ipam=`docker network inspect $net --format {{.IPAM}}` && echo $net - $ipam; done
```

##### Adicionando outros nodes manager no cluster

##### Criando regra de firewall em todos os nodes

```bash
firewall-cmd --permanent --add-port=2377/tcp
firewall-cmd --permanent --add-port=7946/tcp
firewall-cmd --permanent --add-port=7946/udp
firewall-cmd --permanent --add-port=4789/udp
firewall-cmd --reload
```

##### Pegando Token para manager

```bash
docker swarm join-token manager
```

##### Deploy stack Zabbix

##### Instalar o GIT

```bash
dnf install -y git
```

##### Clonando depositório

```bash
cd ~/
git clone <URL DO SEU GIT>
```

`git@github.com:robertsilvatech/deploy-zabbix-docker-advanced.git`

##### Inicianlizando a stack 

```bash
cd <NOME_DA_PASTA_CLONADA>
sh create.sh
sh grafana.sh
docker stack deploy -c docker-compose.yaml maratonazabbix
```

##### Listando stacks disponiveis

```bash
docker stack ls
```

##### Listando status dos serviços

```bash
docker service ls
```

##### Verificar logs

```bash
docker service logs -f maratonazabbix_zabbix-server
docker service logs -f maratonazabbix_zabbix-frontend
docker service logs -f maratonazabbix_grafana
```
 
##### Removendo stack

```bash
docker stack rm maratonazabbix
```







