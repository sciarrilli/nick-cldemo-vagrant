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
    git clone https://github.com/sciarrilli/nick-cldemo-vagrant
    cd cldemo-vagrant
    vagrant plugin install cumulus-vagrant
    vagrant up oob-mgmt-server oob-mgmt-switch spine01 spine02 spine03 spine04 leaf01 leaf02 leaf03 leaf04
    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    cd nick-cldemo-ansible
    ansible-playbook playbook-ebgp.yml #for ebgp unnumbered
    ansible all -a "ip route"

    ansible-playbook playbook-cleanup.yml #erase all quagga and /etc/network/interfaces

    ansible-playbook playbook-ospf.yml #for ospf unnumbered
    ansible all -a "ip route"
