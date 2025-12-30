# Title Page
<img width="100" alt="MyLogo" src="https://github.com/rafaelurrutiasilva/images/blob/main/logos/MyLogo_2.png" align=left><br>
<br>
Titel på dokumentet<br>
Författare<br>
Publiceringsdatum<br>

<br>

## Abstract
Kort sammanfattning av dokumentet

## Table of Contents

1. [Introduction](#introduction)
2. [Goals and Objectives](#goals-and-objectives)
3. [Method](#method)
4. [Target Audience](#target-audience)
5. [Document Status](#document-status)
6. [Disclaimer](#disclaimer)
7. [Scope and Limitations](#scope-and-limitations)
8. [Environment](#environment)
9. [Acknowledgments](#acknowledgments)
10. [References](#references)
11. [Conclusion](#conclusion)

## Introduction
This is the third project <a href="https://github.com/rafaelurrutiasilva/Proxmox_on_Nuc/blob/main/Extra/Mermaid/Projects.md">in a series of projects</a>, with the goal of setting up a complete virtualized, automated, and monitored IT-Enviroment as a part of our internship at [The Swedish Meteorological and Hydrological Institute (SMHI)](https://www.smhi.se/en/about-smhi). Previously, <a href=https://github.com/rafaelurrutiasilva/Proxmox_on_Nuc>Proxmox was installed and configured</a> on a server, and a <a href=https://github.com/Filipanderssondev/Rocky_Linux_OS_Base_for_VMs>Rocky Linux golden image</a> was created for cloning. At this stage in the project, we have 3 VMs, one for management of the environment, one for monitoring, and one for running applications. In this project, I will begin work on the management VM using Ansible, an automation IaC platform. 

## Goals and Objectives
This is part of a larger ongoing Infrastructure as Code (IaC) project that will use Proxmox as a base, with Rocky Linux as the OS running on each virtual machine.
The goal of this project is to build a complete IT-environment and gain a deeper understanding of the underlying components and their part in a larger production chain.

With Ansible, we can automate installation and deployment, orchestration and configuration. The goal of this project is to set up Ansible in a scalable and secure way. Ansible will handle the bulk of management in our environment. This should work as efficiently iregardless of the amount of hosts. 

## Method
### 3.1. Download Ansible

#### 3.1.1. **Prepare repos** <br>

Go into the Managament VM (mgmt-01). This is where Ansible will be installed.
 
On Rocky Linux, different Ansible-related packages can be found in both Appstream and Extra Packages for Enterprise Linux (EPEL) repos. 

Open epel.repo: `sudo vi /etc/yum.repos.d/epel.repo` <br>

Replace baseurl with `mirrors.nsc.liu.se/fedora-epel/$releasever${releasever_minor:+-z}/Everything/$basearch/`

If enabled=0, set to 1. Save and exit. 

#### 3.1.2 **Download Ansible and other packages** <br>

Run `sudo dnf update`

Use `dnf search ansible` to view the available Ansible packages. 

Install Ansible Core: `sudo dnf install ansible-core-1:2.16.14-1.el10.noarch`

Aside from Ansible Core, some of the collections may be of interest. The Community collection contains a large number of modules. The Posix collection and the Podman Containers collection may also prove useful further on. 

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

This is a suggested directory structure, placed in /opt. For a lab, it would suffice to create a single ansible folder placed in your home directory. 

#### 3.2.2. Set the correct permissions for the directories and files <br>

Change the group ownership to wheel: `sudo chown -R root:wheel /opt/ansible`

Give group write access: `sudo chmod -R g+rw /opt/ansible`
Make new files inherit the wheel group: sudo find /opt/ansible -type d -exec chmod g+s {} \;

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

Edit the /inventory/hosts.ini file: <br>

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
For the very first playbook, I'll begin with something simple, the Ansible equivelent of *sudo dnf update*.

#### 3.4.1 **Create a playbook** <br>

`vi /opt/ansible/playbooks/dnf_update.yml`

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

## Target Audience
Målgrupp

## Document Status
Dokumentstatus (om det finns relevant information om dokumentets status, till exempel utkast, slutfört, etc.). Tex:
> [!NOTE]  
> My work here is not finished yet. I need, among other things, to supplement with instructions on how each component should be configured to work together as well supplement with an overview image that explains how the whole thing works.


## Disclaimer
Ansvarsfriskrivning. Tex:
> [!CAUTION]
> This is intended for learning, testing, and experimentation. The emphasis is not on security or creating an operational environment suitable for production.

## Scope and Limitations
Omfattning och begränsningar

## Environment
Miljö som användes

## Acknowledgments
Tack och erkännanden. Tex:
Big thanks to all the people involved in the material I refer to in my links! I would also like to express gratitude to everyone out there, including my colleagues and friends, who are creating things that help and inspire us to continue learning and exploring this never-ending world of computer technology.

## References
Referenser (om det behövs)

## Conclusion
Slutsats
