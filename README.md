# Rancher HA - Alta Disponibilidade

Repositorio usado para mostrar instalação do Rancher em HA.

https://rancher.com/docs/rancher/v2.x/en/installation/how-ha-works/

## Requisitos

1 DNS
1 máquina para load balancer
3 máquinas para o rancher-server

Utilizado na demonstração: UBUNTU 21.04

## Docker instalado em todas as máquinas

```sh
$ sudo apt update
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
$ sudo apt update
$ apt-cache policy docker-ce
$ sudo apt install docker-ce
$ sudo systemctl status docker
$ sudo usermod -aG docker ubuntu
```

## Portas

https://rancher.com/docs/rancher/v2.x/en/installation/requirements/ports/

## RKE

https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/create-nodes-lb/

Por que três nós?

Em um cluster RKE, os dados do servidor Rancher são armazenados no etcd. Este banco de dados etcd é executado em todos os três nós.

O banco de dados etcd requer um número ímpar de nós para que possa sempre eleger um líder com a maioria do cluster etcd. 

Se o banco de dados etcd não puder eleger um líder, o etcd pode sofrer a perda do lider, exigindo que o cluster seja restaurado do backup. 

Se um dos três nós etcd falhar, os dois nós restantes podem eleger um líder porque eles têm a maioria do número total de nós etcd. 

## INICIO

Logar na máquina do ELB - onde tudo será realizado
Instalar o kubectl nela também
Instalar o RKE nela também.

```
$ ssh ubuntu@172.16.0.7  35.175.118.212   # - NGINX - ELB

$ ssh ubuntu@172.16.0.27 18.206.46.208    # - RANCHER-SERVER-1
$ ssh ubuntu@172.16.0.28 3.237.96.159     # - RANCHER-SERVER-2
$ ssh ubuntu@172.16.0.29 34.234.225.242   # - RANCHER-SERVER-3
```

# Instalar Kubectl

```sh
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x ./kubectl
$ mv ./kubectl /usr/local/bin/kubectl
$ kubectl version --client
```

# Instalar RKE

```sh
$ curl -LO https://github.com/rancher/rke/releases/download/v1.2.7/rke_linux-amd64
$ mv rke_linux-amd64 rke
$ chmod +x rke
$ mv ./rke /usr/local/bin/rke
$ rke --version
$ exit
```

# COPIAR rancher-cluster.yml para Servidor

# USAR user ubuntu
# Copiar a para os outros hosts.

```sh
$ ssh-keygen
$ vi ~/.ssh/id_rsa
$ chmod 600 /home/ubuntu/.ssh/id_rsa
$ ssh-copy-id username@remote_host
```

# Rodar RKE

```sh
$ rke up --config ./rancher-cluster.yml
```

# Após o cluster subir:...

```sh
$ export KUBECONFIG=$(pwd)/kube_config_rancher-cluster.yml
$ kubectl get nodes
$ kubectl get pods --all-namespaces
```

# SALVAR OS ARQUIVOS

# Instalar HELM

```sh 
$ curl -LO https://get.helm.sh/helm-v3.3.1-linux-amd64.tar.gz
$ tar -zxvf helm-v3.3.1-linux-amd64.tar.gz
$ sudo mv linux-amd64/helm /usr/local/bin/helm
```

# Instalar o Rancher - Preparar

```sh 
$ helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
$ kubectl create namespace cattle-system
```

# Certificate Manager

```sh 
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.crds.yaml
$ kubectl create namespace cert-manager
$ helm repo add jetstack https://charts.jetstack.io

$ helm repo update
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.0

$ kubectl get pods --namespace cert-manager
```

# Instalar Rancher

```sh 
$ helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.tcemt.tc.br
```

# Verificar deployment

```sh 
$ kubectl -n cattle-system rollout status deploy/rancher
$ kubectl -n cattle-system get deploy rancher
```

# RODAR O NGINX

```sh 
$ sudo vi /etc/nginx.conf
$ docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /etc/nginx.conf:/etc/nginx/nginx.conf \
  nginx:1.14
```

# Kubernetes-HA - Alta Disponibilidade

Repositorio usado para mostrar instalação do Rancher em HA.

https://rancher.com/docs/rancher/v2.x/en/troubleshooting/kubernetes-components/etcd/

https://rancher.com/learning-paths/building-a-highly-available-kubernetes-cluster/


## Requisitos

Cluster Kubernetes HA de Produção

3 instâncias para ETCD         - Podendo perder 1
2 instâncias para CONTROLPLANE - Podendo perder 1
4 instâncias para WORKER       - Podendo perder todas

Usando na demonstração: UBUNTU 21.04

## Docker instalado em todas as máquinas

```sh
$ sudo apt update
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
$ sudo apt update
$ apt-cache policy docker-ce
$ sudo apt install docker-ce
$ sudo systemctl status docker
$ sudo usermod -aG docker ubuntu
```

## INICIO

Abrir o Rancher e criar um novo cluster.

Adicionar novo cluster com Existing Nodes

```sh
#ETCD
# Processador= 4 / Memória= 10

$ ssh ubuntu@172.16.0.16   # - etcd01 
$ ssh ubuntu@172.16.0.17   # - etcd02
$ ssh ubuntu@172.16.0.18   # - etcd03

#CONTROLPLANE
# Processador= 8 / Memória= 8

$ ssh ubuntu@172.16.0.16   # - controlplane01 
$ ssh ubuntu@172.16.0.16   # - controlplane02

#WORKER
# Processador/Memória = Onde irá rodar seus Pods

$ ssh ubuntu@172.16.0.16  # - worker01
$ ssh ubuntu@172.16.0.16  # - worker02
$ ssh ubuntu@172.16.0.16  # - worker03
$ ssh ubuntu@172.16.0.16  # - worker04

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name etcd-1 --etcd

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name etcd-2 --etcd

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name etcd-3 --etcd

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name controlplane-1 --controlplane

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name controlplane-2 --controlplane

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name worker-1 --worker

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name worker-2 --worker


```



