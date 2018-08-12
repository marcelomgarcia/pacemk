# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  # Common configuration for both machines.
  config.vm.box = "centos/7"
  config.vm.provider "virtualbox" do |vv|
    vv.memory = 2048
    vv.cpus = 2
  end

  # Define the first server. This will be the master of pcs cluster.
  config.vm.define "flik", primary: true  do |flik|
    flik.vm.hostname = "flik"
    #flik.vm.network "private_network", ip: "10.0.2.50"
    flik.vm.network "private_network", ip: "192.168.50.10"
  end

  # Define the second server. This will be the slave of pcs cluster.
  config.vm.define "atta" do |atta|
    atta.vm.hostname = "atta"
    #atta.vm.network "private_network", ip: "10.0.2.51"    
    atta.vm.network "private_network", ip: "192.168.50.11"
  end

end
 