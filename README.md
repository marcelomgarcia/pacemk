# Testing pacemaker/corosync

The idea is to test a pacemaker/corosync using Vagrant and Ansible.

## Vagrant file

The `Vagrantfile` defines two VM: `flik` and `atta`:

    config.vm.define "flik", primary: true  do |flik|
    (...)
    config.vm.define "atta" do |atta|

The definition of `flik` has an extra clausule that defines this machine as the *primary*. That is this will be *default* machine in a multi machine environment.    