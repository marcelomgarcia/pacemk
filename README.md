# Testing pacemaker/corosync

The idea is to test a pacemaker/corosync using Vagrant and Ansible.

## Vagrant file

The `Vagrantfile` defines two VM: `flik` and `atta`:

    config.vm.define "flik", primary: true  do |flik|
    (...)
    config.vm.define "atta" do |atta|

The definition of `flik` has an extra clausule that defines this machine as the *primary*. That is this will be *default* machine in a multi machine environment.

We also define the private address for each machine so they can communicate between them. The definition is:

    flik.vm.network "private_network", ip: "192.168.50.10"
    atta.vm.network "private_network", ip: "192.168.50.11"

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