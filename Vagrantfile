# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
    {
        :name => "k8s-master",
        :type => "master",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :k8s_version => "1.15.0-00",
        :k8s_release => "stable-1.15",
        :eth1 => "192.168.205.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-1",
        :type => "node",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :k8s_version => "1.15.0-00",
        :k8s_release => "stable-1.15",
        :eth1 => "192.168.205.11",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-2",
        :type => "node",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :k8s_version => "1.15.0-00",
        :k8s_release => "stable-1.15",
        :eth1 => "192.168.205.12",
        :mem => "2048",
        :cpu => "2"
    }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT

    # install docker v17.03
    # reason for not using docker provision is that it always installs latest version of the docker, but kubeadm requires 17.03 or older

    k8s_version=$1
    cd $HOME
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')

    # run docker commands as root user (sudo not required)
    usermod -aG docker vagrant

    # install kubeadm
    apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
    apt-get update
    apt-get install -y kubelet=${k8s_version} kubeadm=${k8s_version} kubectl=${k8s_version}
    apt-mark hold kubelet kubeadm kubectl

    # kubelet requires swap off
    swapoff -a

    # keep swap off after reboot
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
    # set node-ip
    cp /vagrant/config/kubelet /etc/default/kubelet
    if [ -f "/bin/systemd" ];then
      cp /vagrant/config/daemon.json /etc/docker/daemon.json
      sudo systemctl restart docker
    fi
    sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" /etc/default/kubelet
    sudo systemctl restart kubelet
    sudo --user=vagrant touch /home/vagrant/.Xauthority

    # required for setting up password less ssh between guest VMs
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart


SCRIPT

$configureMaster = <<-SCRIPT
    echo -e "\nThis is master:\n"
    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`

    k8s_release=$1
    cd $HOME
    if [ ! -f /etc/kubernetes/admin.conf ];then
      # install k8s master
      HOST_NAME=$(hostname -s)
      kubeadm init --kubernetes-version ${k8s_release} --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16

      #copying credentials 
      mkdir -p $HOME/.kube
      cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

      # install Calico pod network addon
      export KUBECONFIG=/etc/kubernetes/admin.conf
      kubectl apply -f https://raw.githubusercontent.com/elanadmin/kubernetes-cluster/master/calico/calico.yaml

      kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
      chmod +x /etc/kubeadm_join_cmd.sh
      kubectl taint nodes --all node-role.kubernetes.io/master-
    fi

    # Install Helm Package Manager
    if [ ! -f /usr/sbin/helm ];then
      wget https://get.helm.sh/helm-v2.9.0-linux-amd64.tar.gz
      wget https://git.io/get_helm.sh -O get_helm.sh
      tar -zxvf helm-v2.9.0-linux-amd64.tar.gz
      sudo cp -rp linux-amd64/helm /usr/sbin/
      echo export HELM_HOME="$HOME/.helm" >> $HOME/.bash_profile
      export HELM_HOME="$HOME/.helm"
      kubectl create serviceaccount --namespace kube-system tiller 2> /dev/null
      kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller 2>/dev/null
      helm init --stable-repo-url https://charts.helm.sh/stable --service-account tiller | tee -a ~/helmet.init
      kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' 2>/dev/null
    fi

SCRIPT

$configureNode = <<-SCRIPT
    PWD=$(pwd)
    cd $HOME
    if [ ! -f $PWD/kubeadm_join_cmd.sh ];then
      echo -e "\nThis is worker:\n"
      apt-get install -y sshpass
      sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
      sh ./kubeadm_join_cmd.sh
    fi
SCRIPT

Vagrant.configure("2") do |config|

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]
            config.vm.network :forwarded_port, guest: 22, host: 2222, id: "ssh", disabled: true
            config.vm.network :forwarded_port, guest: 22, host: 2200, auto_correct: true
            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
            	 v.customize ["modifyvm", :id, "--groups", "/ELAN Development"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            # we cannot use this because we can't install the docker version we want - https://github.com/hashicorp/vagrant/issues/4871
            # config.vm.provision "docker"

            k8s_version = opts[:k8s_version]
            k8s_release = opts[:k8s_release]

            config.vm.provision "shell", inline: $configureBox, :args => [k8s_version]

            if opts[:type] == "master"
              config.vm.provision "shell", inline: $configureMaster, :args => [k8s_release],
                  run: "always"
            else
                config.vm.provision "shell", inline: $configureNode,
                  run: "always"
            end

        end

    end

end 
