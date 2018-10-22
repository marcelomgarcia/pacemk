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

    export VAGRANT_DOTFILE_PATH=${HOME}/Work/pacemk/.vmulti
    vagrant up

And on the inventory file we point to the

    flik ansib(...)be_ssh_private_key_file=/vagrant/.vmulti/machines/...

### SSH keys

Setting the [ssh keys](https://www.rittmanmead.com/blog/2014/12/linux-cluster-sysadmin-ssh-keys/) for the _root_ user with Ansible module `copy`.

    mgarcia@mercury:~/Work/pacemk/files$ mkdir ssh_keys  
    mgarcia@mercury:~/Work/pacemk/files/ssh_keys$ KEYS_DIR=`pwd`                    
    mgarcia@mercury:~/Work/pacemk/files/ssh_keys$ ssh-keygen -f ${KEYS_DIR}/id_rsa -q -N ""     
    mgarcia@mercury:~/Work/pacemk/files/ssh_keys$ cp id_rsa.pub authorized_keys

Then on the Ansible playbook they are copied to the `.ssh` directory in the home of the _root_

    - name: Copy ssh config file to root's ssh dir.
        copy:
        src: files/ssh_keys/config
        dest: /root/.ssh/
        owner: root
        group: root
        mode: 0400

<<<<<<< HEAD
Sometimes the service don't start on the cluster. The problem seems to be with the `flik` node. 
=======
### Shell commands

The `pcs` commands are executed with the _shell_ module, and they always return as "changed" so maybe it's a good idea to register the variable and print the output.
>>>>>>> 41172c4eb4a2557808d7557faa141040bdbb6e6d
