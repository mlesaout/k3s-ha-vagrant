# -*- mode: ruby -*-
# vi: set ft=ruby :

controlplane_eip = "192.168.56.50"

server_ip = "192.168.56.10"

otherservers = { "server2" => "192.168.56.11", 
                 "server3" => "192.168.56.12" }

agents = { "agent1" => "192.168.56.21",
           "agent2" => "192.168.56.22" }

secondaryserver_ip = ""

# Extra parameters in INSTALL_K3S_EXEC variable because of
# K3s picking up the wrong interface when starting server and agent
# https://github.com/alexellis/k3sup/issues/306

server_script = <<-SHELL
    sudo -i
    apk add curl
    export INSTALL_K3S_EXEC="--cluster-init --tls-san=#{controlplane_eip} -node-taint CriticalAddonsOnly=true:NoExecute --no-deploy=traefik --no-deploy=servicelb --bind-address=0.0.0.0 --flannel-iface=eth1"
    curl -sfL https://get.k3s.io | sh -
    echo "Sleeping for 10 seconds to wait for k3s to start"
    sleep 10
    kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
    kubectl apply -f /vagrant_shared/kube-vip-ds.yaml
    kubectl rollout status daemonset/kube-vip-ds -n kube-system
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
    kubectl apply -f /vagrant_shared/metallb-cm.yaml
    cp /var/lib/rancher/k3s/server/token /vagrant_shared
    cp /etc/rancher/k3s/k3s.yaml /vagrant_shared
    sed -i s/0.0.0.0/#{controlplane_eip}/g /vagrant_shared/k3s.yaml
    SHELL

otherserver_script = <<-SHELL
    sudo -i
    apk add curl
    export K3S_TOKEN_FILE=/vagrant_shared/token
    export K3S_URL=https://#{server_ip}:6443
    export INSTALL_K3S_EXEC="server --tls-san=#{controlplane_eip} -node-taint CriticalAddonsOnly=true:NoExecute --no-deploy=traefik --no-deploy=servicelb --bind-address=0.0.0.0 --flannel-iface=eth1"
    curl -sfL https://get.k3s.io | sh -s -
    SHELL

agent_script = <<-SHELL
    sudo -i
    apk add curl
    export K3S_TOKEN_FILE=/vagrant_shared/token
    export K3S_URL=https://#{server_ip}:6443
    export INSTALL_K3S_EXEC="--flannel-iface=eth1"
    curl -sfL https://get.k3s.io | sh -
    SHELL

Vagrant.configure("2") do |config|
  config.vm.box = "generic/alpine314"
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  config.vm.define "server1", primary: true do |server|
    server.vm.network "private_network", ip: server_ip
    server.vm.synced_folder "./shared", "/vagrant_shared"
    server.vm.hostname = "server1"
    server.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = "2"
    end
    server.vm.provision "shell", inline: server_script
  end

  otherservers.each do |server_name, server_ip|
    config.vm.define server_name do |server|
      server.vm.network "private_network", ip: server_ip
      server.vm.synced_folder "./shared", "/vagrant_shared"
      server.vm.hostname = server_name
      server.vm.provider "virtualbox" do |vb|
        vb.memory = "1024"
        vb.cpus = "1"
      end
      server.vm.provision "shell", inline: otherserver_script
    end
  end

  agents.each do |agent_name, agent_ip|
    config.vm.define agent_name do |agent|
      agent.vm.network "private_network", ip: agent_ip
      agent.vm.synced_folder "./shared", "/vagrant_shared"
      agent.vm.hostname = agent_name
      agent.vm.provider "virtualbox" do |vb|
        vb.memory = "1024"
        vb.cpus = "1"
      end
      agent.vm.provision "shell", inline: agent_script
    end
  end
end