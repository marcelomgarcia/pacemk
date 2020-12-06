# Testing pacemaker/corosync

The idea is to test a pacemaker/corosync using Vagrant and Ansible. I will be using the guide from Cluster labs [quickstart](https://clusterlabs.org/quickstart-redhat.html)

## Vagrant file

The `Vagrantfile` defines three VMs: `flik`, `atta`, and `hopper`:

    config.vm.define "flik" do |machine|
    (...)
    config.vm.define "hopper" do |hopper|
    (...)
    config.vm.define "atta", primary: true do |atta|
    (...)

The definition of `atta` has an extra clausule that defines this machine as the *primary*. That is this will be *default* machine in a multi machine environment. Ansible will be installed on this machine, and it will controll the other two.

### Memory and CPU

On the Vagrant file we specify the CPU and the memory of each node

```
    flik.vm.provider :virtualbox do |vb|
        vb.memory = 2048
        vb.cpus = 2
    end
```

And there is a similar definition for _hopper_ and _atta_. In this way each node can have the necessary number of CPUs and amount of memory. In this case _flik_ has 2GB of memory.

### Ansible

We also define Ansible as the provisioning mechanism for configuration. Since Windows can't be an Ansible controller, we will need to define one of the guest machines to be the controller. For this [local](https://www.vagrantup.com/docs/provisioning/ansible_local) installation, we will use the `ansible_local` provisioner.

On the Vagrant file we define

```
    atta.vm.provision :ansible_local do |ansible|
      ansible.verbose        = false
      ansible.install        = true
      ansible.limit          = "all"
      ansible.inventory_path = "inventory"
      ansible.playbook       = "pacemk_pbk.yaml"
    end
```

The verbosity is set to false otherwise the output becomes polluted with too many messages. The option to install isn't necessary because it's the default. From the controller we provision all (`limit`) all nodes. Finally there are the path to inventory file, and the playbook to use.

### Networking

We also define the private address for each machine so they can communicate between them. The definition is:

    flik.vm.network "private_network", ip: "192.168.50.11"
    hopper.vm.network "private_network", ip: "192.168.50.12"
    atta.vm.network "private_network", ip: "192.168.50.10"

We test by booting the machines and trying to ping the nodes from the controller:

```
PS C:\Users\mgarcia\Documents\Work\pacemk> vagrant ssh
Last login: Sun Dec  6 15:42:41 2020 from 10.0.2.2
[vagrant@atta ~]$ hostname
atta
[vagrant@atta ~]$
[vagrant@atta ~]$ ping -c 3 flik
PING flik (192.168.50.11) 56(84) bytes of data.
64 bytes from flik (192.168.50.11): icmp_seq=1 ttl=64 time=0.334 ms
(...)
[vagrant@atta ~]$
[vagrant@atta ~]$ ping -c 3 hopper
PING hopper (192.168.50.12) 56(84) bytes of data.
64 bytes from hopper (192.168.50.12): icmp_seq=1 ttl=64 time=0.336 ms
(...)
[vagrant@atta ~]$
```

To force the re-sync of the shared folder use `vagrant reload` or `vagrant up`. The first option, `reload` is in fact a reboot of the virtual machines.

## Ansible

### Inventory

The easiest way to access the Ansible inventory on the VMs is to simply [export](https://www.vagrantup.com/docs/synced-folders/basic_usage) the `vagrant` folder to the clients. In the `Vagrantfile` we defined the folder to exported (synched)

```
  config.vm.synced_folder ".", "/vagrant", type: "rsync",
```
here `.` is the folder project, and it will be synced with the folder `/vagrant` on the clients via `rsync` command.

The inventory file defines the clients and their IP address inside the Vagrant environment. We also define a group with the nodes that will compose the PCS cluster. The host `atta` has the option _local_ for the type of connection because it's the controller, and the playbook should run on the node itself.

```
atta      ansible_ssh_host=192.168.50.10   ansible_connection=local
flik      ansible_ssh_host=192.168.50.11
hopper    ansible_ssh_host=192.168.50.12

[nodes]
hopper
flik
```

To use Ansible on the VMs, we have to provide the inventory file to the `ansible` command

```
[vagrant@atta ~]$ ansible -i /vagrant/inventory -m ping nodes
hopper | SUCCESS => {
(...)
}
flik | SUCCESS => {
(...)
}
[vagrant@atta ~]$
```

### Disable SSH checking

One important step is to create a `ansible.cfg` to fully disable SSH host key checking

```
[defaults]
host_key_checking = no

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o IdentitiesOnly=yes -o StrictHostKeyChecking=no
```

This file is available to the clients on the `/vagrant` folder.

## SSH keys

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

Sometimes the service don't start on the cluster. The problem seems to be with the `flik` node.

## Windows 10 and Powershell

Windows 10 has its own `ssh` client, and it's necessary to disable it so Vagrant can use the buid-in client:

```
PS C:\Users\mgarcia\Documents\Work\pacemk> vagrant ssh
vagrant@127.0.0.1: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
PS C:\Users\mgarcia\Documents\Work\pacemk>
PS C:\Users\mgarcia\Documents\Work\pacemk\keys> $Env:VAGRANT_PREFER_SYSTEM_BIN += 0
PS C:\Users\mgarcia\Documents\Work\pacemk\keys> vagrant ssh
[vagrant@atta ~]$
```

## PCS configuration

### Shell commands

The `pcs` commands are executed with the _shell_ module, and they always return as "changed" so maybe it's a good idea to register the variable and print the output.

### Starting the cluster

The service `pacemaker` doesn't start automatically without a STONITH configuration. Since it's a simple test cluster, I will disable STONITH (fencing), and check the configuration of the cluster by printing the _cib_

```
[root@hopper ~]# pcs property set stonith-enabled=false
[root@hopper ~]# pcs cluster cib
(...)
        <nvpair id="cib-bootstrap-options-stonith-enabled" name="stonith-enabled" value="false"/>
```

Restart the cluster with new configuration

```
PS C:\Users\mgarcia\Documents\Work\pacemk> vagrant reload
```

But _pcs_ was still not working because the services `corosync` and `pacemaker` weren't enabled

```
[root@hopper ~]# pcs status
Error: cluster is not currently running on this node
[root@hopper ~]#
[root@hopper ~]#  systemctl status corosync
● corosync.service - Corosync Cluster Engine
   Loaded: loaded (/usr/lib/systemd/system/corosync.service; disabled; vendor preset: disabled)
(...)
[root@hopper ~]# systemctl enable corosync
[root@hopper ~]# systemctl enable pacemaker
```

After enabling the services, and restarting the cluster, `pcs` was working

```
[root@flik ~]# pcs status
Cluster name: mycluster
Stack: corosync
Current DC: flik (version 1.1.23-1.el7-9acf116022) - partition with quorum
(...)
```

