# ClusterKubernetes
Projeto SIN142 - Sistemas distribuídos Desafio 2, Cluster Kubernetes

# Gleidson Vinícius Gomes Barbosa - 6331
# Cluster Kubernetes - Como fazer?

#OBS.: O PDF do repositório descreve exatamente os mesmos passos porém com imagens para mais detalhes.

O procedimento será executado por mim em duas máquinas virtuais utilizando VMWare, então o primeiro passo será a criação de ambas as máquinas virtuais e ativar a virtualização das mesmas visto que será utilizado o KVM. Será instalado em ambas as máquinas o Ubuntu Server 20.04. Os procedimentos detalhados nesse relatório exceto pelos relacionados ao VMWare podem ser encontrados em :		

•	Installing kubeadm | Kubernetes
•	Install Docker Engine on Ubuntu | Docker Documentation
•	Ports and Protocols | Kubernetes
•	Como abrir portas no Ubuntu Server? (vocepergunta.com) 
•	kubernetes - Kubelet service is not running. It seems like the kubelet isn't running or healthy - Stack Overflow
•	Creating a cluster with kubeadm | Kubernetes

1.	Criando a VM Master, ela será a máquina master desse cluster.
 
2.	Criando a VM Worker que será a máquina cliente desse processo, ela será o único worker desse processo.  



3.	Ativando a virtualização no VMWare, ao clicar em Customize Hardware, selecione Processors e ative a virtualização.
 
4.	Instalação do SO, nesse caso irei utilizar o Ubuntu server 18.04
 

# A instalação do Docker e Kubernetes

Neste momento estaremos instalando o Docker em ambas as máquinas, por isso colocarei prints de apenas uma delas, basta repetir o procedimento em ambas.

1.	Como estamos trabalhando com máquinas virtuais, estarei acessando primeiramente via SSH a máquina Master para trabalhar sem problemas de desempenho do VMWare.  
2.	Logo após, entrarei em modo root para não ter problemas com permissões.  Para isso utiliza-se o comando  sudo -i.
 
3.	Atualizaremos os pacotes do linux com apt update  
4.	O primeiro passo que faremos é desligar a memória swap com o comando swapoff -a  
5.	Depois instalaremos o Docker com o comando curl -fsSL https://get.docker.com | bash
após completar a instalação conferimos a versão com docker --version
   
6.	Nesse ponto, utilizaremos uma série de comandos para carregar os modulos necessários para o kubernetes e configurar as bridges, seguem os comandos:
modprobe br_netfilter
lsmod | grep br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system 
   
7.	Agora verificaremos se as portas requeridas pelo kubernetes estão abertas. Para abrir usaremos os seguintes comandos:
iptables -I INPUT -p tcp --dport 6443 -j ACCEPT 
iptables -I INPUT -p tcp --dport 2379 -j ACCEPT
iptables -I INPUT -p tcp --dport 2389 -j ACCEPT
iptables -I INPUT -p tcp --dport 10250 -j ACCEPT
iptables -I INPUT -p tcp --dport 10259 -j ACCEPT
iptables -I INPUT -p tcp --dport 10257 -j ACCEPT 

8.	Agora atualizaremos os pacotes do Linux com o comando apt-get update  
9.	Feito isso, alguns pré requisitos do kubernetes, como os certificados com o comando
apt-get install -y apt-transport-https ca-certificates curl  
10.	Adicionaremos uma chave publica do google cloud com o comando:
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

logo após adicionaremos o repositório kubernetes com o comando:
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list  
11.	Após adicionar o repositório, atualizaremos novamente os pacotes e por fim instalaremos o kubbernetes com a seguinte sequência de comandos:
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl      
12.	Executaremos o comando kubeadm – init para iniciar nosso mastere ao fim deste comando será gerado um token, lembre-se de salvá-lo, ele será necessário no futuro. No meu caso é o seguinte:

kubeadm join 192.168.254.147:6443 --token vmurg9.m20zvlq7pd7myruf \        --discovery-token-ca-cert-hash sha256:cd930035a44414aba217c3081632ad37035caaead039032dffd41b60bde26e0d  
13.	Agora iremos configurar o kubernetes de acordo com a própria resposta anterior utilizando os comandos indicados: 
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config  
14.	Verificamos com o comando kubectl get nodes que já temos nosso node Master no estado de NotReady.  
15.	Agora iremos adicionar nosso woker no nosso cluster. Precisaremos do comando com o token no passo 12, basta copiá-lo e executar na máquina worker.  
16.	Agora verificaremos no nosso master com o comando kubectl get nodes que temos o master e o worker no nosso cluster, ambos em estado NotReady. 
17.	Agora executaremos o comando que irá deixar nosso status como pronto:
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"  
18.	Executamos novamente o comando kubectl get nodes e observamos que nosso cluster está prontinho.  


# Erro no Kubelet

Fique atento a esse tópico pois temos alguns passos que só serão executados no nosso master, esses passos estão marcados com a tag [APENAS NA MÁQUINA MASTER], se por acaso for executado no worker, pode comprometer todo nosso cluster.

1.	Existe a possibilidade de alguns erros serem apresentados devido a problemas com o kubeletes, o seguinte erro é apresentado:  
2.	Nesse caso seguiremos os seguintes procedimentos: 
Primeiro desabilitaremos novamente o swap e após isso verificaremos o grupo do docker com a seguinte sequência de comandos:
sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo swapoff -a
docker info |grep -i cgroup
 

3.	Após isso editaremos o arquivo com o comando nano /etc/docker/daemon.json e adicionaremos o seguinte texto: 
{
    "exec-opts": ["native.cgroupdriver=systemd"]
}
  
 

4.	Agora executaremos a seguinte sequência de comandos para reiniciar alguns serviços e verificar o kubelet:
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl restart kubelet
 

5.	[APENAS NA MÁQUINA MASTER] Depois iremos refazer a fase de tokens do kubeadm com o comando 
kubeadm init phase bootstrap-token
 

6.	Agora desabilitaremos o firewall com o comando ufw disable 

7.	[APENAS NA MÁQUINA MASTER] Finalmente executaremos o comando  kubeadm init –ignore-preflight-errors=all  



Corrigimos nossos erros, agora voltamos ao passo que paramos e seguimos para concluir nosso cluster, Felicidade meus amigos!
                                         
