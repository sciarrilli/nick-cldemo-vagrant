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
    # spin up the vagrant environment
    git clone https://github.com/sciarrilli/nick-cldemo-vagrant
    cd nick-cldemo-vagrant
    vagrant plugin install vagrant-cumulus
    vagrant up
    vagrant ssh workbench
    cd nick-cldemo-ansible

    # ebgp playbook
    ansible all -m ping
    ansible all -a "ip route"
    ansible-playbook playbook-ebgp.yml
    ansible all -a "ip route"
    ansible leaf01 -a "cat /etc/network/interfaces"
    ansible-playbook playbook-cleanup.yml
    ansible all -a "ip route"

    # ospf playbook
    ansible-playbook playbook-ospf.yml
    ansible all -a "ip route"
