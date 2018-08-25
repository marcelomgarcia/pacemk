# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  #Synced folders
  config.vm.synced_folder ".", "/vagrant", type: "rsync",
    rsync__exclude: ".git/"
   
  # Common configuration for both machines.
  config.vm.box = "centos/7"
  config.vm.provider "virtualbox" do |vv|
    vv.memory = 2048
    vv.cpus = 2
  end

  # Define the first server. This will be the master of pcs cluster.
  config.vm.define "flik" do |machine|
    machine.vm.hostname = "flik"
    machine.vm.network "private_network", ip: "192.168.50.10"
  end

  # Define the second server. This will be the slave of pcs cluster.
  config.vm.define "atta" do |machine|
    machine.vm.hostname = "atta"
    machine.vm.network "private_network", ip: "192.168.50.11"
  end

  # Create a "controller" where we will install Ansible and from where we'll 
  # install the software of the two other machines.
  config.vm.define 'controller', primary: true  do |machine|
    machine.vm.network "private_network", ip: "192.168.50.13"
    machine.vm.provision :ansible_local do |ansible|
      ansible.playbook = "pacemk_pbk.yaml"
      ansible.verbose = "false"
      ansible.install = "true"
      ansible.limit = "all"
      ansible.inventory_path = "inventory"
    end
  end
end
 