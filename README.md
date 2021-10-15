# Rancher v2.5 Kubernetes - Alta Disponibilidade

Repositório utilizado para mostrar a instalação do Rancher em HA.

https://rancher.com/docs/rancher/v2.x/en/installation/how-ha-works/

![rancher-kubernetes-producao](https://user-images.githubusercontent.com/52961166/116142266-b4eb4980-a6a7-11eb-88d3-07d2e801d887.png)

## Requisitos

1 DNS
1 máquina para load balancer
3 máquinas para o rancher-server

Utilizado na demonstração: UBUNTU 21.04

## Docker instalado em todas as máquinas

```sh
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
sudo su
curl https://releases.rancher.com/install-docker/20.10.sh | sh
usermod -aG docker ubuntu
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
ssh ubuntu@172.16.0.11    # - NGINX - ELB

ssh ubuntu@172.16.0.13   # - RANCHER-SERVER-1
ssh ubuntu@172.16.0.14   # - RANCHER-SERVER-2
ssh ubuntu@172.16.0.15   # - RANCHER-SERVER-3
```

# Instalar Kubectl

```sh
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

# Instalar RKE (Rancher Kubernetes Engine)

O RKE é uma distribuição Kubernetes com certificação CNCF que resolve complexidades de instalação comuns do Kubernetes, removendo a maioria das dependências de host, apresentando um caminho estável para implantação, atualizações e reversões.

```sh
curl -LO https://github.com/rancher/rke/releases/download/v1.2.9/rke_linux-amd64
mv rke_linux-amd64 rke
chmod +x rke
mv ./rke /usr/local/bin/rke
rke --version
```

COPIAR rancher-cluster.yml para Servidor

Utilizar o user "ubuntu"

```sh
ssh-keygen
vi ~/.ssh/id_rsa
chmod 600 /home/ubuntu/.ssh/id_rsa
```
Copiar a chave gerada para os outros hosts.

```sh
ssh-copy-id username@remote_host
```

# Rodar RKE

```sh
rke up --config ./rancher-cluster.yml
```

# Após o cluster subir:...

```sh
export KUBECONFIG=$(pwd)/kube_config_rancher-cluster.yml
kubectl get nodes
kubectl get pods --all-namespaces
```

# Salvar os Arquivos

# Instalar HELM

O Helm é uma ferramenta de empacotamento de software livre que ajuda você a instalar e gerenciar o ciclo de vida de aplicativos kubernetes. ... Assim como os gerenciadores de pacotes do Linux, como apt e yum, o Helm é usado para gerenciar os gráficos do kubernetes, que são pacotes de recursos kubernetes pré-configurados

```sh 
curl -LO https://get.helm.sh/helm-v3.3.1-linux-amd64.tar.gz
tar -zxvf helm-v3.3.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

# Instalar o Rancher - Preparar

```sh 
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
kubectl create namespace cattle-system
```

# Certificate Manager

```sh 
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io

helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.0

kubectl get pods --namespace cert-manager
```

# Instalar Rancher

```sh 
helm install rancher rancher-stable/rancher \
--namespace cattle-system \
--set hostname=rancher.xxxx.xx.br
```

# Verificar deployment

```sh 
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get deploy rancher
```

# Rodar o Nginx

```sh 
sudo vi /etc/nginx.conf
docker run -d --restart=unless-stopped \
-p 80:80 -p 443:443 \
-v /etc/nginx.conf:/etc/nginx/nginx.conf \
nginx:1.14
```

# Kubernetes-HA - Alta Disponibilidade

Repositorio usado para mostrar instalação do Rancher em HA.

https://rancher.com/docs/rancher/v2.x/en/troubleshooting/kubernetes-components/etcd/

https://rancher.com/learning-paths/building-a-highly-available-kubernetes-cluster/

![k8s-distribuicao](https://user-images.githubusercontent.com/52961166/116142334-c7fe1980-a6a7-11eb-9c5d-ada6611b2363.png)

## Requisitos

Cluster Kubernetes HA de Produção

3 instâncias para ETCD         - Podendo perder 1
2 instâncias para CONTROLPLANE - Podendo perder 1
4 instâncias para WORKER       - Podendo perder todas

Usando na demonstração: UBUNTU 21.04

## Docker instalado em todas as máquinas

```sh
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
sudo su
curl https://releases.rancher.com/install-docker/20.10.sh | sh
sudo usermod -aG docker ubuntu
```

## INICIO

Abrir o Rancher e criar um novo cluster.

Adicionar novo cluster com Existing Nodes

```sh
#ETCD
# Processador= 4 / Memória= 10

ssh ubuntu@172.16.0.14   # - etcd01 
ssh ubuntu@172.16.0.15   # - etcd02
ssh ubuntu@172.16.0.16   # - etcd03

#CONTROLPLANE
# Processador= 8 / Memória= 8

ssh ubuntu@172.16.0.19   # - controlplane01 
ssh ubuntu@172.16.0.20   # - controlplane02

#WORKER
# Processador/Memória = Onde irá rodar seus Pods

ssh ubuntu@172.16.0.27  # - worker01
ssh ubuntu@172.16.0.28  # - worker02
ssh ubuntu@172.16.0.29  # - worker03
ssh ubuntu@172.16.0.30  # - worker04
```

# Exemplos

```sh
# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name etcd-1 --etcd

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name etcd-2 --etcd

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name etcd-3 --etcd

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name controlplane-1 --controlplane

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name controlplane-2 --controlplane

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name worker-1 --worker

# docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.5.0 --server https://3.227.241.169 --token zw9dgzb99n7fkg7l7lsb4wn6p49gmhcfjdp9chpzllzgpnjg9gv967 --ca-checksum 7c481267daae071cd8ad8a9dd0f4c5261038889eccbd1a8e7b0aa1434053731b --node-name worker-2 --worker
```

## Agradecimentos:

Este material é baseado no curso:

DevOps Ninja: Docker, Kubernetes e Rancher

https://www.udemy.com/course/devops-mao-na-massa-docker-kubernetes-rancher

https://github.com/jonathanbaraldi

Digo de passagem um ótimo curso, recomendo que se tiver oportunidade faça, pois a parte dos extras é SHOW de bola.

# License

Copyright (c) 2014-2018 [Rancher Labs, Inc.](http://rancher.com)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
