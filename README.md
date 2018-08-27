# Testing pacemaker/corosync

The idea is to test a pacemaker/corosync using Vagrant and Ansible. I will be using the guide from Cluster labs [quickstart](https://clusterlabs.org/quickstart-redhat.html)

## Vagrant file

The `Vagrantfile` defines three VMs: `flik`, `atta`, and `hopper`:

    config.vm.define "flik" do |machine|
    (...)
    config.vm.define "atta" do |machine|
    (...)
    config.vm.define 'hopper', primary: true  do |machine|

The definition of `hopper` has an extra clausule that defines this machine as the *primary*. That is this will be *default* machine in a multi machine environment. Ansible will be installed on this machine, and it will controll the other two.

We also define the private address for each machine so they can communicate between them. The definition is:

    machine.vm.network "private_network", ip: "192.168.50.10"
    machine.vm.network "private_network", ip: "192.168.50.11"
    machine.vm.network "private_network", ip: "192.168.50.13"

We test by booting the machines and trying to ping each other:

    PS C:\Users\mgarcia\Documents\Work\pacemk> vagrant up
    (...)
    PS C:\Users\mgarcia\Documents\Work\pacemk> vagrant ssh flik
    [vagrant@flik ~]$ ping 192.168.50.11
    PING 192.168.50.11 (192.168.50.11) 56(84) bytes of data.
    64 bytes from 192.168.50.11: icmp_seq=1 ttl=64 time=0.885 ms
    (...)
    PS C:\Users\mgarcia\Documents\Work\pacemk> vagrant ssh atta
    (...)
    [vagrant@atta ~]$ ping 192.168.50.10
    PING 192.168.50.10 (192.168.50.10) 56(84) bytes of data.
    64 bytes from 192.168.50.10: icmp_seq=1 ttl=64 time=0.461 ms
    (...)

To force the re-sync of the shared folder use `vagrant reload` or `vagrant up`. The first option, `reload` is in fact a reboot of the virtual machines.

## Ansible Playbook

When dealing with a multi-machine, the configuration is from the outside to the inside, or *outside-in* as define in the documentation. This means, the current playbook definition is global, that is apply to both machines.

    config.vm.define "flik", primary: true  do |flik|
    (...)
    config.vm.define "atta" do |atta|
    (...)
    config.vm.provision "ansible_local" do |ansible|
    (...)
