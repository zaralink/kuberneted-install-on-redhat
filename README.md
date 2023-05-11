# kuberneted-install-on-redhat



sudo yum update -y
sudo su -
sudo swapoff -a
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
#configure network by updating the /etc/hosts with the ip addresses and hostname of all the nodes

#10.128.15.228 master-node-k8          // For the Master node
#10.128.15.230 worker-node-1-k8       //  For the Worker node1
#10.128.15.233 worker-node-2-k8       //  For the Worker node2

sudo sed -i -e '$ a\10.128.15.228 master-node-k8' -e '$ a\10.128.15.230 worker-node-1-k8' -e '$ a\10.128.15.233 worker-node-1-k8' /etc/hosts
#Next, install the traffic control utility package:
sudo dnf install -y iproute-tc
#Allow firewall rules for k8s
#On the master node, allow the following ports
# sudo firewall-cmd --permanent --add-port=6443/tcp
# sudo firewall-cmd --permanent --add-port=2379-2380/tcp
# sudo firewall-cmd --permanent --add-port=10250/tcp
# sudo firewall-cmd --permanent --add-port=10251/tcp
# sudo firewall-cmd --permanent --add-port=10252/tcp
# sudo firewall-cmd --reload

sudo yum install firewalld -y
sudo dnf install nftables -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
#on master node uncomment
<<master
sudo firewall-cmd --permanent --add-port=6443/tcp && sudo firewall-cmd --permanent --add-port=2379-2380/tcp && sudo firewall-cmd --permanent --add-port=10250/tcp && sudo firewall-cmd --permanent --add-port=10251/tcp && sudo firewall-cmd --permanent --add-port=10252/tcp && sudo firewall-cmd --reload
master

#on the worker nodes
sudo firewall-cmd --permanent --add-port=10250/tcp && sudo firewall-cmd --permanent --add-port=30000-32767/tcp && sudo firewall-cmd --reload && sudo firewall-cmd --reload

#Kubernetes requires a container runtime for pods to run.
#we will install CRI-O which is a high-level container runtime.
#First, create a modules configuration file for Kubernetes.
       #sudo vi /etc/modules-load.d/k8s.conf
#Add these lines and save the changes
       #overlay
       #br_netfilter
sudo sh -c 'echo "overlay" >> /etc/modules-load.d/k8s.conf && echo "br_netfilter" >> /etc/modules-load.d/k8s.conf'



#Then load both modules using the modprobe command.

     #sudo modprobe overlay
     #sudo modprobe br_netfilter
sudo modprobe overlay && sudo modprobe br_netfilter

#Next, configure the required sysctl parameters as follows
      #sudo vi /etc/sysctl.d/k8s.conf
#Add the following lines:

          #net.bridge.bridge-nf-call-iptables  = 1
          #net.ipv4.ip_forward                 = 1
          #net.bridge.bridge-nf-call-ip6tables = 1

sudo sh -c 'echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.d/k8s.conf && echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/k8s.conf && echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/k8s.conf'

#To confirm the changes have been applied, run the command:

           #sudo sysctl --system

#To install CRI-O, set the $VERSION environment variable to match your CRI-O version

export VERSION=1.26

#Next, run the following commands:
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/CentOS_8/devel:kubic:libcontainers:stable.repo

#followed by:

sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/CentOS_8/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo

#Then use the DNF package manager to install CRI-O:

sudo dnf install cri-o

#Next, enable CRI-O on boot time and start it:

sudo systemctl enable cri-o && sudo systemctl start cri-o


#Install Kubernetes Packages
         #sudo vi /etc/yum.repos.d/kubernetes.repo

#And add the following lines and save
<<co
[kubernetes] 
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
co

sudo bash -c 'echo "[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl" > /etc/yum.repos.d/kubernetes.repo'

#run below commands
sudo dnf install -y kubelet-1.26.1 kubeadm-1.26.1 kubectl-1.26.1 --disableexcludes=kubernetes

sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

#Validate the kubectl binary against the checksum file:

echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

sudo yum update -y

sudo systemctl enable kubelet && sudo systemctl start kubelet


sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

             
#to add the worker node to the cluster, run the command generated by master
sudo kubeadm join 172.31.26.187:6443 --token a54gn9.dbctpn1592t6rcxx \
	--discovery-token-ca-cert-hash sha256:45a0fb962a9e641ac90e92d5dd09555624d5099092fb9868ff8b9abd80eca503



#only master node get below configs
===================================================================================================
 
#on master node as a regular user run the commands below sequentially

mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

#At the end you will be given the command to run on worker nodes to join the cluster

#Install Calico Pod Network Add-on used to provide container networking and security.
#container network interface CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

#be sure to remove taints from the master node:

kubectl taint nodes --all node-role.kubernetes.io/master-

#To confirm if the pods have started, run the command:
kubectl get pods -n kube-system






