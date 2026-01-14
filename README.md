# Ansible on management VM

![](https://github.com/JonatanHogild/Ansible_on_management_vm/blob/main/Extra/ahbxqg.jpg) <br>
<img width="400" alt="" src="[https://github.com/JonatanHogild/Ansible_on_management_vm/blob/main/Extra/ahbxqg.jpg](https://github.com/JonatanHogild/Ansible_on_management_vm/blob/main/Extra/ahbxqg.jpg)" allign=left><br>



<br>**Authors:** _<a href="https://github.com/Filipanderssondev">Filip Andersson</a> and <a href="https://github.com/JonatanHogild">Jonatan Högild</a>_<br>
30-12-2025<br>

<br>

## Abstract
Setting up Ansible on a management VM to to manage other vms in our virtual IT-environment. 

## Table of Contents

1. [Introduction](#introduction)
2. [Goals and Objectives](#goals-and-objectives)
3. [Method](#method) <br>
   3.1 [Download Ansible](#31-download-ansible) <br>
   3.2 [Ansible directory structure](#32-ansible-directory-structure) <br>
   3.3 [Inventory](#33-inventory) <br>
   3.4 [Playbooks](#34-playbooks) <br>
   3.5 [ansible.cfg](#35-ansiblecfg) <br>
   3.6 [SSH Keys](#36-ssh-keys) <br>
   3.7 [Restricting SSH communication](#37-restricting-ssh-communication) <br>
4. [Target Audience](#target-audience)
5. [Document Status](#document-status)
6. [Disclaimer](#disclaimer)
7. [Scope and Limitations](#scope-and-limitations)
8. [Environment](#environment)
9. [Acknowledgments](#acknowledgments)
10. [References](#references)
11. [Conclusion](#conclusion)

## Introduction
This is the third project <a href="https://github.com/rafaelurrutiasilva/Proxmox_on_Nuc/blob/main/Extra/Mermaid/Projects.md">in a series of projects</a>, with the goal of setting up a complete virtualized, automated, and monitored IT-Enviroment as a part of our internship at [The Swedish Meteorological and Hydrological Institute (SMHI)](https://www.smhi.se/en/about-smhi). Previously, <a href=https://github.com/rafaelurrutiasilva/Proxmox_on_Nuc>Proxmox was installed and configured</a> on a server, and a <a href=https://github.com/Filipanderssondev/Rocky_Linux_OS_Base_for_VMs>Rocky Linux golden image</a> was created for cloning. At this stage in the project, we have 3 VMs, one for management of the environment, one for monitoring, and one for running applications. In this project, I will begin work on the management VM by setting up Ansible, an automation IaC platform. 

## Goals and Objectives
This is part of a larger ongoing Infrastructure as Code (IaC) project that will use Proxmox as a base, with Rocky Linux as the OS running on each virtual machine.
The goal of this project is to build a complete IT-environment and gain a deeper understanding of the underlying components and their part in a larger production chain.

With Ansible, we can automate just about anything. The goal of this project is to set up Ansible in a scalable and secure way. Ansible will handle the bulk of management in our environment and it should work as efficiently iregardless of the amount of hosts. 

## Method
### 3.1. Download Ansible
<!--
#### 3.1.1. **Prepare repos** <br>

Go into the Managament VM (mgmt-01). This is where Ansible will be installed.
 
On Rocky Linux, different Ansible-related packages can be found in both Appstream and Extra Packages for Enterprise Linux (EPEL) repos. 

Open epel.repo: `sudo vi /etc/yum.repos.d/epel.repo` <br>

Replace baseurl with `mirror.nsc.liu.se/fedora-epel/$releasever${releasever_minor:+-z}/Everything/$basearch/`

If enabled=0, set to 1. Save and exit. 

#### 3.1.2 **Download Ansible and other packages** <br> 

Use `dnf search ansible` to view the available Ansible packages. -->

Run `sudo dnf update`

Install Ansible Core: `sudo dnf install ansible-core`

Verify installation with: `ansible --version`

Also download *sshpass*, which allows ssh password authoirzation via Ansible. 

### 3.2 Ansible directory structure

#### 3.2.1. Create ansible folders and files <br>

/opt/ansible/ <br>
├── ansible.cfg  <br>
├── inventory/  <br>
│   ├── hosts.ini <br>
│   └── group_vars/  <br>
│   └── host_vars/  <br>
├── playbooks/ <br>
└── roles/  <br>
    └── common/  <br>

This is a suggested directory structure, placed in */opt*. For a lab, it would suffice to create a single ansible folder placed in your home directory. 

#### 3.2.2. Set correct permissions for the directories and files <br>

Change the group ownership to wheel: `sudo chown -R root:wheel /opt/ansible`

Give group write access: `sudo chmod -R g+rw /opt/ansible`
Make new files inherit the wheel group: `sudo find /opt/ansible -type d -exec chmod g+s {} \;`

Confirm: `ls -l /opt`

*drwxrwsr-x. root wheel ansible*

`ls -l /opt/ansible`

*-rw-rw-r--. root wheel ansible.cfg <br>
drwxrwsr-x. root wheel inventory <br>
drwxrwsr-x. root wheel playbooks <br>
drwxrwsr-x. root wheel roles* <br>

Add user to wheel (if this isn't done already): `sudo usermod -aG wheel username`

Confirm: `cat /etc/group | grep username`

### 3.3 Inventory

The inventory contains all the hosts that Ansible will manage. For now, I will keep it simple and add static hosts. 

#### 3.3.1 **hosts.ini** <br>

A hosts-file can either use the .ini or .yml format, use whichever you prefer. I used the .ini format.

Edit the */inventory/hosts.ini* file: <br>

```ini
[rocky]
localhost ansible_connection=local
metrics-01 ansible_host=10.208.12.102
app-01 ansible_host=10.208.12.103

[management]
localhost ansible_connection=local

[metrics]
metrics-01 ansible_host=10.208.12.102

[application]
app-01 ansible_host=10.208.12.103
```

Localhost is included, with its connection type specified as local. Changes made on other VMs can then also be applied locally on this system. 

Hosts have also been grouped into different categories, as shown. 

#### 3.3.2 Verify inventory and hosts

`ansible-inventory --graph` can be used to show the current inventory. 

Use `ansible all -m ping` to confirm connectivity.

### 3.4 Playbooks
Playbooks are structured instruction files written in YAML that tell Ansible which hosts to target and what tasks to perform on them.
For the very first playbook, I'll begin with something simple, the Ansible equivelent of *sudo dnf update*.

#### 3.4.1 **Create a playbook** <br>

Create a new file in the *playbooks* directory and edit it: `vi /opt/ansible/playbooks/dnf_update.yml`

```yaml
---
- name: Update Rocky Linux systems
  hosts: rocky
  become: true
  gather_facts: true

  tasks:
    - name: Update packages to latest version
      ansible.builtin.dnf:
        name: "*"
        state: latest
        update_cache: true

    - name: Remove unused packages
      ansible.builtin.dnf:
        autoremove: true
```

The dnf module is found in the Ansible builtin namespace, and the documentation is available <a href=https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/dnf_module.html>here.</a>

#### 3.4.2. **Run playbook** <br>

Make sure the other VMs are turned on.

Run the playbook with the following command: `ansible-playbook ./playbooks/update_dnf.yml --ask-pass --ask-become-pass -i ./inventory/hosts.ini`

### 3.5 ansible.cfg

The current ansible-playbook command is quite long. It can be shortened considerably by defining settings in the ansible.cfg file.

#### 3.5.1 **Reading the right configuration file** <br>

First, make sure the right config-file is used with `ansible --version`

I want to change *config file = /etc/ansible/ansible.cfg* to *config file = /opt/ansible/ansible.cfg*.

This can be done by changing the environment variable: `export ANSIBLE_CONFIG=/opt/ansible/ansible.cfg`

Make it permanent by adding the command to a new profile script: `sudo vi /etc/profile.d/ansible.sh`

#### 3.5.2 **Write an ansible.cfg file** <br>

The ansible configuration file can be generated using the *ansible-config* command, or created manually. I prefer to do this manually.

Here's what a configuration may look like:
<pre>
[defaults]
inventory = /opt/ansible/inventory/hosts.ini #inventory/hosts path
gather_facts = True #collect information about remote systems
gathering = smart #caches gathered information
interpreter_python = auto_silent #find the python interpreter on the host without issuing any warnings
stdout_callback = yaml #output format set to yaml

[privilege_escalation]
become = True #privilege escalation by default
become_method = sudo
</pre>

The configuration settings placed here should be globally applicable. Circumstantial settings are better placed in playbooks, or included at execution. 

### 3.6 SSH Keys

Ansible uses SSH to communicate, and SSH-keys will make this communication easier and safer.

#### 3.6.1 **Generate Keys** <br>

To generate new SSH keys, use: `ssh-keygen`

Then copy the public key to the other VMs: `scp /home/username/.ssh/id_ed25519.pub username@hostname:/home/username/.ssh/`

#### 3.6.2 **Change SSHD settings** <br>

Log into each VM and make the following changes in */etc/ssh/sshd_config*: <br>
Set *PubkeyAuthentication* to yes. <br>
Set *PasswordAuthentication* to no. <br>

Restart sshd after making changes: `sudo systemctl restart sshd` <br>

Verify connections with: `ansible all -m ping`

### 3.7 Restricting SSH communication

The *mgmt-01* VM should be able to access the other VMs via SSH. This fits the role of the management VM, it should be able to manage other hosts remotely. It is also necessary for Ansible to function. However, we want to restrict SSH communication for the other VMs. For example, *app-01* shouldn't be able to SSH into *mgmt-01*. 

#### 3.7.1 Change firewall rules

The Proxmox firewall allows SSH communication between all VMs. This is what we will change.

In the Proxmox web UI, go into Datacenter > Firewall > IPSet

Create a new IPSet, call it something like *mgmt* (the name *management* is reserved by a <a href=https://pve.proxmox.com/pve-docs/chapter-pve-firewall.html#_standard_ip_set_span_class_monospaced_management_span>standard IP set</a>, and should not be used here). 

Add the IP-address of *mgmt-01*
You may also want to add the address of your physical machine(s). 

Next, go to Datacenter > Firewall > Security Group > *allow-ssh* <br>
Here are the two SSH rules created in the previous project. <br>
Edit each of these, and set *mgmt* as source. <br>


## Target Audience
This repo is for anyone who wants a step-by-step guide on preparing Ansible for management of multiple hosts.
This repo is also part of a larger project aimed at people interested in learning about IaC, and building such an environment from scratch.

## Document Status
> [!NOTE]  
> This is a work in progress.<br>
> This repo is part of a larger ongoing project.
<br>

## Disclaimer
> [!CAUTION]
> This is intended for learning, testing, and experimentation. The emphasis is not on security or creating an operational environment suitable for production.
<br>

## Scope and Limitations
- ### 7.1. Scope
   * Instructions for installing and configuring Ansible for an enterprise environment. 
   * Instructions for creating Ansible inventories and playbooks.

- ### 7.2. Limitations
   * This guide is not intended for production-grade, multi-node clusters or advanced HA setups.
   * Network configuration is for now limited to a single-node setup and may not apply to complex environments.
   * Instructions may become outdated as software updates; always verify with the official documentation.
<br>

## Environment
- ### 8.1. Hardware
   - Asus PN64 ax210NGW

- ### 8.2. Software
   - Proxmox VE (9.1.1)
   - Rocky Linux (10.1)
   - Ansible (core 2.16.14)
<br>

## Acknowledgments
I would like to thank <a href=https://github.com/rafaelurrutiasilva>Rafael Urrutia</a> for his continuous support and guidance. 

## References
- [SMHI](https://www.smhi.se/en/about-smhi)
- [Project 1: Installing Proxmox on an Asus PN64](https://github.com/rafaelurrutiasilva/Proxmox_on_Nuc)
- [Project 2: Rocky Linux Golden Image](https://github.com/Filipanderssondev/Rocky_Linux_OS_Base_for_VMs)
- [Ansible builtin dnf module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/dnf_module.html)
- [Proxmox firewall standard ip sets](https://pve.proxmox.com/pve-docs/chapter-pve-firewall.html#_standard_ip_set_span_class_monospaced_management_span)

## Conclusion
Previously, I've been using scripting as a means of automation, with languages such as Bash, Python and Perl. Ansible has given me a new perspective on automation, while also being fairly easy to learn. Though this project only scratches the surface of Ansible and its capabilities, it shows how automation can be applied in complex enterprise environments, and the benefits of infrastructure as code. 

