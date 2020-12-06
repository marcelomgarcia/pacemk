# Testing pacemaker/corosync

The idea is to test a pacemaker/corosync using Vagrant and Ansible. I will be using the guide from Cluster labs [quickstart](https://clusterlabs.org/quickstart-redhat.html)

## Vagrant file

The `Vagrantfile` defines three VMs: `flik`, `atta`, and `hopper`:

    config.vm.define "flik" do |machine|
    (...)
    config.vm.define "hopper" do |hopper|
    (...)
    config.vm.define "atta", primary: true do |atta|

The definition of `hopper` has an extra clausule that defines this machine as the *primary*. That is this will be *default* machine in a multi machine environment. Ansible will be installed on this machine, and it will controll the other two.

We also define the private address for each machine so they can communicate between them. The definition is:

    machine.vm.network "private_network", ip: "192.168.50.10"
    machine.vm.network "private_network", ip: "192.168.50.11"
    machine.vm.network "private_network", ip: "192.168.50.12"

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

