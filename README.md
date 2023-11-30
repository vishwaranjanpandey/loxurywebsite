FIRST>>>Setup Kubernetes Cluster version v1.23 & below with Docker Container Runtime ON UBUNTU

iStep1: On Master Node Only

## Install Docker

sudo apt-get update -y
sudo wget https://raw.githubusercontent.com/lerndevops/labs/master/scripts/installDocker.sh -P /tmp
sudo chmod 755 /tmp/installDocker.sh
sudo bash /tmp/installDocker.sh
sudo systemctl restart docker

## Install kubeadm,kubelet,kubectl

sudo wget https://raw.githubusercontent.com/lerndevops/labs/master/scripts/installK8S-v1-23.sh -P /tmp
sudo chmod 755 /tmp/installK8S-v1-23.sh
sudo bash /tmp/installK8S-v1-23.sh

Now check validate>>>>>>>>>>>>>>>>>>>>>
   1  docker -v
   2  kubeadm version -o short
   3  kubelet --version
   4  kubectl version --short --client

## Initialize kubernetes Master Node
 
   sudo kubeadm init --ignore-preflight-errors=all

   sudo mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config

   ## install networking driver -- Weave/flannel/canal/calico etc... 

   ## below installs calico networking driver 
    
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml 

   # Validate:  kubectl get nodes


Step2: On All Worker Nodes>>>>>>>>>>>>>>>

## Install Docker

sudo wget https://raw.githubusercontent.com/lerndevops/labs/master/scripts/installDocker.sh -P /tmp
sudo chmod 755 /tmp/installDocker.sh
sudo bash /tmp/installDocker.sh
sudo systemctl restart docker

## Install kubeadm,kubelet,kubectl

sudo wget https://raw.githubusercontent.com/lerndevops/labs/master/scripts/installK8S-v1-23.sh -P /tmp
sudo chmod 755 /tmp/installK8S-v1-23.sh
sudo bash /tmp/installK8S-v1-23.sh

   1  docker -v
   2  kubeadm version -o short
   3  kubelet --version
   4  kubectl version --short --client

## Run Below on Master Node to get join token 

sudo kubeadm token create --print-join-command 

    copy the kubeadm join token from master & run it as sudo on all nodes

    Ex: sudo kubeadm join 10.128.15.231:6443 --token mks3y2.v03tyyru0gy12mbt \
           --discovery-token-ca-cert-hash sha256:3de23d42c7002be0893339fbe558ee75e14399e11f22e3f0b3435



Now Write Deployment and Service file in Kubernetesmaster server>>>>>>>>>>>>>>>>
 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetesproject
spec:
  replicas: 10
  minReadySeconds: 45 # wait for 45 sec before pod is ready going to next
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  
      maxSurge: 2        
  selector:
    matchLabels:
      app: kubeserve
  template:
    metadata:
      name: kubeserve
      labels:
        app: kubeserve
    spec:
      containers:
      - image: vishwaranjanpandey/kubernetesproject
        name: app

---
kind: Service
apiVersion: v1
metadata:
   name: kubernetesproject-svc
spec:
  type: LoadBalancer
  selector: 
    app: kubeserve
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32110




NOW INSTALL DOCKER AND ANSIBLE ON SECOND UBUNTU SERVER 

######  INSTALL DOCKER>>>>>>>>>>>>>>>>>>>>>

    sudo apt-get update -y
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release software-properties-common
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update ; clear
    sudo apt-get install -y docker-ce

    sudo wget https://raw.githubusercontent.com/lerndevops/labs/master/kubernetes/0-install/daemon.json -P /etc/docker
    sudo systemctl restart docker.service
    sudo service docker status

After installation login docker hub account also >>>>>>>>>>>>

    docker login
    username: vishwaranjanpandey
    password: <*******>

######   INSTALL ANSIBLE>>>>>>>>>>>>>>>>>>




    sudo apt-get install ansible -y

    enable ssh communication between ansible and terraform master server    
    
    sudo vim /etc/ssh/sshd_config
    Now give permitRootLogin as (yes) and PasswordAuthentication as (yes) in ansible and kubernetes master server 
    useradd root
    passwd root
 Now generate ssh key and copy key into kubernetes master server with root@<pub/pri IP of k8s master>
    ssh-keygen
    ssh-copy-id root@<pubIP>


######  NOW WRITE ansible.yml file in ansible server 

vim ansible.yml
- hosts: all
  become: yes
  tasks:
    - name: create new deployment
      command: kubectl apply -f /home/vishwaranjan_p12/deployment.yml





######  NOW INSTALL JENKINS ON 3rd SERVER  >>>>>>>>>>>>>>    


    sudo apt-get install java -y 
    sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
        https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
  

Then add a Jenkins apt repository entry:

    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
        https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
        /etc/apt/sources.list.d/jenkins.list > /dev/null

Update your local package index, then finally install Jenkins:
  
    sudo apt-get update
    sudo apt-get install fontconfig openjdk-17-jre
    sudo apt-get install jenkins
    sudo systemctl start jenkins


Now enable ssh communication between jenkin server and ansible server with 

    sudo vim /etc/ssh/sshd_config
    Now give permitRootLogin as (yes) and PasswordAuthentication as (yes) in ansible and kubernetes master server
    useradd root
    passwd root
Now generate sshkey and copy rsa_pub into ansible  server with root@<pub/pri IP of k8s master>
    ssh-keygen
    ssh-copy-id root@<pubIP>    

 mm  
Download jenkin Plugin to enable ssh communication between jenkin and ansible server 

Now go to manage jenkin and Download "Publish Over SSH" plugin
now go to tools and configure ssh server and add jenkin and ansible server 
ssh server 
name: ansible
hostname: <ansible pub ip>
username: root
password: devops

add jenkin server also to copy file from jenkin server to ansible server 

ssh server
name: jenkin
hostname: <jenkin pub ip>
username: root
password: devops



Now create job in jenkin server integerate github url with jenkin


https://github.com/vishwaranjanpandey/kubernetesproject.git 

Now go to build step and click on Send files or execute commands over SSH

To copy dockerfile from jenkin to ansible server use rsync command on jenkin ssh server 
rsync -avh /var/lib/jenkins/workspace/kubernetesproject/Dockerfile root@<ansilbeIP>:/home/ec2-user

now Build the docker image and push Docker image in docker hub on ansible ssh server because docker alreay install in ansible server 

now execute below command on ssh ansible server

user Exec command
cd /home/ec2-user
docker build -t kubernetesproject . 
docker image tag kubernetesproject:latest vishwaranjanpandey/kubernetesproject
docker push vishwaranjanpandey/kubernetesproject


Now create Webhook on dockerfile repositery




