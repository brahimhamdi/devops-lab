Vagrant.require_version ">= 2.0.0"

boxes = [
  { 
    :name => "kube-control-plane", 
    :eth1 => "192.168.56.10", 
    :mem => "4096", 
    :cpu => "2" 
  },
  { 
    :name => "kube-node1", 
    :eth1 => "192.168.56.11", 
    :mem => "2048", 
    :cpu => "1" 
  },
  { 
    :name => "kube-node2", 
    :eth1 => "192.168.56.12", 
    :mem => "2048", 
    :cpu => "1" 
  }
]

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu64/noble64"
  config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")

# ---------- Machine Docker-Jenkins -------------------- #
  config.vm.define "docker-jenkins" do |docker|
    docker.vm.hostname = "docker-jenkins"
    docker.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus = 2
    end
    docker.vm.network "private_network", ip: "192.168.56.100"

    docker.vm.provision "shell", inline: <<-SHELL
      # Configuration de Vim
      echo 'colorscheme ron' >> ~/.vimrc
      echo 'set tabstop=2' >> ~/.vimrc
      echo 'set shiftwidth=2' >> ~/.vimrc
      echo 'set expandtab' >> ~/.vimrc

      # Mise à jour et installation des paquets de base
      sudo apt update
      sudo apt install -y apt-transport-https ca-certificates curl software-properties-common git gcc tree openjdk-21-jdk maven mariadb-client

      # Installation de Docker
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
      sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

      # Installation de Trivy
      wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
      echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list

      # Installation des paquets Docker et Trivy
      sudo apt update
      sudo apt install -y docker-ce trivy

      # Ajout de l'utilisateur vagrant au groupe docker
      sudo usermod -aG docker vagrant

      # Redémarrage du service Docker
      sudo systemctl enable docker
      sudo systemctl restart docker

      # Installation de Jenkins
      sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
      echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
      sudo apt update
      sudo apt install -y jenkins

      echo "Provisioning de docker-jenkins terminé !"
    SHELL
  end

# ---------- Cluster Kubernetes -------------------- #
  boxes.each do |opts|
    config.vm.define opts[:name] do |node|
      node.vm.hostname = opts[:name]

      node.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", opts[:mem]]
        v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
        v.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
        v.customize ["modifyvm", :id, "--uartmode1", "file", File::NULL]
      end

      node.vm.network :private_network, ip: opts[:eth1]

# ---------- Cluster Kubernetes -------------------- #
      node.vm.provision "shell", inline: <<-SHELL
        apt --allow-unauthenticated update
        apt --allow-unauthenticated install -y bash-completion binutils
        echo 'colorscheme ron' >> ~/.vimrc
        echo 'set tabstop=2' >> ~/.vimrc
        echo 'set shiftwidth=2' >> ~/.vimrc
        echo 'set expandtab' >> ~/.vimrc
        echo 'source <(kubectl completion bash)' >> ~/.bashrc
        echo 'alias k=kubectl' >> ~/.bashrc
        echo 'alias c=clear' >> ~/.bashrc
        echo 'complete -F __start_kubectl k' >> ~/.bashrc
        sed -i '1s/^/force_color_prompt=yes\\n/' ~/.bashrc

        # modules & fonctionnalities
        #################################################
        sudo modprobe overlay
        sudo modprobe br_netfilter
        echo 'overlay' | sudo tee /etc/modules-load.d/containerd.conf
        echo 'br_netfilter' | sudo tee -a /etc/modules-load.d/containerd.conf
        echo 'net.bridge.bridge-nf-call-iptables = 1' | sudo tee /etc/sysctl.d/kubernetes.conf
        echo 'net.bridge.bridge-nf-call-ip6tables = 1' | sudo tee -a /etc/sysctl.d/kubernetes.conf
        echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/kubernetes.conf
        sudo sysctl --system

        # Install & config containerd
        #################################################
        sudo apt update
        sudo apt install -y apt-transport-https ca-certificates curl software-properties-common tree
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
        sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
        sudo apt install -y containerd.io
        sudo containerd config default | sudo tee /etc/containerd/config.toml
        sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
        sudo systemctl restart containerd
        sudo crictl config runtime-endpoint unix:///run/containerd/containerd.sock

        # Install kubernetes
        ##############################################################
        sudo mkdir -p /etc/apt/keyrings
        echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        sudo apt update
        sudo apt install -y kubelet kubeadm

        # swap off
        sudo swapoff -a
        sudo sed -i '/swap/s/^/#/' /etc/fstab
        sudo rm -f /swap.img
      SHELL

 # Provision control-plane
      if opts[:name] == "kube-control-plane"
        node.vm.provision "shell", inline: <<-SHELL
          # Install Kubectl
          sudo apt install -y kubectl
          echo "source <(kubectl completion bash)" >> ~/.bashrc

          # Install Docker, openjdk, maven, ...
          sudo apt install -y git docker-ce openjdk-21-jdk maven mariadb-client
          sudo usermod -aG docker vagrant

          # Install Helm
          sudo snap install helm --classic
          echo "source <(helm completion bash)" >> ~/.bashrc

          # Install Kustomize
          wget "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"
          chmod +x install_kustomize.sh
          ./install_kustomize.sh
          sudo mv kustomize /usr/bin/
        SHELL
      end
    end
  end
end
