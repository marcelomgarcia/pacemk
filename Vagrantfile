# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  #Synced folders
  config.vm.synced_folder ".", "/vagrant", type: "rsync",
    rsync__exclude: ".git/"
    
  # Common configuration for both machines.
  config.vm.box = "centos/7"
  config.ssh.insert_key = false
  config.vm.provision "file", source: "keys/config",  destination: "~/.ssh/config"
  config.vm.provision "file", source: "keys/vagrant", destination: "~/.ssh/id_rsa"
  config.vm.provision "file", source: "keys/vagrant.pub", destination: "~/.ssh/id_rsa.pub"
  config.vm.provision "file", source: "keys/hosts.txt", destination: "/etc/hosts"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa.pub"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/config"

  # Define the first server.
  config.vm.define "flik" do |flik|
    flik.vm.hostname = "flik"
    flik.vm.provider :virtualbox do |vb|
        vb.memory = 2024
        vb.cpus = 2
    end
    flik.vm.network "private_network", ip: "192.168.50.11"
  end

  # Define the second server.
  config.vm.define "hopper" do |hopper|
    hopper.vm.hostname = "hopper"
    hopper.vm.provider :virtualbox do |vb|
        vb.memory = 2024
        vb.cpus = 2
    end
    hopper.vm.network "private_network", ip: "192.168.50.12"
  end

  config.vm.define "atta", primary: true do |atta|
    atta.vm.hostname = "atta"
    atta.vm.network "private_network", ip: "192.168.50.10"
    atta.vm.network "forwarded_port", guest: 9090, host: 9090
    atta.vm.provider :virtualbox do |vb|
        vb.memory = 2048
        vb.cpus = 2
    end
    atta.vm.provision :ansible_local do |ansible|
      ansible.verbose        = false
      ansible.install        = true
      ansible.limit          = "all"
      ansible.inventory_path = "inventory"
      ansible.playbook       = "pacemk_pbk.yaml"
    end
  end
end
