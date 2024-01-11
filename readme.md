## Criação de um cluster kubenetes com o kubeadm

## Cenário

```

- 3 VM com 2GB de RAM, 2 vCPU e 10GB de disco.
- Ubuntu 22.04 em todas as VM.

- hostnames: kubemaster01, kunenode01, kubenode02.

```
# Vamos começar...

## Desativar Swap

```
$ sudo swapoff -a
# Desativar também no fstab algum mount que lá esteja.
```

## Carregar módulos do kernel 

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

## Parametrizar o sysctl conforme requisitos do Kubernetes:

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Instalar os pacotes do Kubernetes via apt

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Instalar o containerd

```
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y containerd.io
```


## Configurar o systemd cgroup driver:

```
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl status containerd
```

## Ativar o serviço kubelet para iniciar automaticamente com o arranque do sistema:

```
sudo systemctl enable --now kubelet
```

## Portos necessários

No master node (ou control plane) é preciso o porto TCP 6443. Já nos worker nodes devem ficar acessíveis os portos 10250-10255. Para mais detalhes sobre portes necessários consultar os créditos mais abaixo.

No caso do cluster a criar for para testes e não produção, podem desativar as firewall dos hosts:

```
iptabels -F
systemctl stop ufw
```

## Inicializar o cluster (só no master node)

```
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=<IP da VM kubemaster01>
```

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```
# testar o cluster:
kubectl get nodes
```

## Nos worker nodes, copiar e colar informação devolvida depois da criação do cluster. Algo do género:

```
sudo kubeadm join <IP da VM kubemaster01>:6443 --token if9hn9.xhxo6s89byj9rsmd \
	--discovery-token-ca-cert-hash sha256:*************************************************************** 
```


## Gerar novo token

O token gerado durante a instalação expira. Então sempre que necessitarem podem gerar um novo token no master node:

```
kubeadm token create --print-join-command
```

## Créditos

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

https://github.com/linuxtips/MesDoKubernetes


