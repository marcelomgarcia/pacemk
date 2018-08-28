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

## Ansible

### Inventory

We define a inventory file with the path to ssh key and host group definition. There is a complication that the default place for machine definition, the `.vagrant` directory, is not exported by default. Therefore is necessary to use other location. In the case of this project, the new folder is exported (as environment variable) before starting vagrant:

    export VAGRANT_DOTFILE_PATH=/home/mgarcia/Work/pacemk/.vmulti
    vagrant up

And on the inventory file we point to the

    flik ansib(...)be_ssh_private_key_file=/vagrant/.vmulti/machines/...

