Virtualizing a Network with Cumulus VX
======================================
Prerequisites
-------------
Before running this demo, install
[VirtualBox](https://www.virtualbox.org/manual/ch02.html),
[Vagrant](https://www.vagrantup.com/downloads.html), and
[Ansible](https://docs.ansible.com/ansible/intro_installation.html)

Provision the topology and log in
---------------------------------
    git clone https://github.com/cumulusnetworks/cldemo-vagrant
    cd cldemo-vagrant
    vagrant plugin install cumulus-vagrant
    vagrant up
    vagrant ssh oob-mgmt-server
    sudo su - cumulus
