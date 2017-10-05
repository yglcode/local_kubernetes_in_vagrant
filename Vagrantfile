# -*- mode: ruby -*-
# vi: set ft=ruby :

master_num=1 #kubeadm only support 1 master, no HA
node_num=1 #can have many minion nodes

os_version="ubuntu/xenial64"

k8s_version="v1.8.0"
#k8s validated docker version 10/2/2017
docker_version="17.03.2~ce-0~ubuntu-xenial"
k8s_apiserver_port=6443
cluster_token="02ee7a.4f59f3429171668e"

#local docker registry
myhubip=ENV["MYHUBIP"]
myhubport=ENV["MYHUBPORT"]

def masterIP(num)
  return "192.168.66.#{10+num}"
end
master1IP=masterIP(1)
def nodeIP(num)
  return "192.168.66.#{20+num}"
end

#backup/cache deb pkgs and dok images for later reinstall
cacheDirVM="/vagrant/cache" #cache dir accessed inside VM
cacheDirHost="./cache" #cache dir accessed from host
deb_pkgs_dir="deb_pkgs"
docker_pkgs_dir="#{deb_pkgs_dir}/docker"
k8s_pkgs_dir="#{deb_pkgs_dir}/kubeadm_kubectl_cni"
dok_images_dir="dok_images"
master_images_dir="#{dok_images_dir}/master"
minion_images_dir="#{dok_images_dir}/minion"
flannel_dashboard_cfg="flannel_dashboard_yml"

num_master_images=12 #current master has 12 dok images
num_minion_images=3  #current minion has 3 dok images


$install_docker_from_internet = <<-SHELL01
        #--install & config docker
        apt-get update 
        apt-get -y install apt-transport-https ca-certificates curl software-properties-common
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
        #apt-key fingerprint 0EBFCD88
        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
        apt-get update && apt-get -y install docker-ce=#{docker_version}
        mkdir -p #{cacheDirVM}/#{docker_pkgs_dir}
        mv /var/cache/apt/archives/*deb #{cacheDirVM}/#{docker_pkgs_dir}/
SHELL01

$install_k8s_from_internet = <<-SHELL02
        #--install kubeadm,kubectl,k8s_cni
        curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
        echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
        apt-get update
        apt-get -y install kubelet kubeadm kubernetes-cni
        mkdir -p #{cacheDirVM}/#{k8s_pkgs_dir}
        mv /var/cache/apt/archives/*deb #{cacheDirVM}/#{k8s_pkgs_dir}/
SHELL02

Vagrant.configure("2") do |config|
  config.vm.box = os_version

  (1..master_num).each do |i|
    config.vm.define vm_name="master%d" % i do |master|
      master.vm.hostname = vm_name
      masterIP=masterIP(i)
      masterIPRegx=masterIP.gsub(/\./,"\.")
      
      master.vm.network "private_network", ip: masterIP
      
      master.vm.provision :shell, inline: "sed 's/127\.0\.0\.1.*#{vm_name}.*/#{masterIPRegx} #{vm_name}/' -i /etc/hosts"

      if not File.exist?("#{cacheDirHost}/#{docker_pkgs_dir}")
        master.vm.provision "shell", inline: $install_docker_from_internet
      else
        master.vm.provision "shell", inline: "dpkg -i #{cacheDirVM}/#{docker_pkgs_dir}/*.deb"
      end

      master.vm.provision "shell", inline: <<-SHELL10
        if [ -n #{myhubip} ]; then  #config local registry
           echo "{ \\"insecure-registries\\" : [\\"#{myhubip}:#{myhubport}\\"] }" >> /etc/docker/daemon.json
           service docker restart 
        fi
        usermod -aG docker ubuntu
      SHELL10
      
      if not File.exist?("#{cacheDirHost}/#{k8s_pkgs_dir}") then
        master.vm.provision "shell", inline: $install_k8s_from_internet
      else
        master.vm.provision "shell", inline: "dpkg -i #{cacheDirVM}/#{k8s_pkgs_dir}/*.deb"
      end

      master.vm.provision "shell", inline: <<-SHELL11
        # install k8s master docker images for next "k8s init"
        if [ -d #{cacheDirVM}/#{master_images_dir} ]; then
            for img in #{cacheDirVM}/#{master_images_dir}/*.tgz; do docker load < $img;done
        fi               
        # cluster init
        kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=#{masterIP} --token #{cluster_token} --kubernetes-version #{k8s_version} --skip-preflight-checks
        #
        cp /etc/kubernetes/admin.conf /home/ubuntu/
        chown ubuntu:ubuntu /home/ubuntu/admin.conf
      SHELL11
      
      if not File.exist?("#{cacheDirHost}/#{flannel_dashboard_cfg}") then
        master.vm.provision "shell", inline: <<-SHELL12
           mkdir -p #{cacheDirVM}/#{flannel_dashboard_cfg}
           cd #{cacheDirVM}/#{flannel_dashboard_cfg}
           curl -sSL -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
           curl -sSL -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
           curl -sSL -O https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
        SHELL12
      end
      
      master.vm.provision "shell", privileged: false, inline: <<-SHELL13
        #setup master env var
        cp /home/ubuntu/admin.conf /vagrant/cache/  #backup for minion kubectl
        echo "export KUBECONFIG=/home/ubuntu/admin.conf" >> /home/ubuntu/.bashrc
        if [ -n #{myhubip} ]; then  #config local registry
           echo "export MYHUB=#{myhubip}:#{myhubport}" >> /home/ubuntu/.bashrc
        fi
        echo "alias kc='kubectl'" >> /home/ubuntu/.bashrc        
        #cat /vagrant/dot_bashrc >> /home/ubuntu/.bashrc
        
        #setup flannel
        export KUBECONFIG=/home/ubuntu/admin.conf
        kubectl apply -f #{cacheDirVM}/#{flannel_dashboard_cfg}/kube-flannel-rbac.yml
        kubectl apply -f #{cacheDirVM}/#{flannel_dashboard_cfg}/kube-flannel.yml
        kubectl create -f #{cacheDirVM}/#{flannel_dashboard_cfg}/kubernetes-dashboard.yaml

        kubectl taint nodes --all node-role.kubernetes.io/master-

        # backup dok/flannel images for later reinstall
        saveDokImages() {
            for s1 in $(sudo docker images --format {{.Repository}}:{{.Tag}}); do s2=${s1//\//-};s3=${s2//\:/-};sudo docker save $s1 | gzip > #{cacheDirVM}/$1/$s3.tgz;done
        }
        if [ ! -d #{cacheDirVM}/#{master_images_dir} ]; then
            mkdir -p #{cacheDirVM}/#{master_images_dir}
            until [ $(sudo docker images -q | wc -l) -ge #{num_master_images} ]; do sleep 2; done
            saveDokImages #{master_images_dir}
        fi
      SHELL13

    end
  end


  (1..node_num).each do |i| 
    config.vm.define vm_name="node%d" % i do |node|
      node.vm.hostname = vm_name
      nodeIP=nodeIP(i)
      nodeIPRegx=nodeIP.gsub(/\./,"\.")
      node.vm.network "private_network", ip: nodeIP

      node.vm.provision :shell, inline: "sed 's/127\.0\.0\.1.*#{vm_name}.*/#{nodeIPRegx} #{vm_name}/' -i /etc/hosts"
      
      if not File.exist?("#{cacheDirHost}/#{docker_pkgs_dir}")
        node.vm.provision "shell", inline: $install_docker_from_internet
      else
        node.vm.provision "shell", inline: "dpkg -i #{cacheDirVM}/#{docker_pkgs_dir}/*.deb"
      end

      node.vm.provision "shell", inline: <<-SHELL20
        if [ -n #{myhubip} ]; then  #config local registry
           echo "{ \\"insecure-registries\\" : [\\"#{myhubip}:#{myhubport}\\"] }" >> /etc/docker/daemon.json
           service docker restart 
        fi
        usermod -aG docker ubuntu
      SHELL20
      
      if not File.exist?("#{cacheDirHost}/#{k8s_pkgs_dir}") then
        node.vm.provision "shell", inline: $install_k8s_from_internet
      else
        node.vm.provision "shell", inline: "dpkg -i #{cacheDirVM}/#{k8s_pkgs_dir}/*.deb"
      end

      node.vm.provision "shell", inline: <<-SHELL21
        # install k8s minion docker images for next "k8s join"
        if [ -d #{cacheDirVM}/#{minion_images_dir} ]; then
            for img in #{cacheDirVM}/#{minion_images_dir}/*.tgz; do docker load < $img;done
        fi               

        # cluster join
        kubeadm join --token #{cluster_token} #{master1IP}:#{k8s_apiserver_port} --skip-preflight-checks   

        # backup dok images fro later use
        saveDokImages() {
            for s1 in $(sudo docker images --format {{.Repository}}:{{.Tag}}); do s2=${s1//\//-};s3=${s2//\:/-};sudo docker save $s1 | gzip > #{cacheDirVM}/$1/$s3.tgz;done
        }
        if [ ! -d #{cacheDirVM}/#{minion_images_dir} ]; then
            mkdir -p #{cacheDirVM}/#{minion_images_dir}
            until [ $(sudo docker images -q | wc -l) -ge #{num_minion_images} ]; do sleep 2; done
            saveDokImages #{minion_images_dir}
        fi
      SHELL21
      
      node.vm.provision "shell", privileged: false, inline: <<-SHELL22
        cp /vagrant/cache/admin.conf /home/ubuntu
        echo "export KUBECONFIG=/home/ubuntu/admin.conf" >> /home/ubuntu/.bashrc
        if [ -n "#{myhubip}" ]; then  #config local registry
           echo "export MYHUB=#{myhubip}:#{myhubport}" >> /home/ubuntu/.bashrc
        fi
        echo "alias kc='kubectl'" >> /home/ubuntu/.bashrc        
        #cat /vagrant/dot_bashrc >> /home/ubuntu/.bashrc
      SHELL22
    end
  end

end
