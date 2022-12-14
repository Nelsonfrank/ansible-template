# Ansible

## Contents

- Prerequisite
  - OpenSSH Overview and Setup
  - Setting up the Git repository

- Installation
- Running elevated ad-hoc Commands
- Introduction to Playbook
- The 'when' Conditional
- Improving your Playbook
- Targeting Specific Nodes
- Tags
## Prerequisite

### OpenSSH Overview and Setup

- Make sure OpenSSH is installed on the workstation and servers

- Connect to each server from the workstation, answer "yes" to initial connection prompt

```bash
ssh server_user@server-ip-address

# Ctrl + d - logout of server
```

- Create an SSH key pair (with a passphrase) for your normal user account

```bash
ssh-keygen -t ed25519 -C "new ssh key"

# -t = ssh key type
#i.e
# ed25519 (OpenSSH 6.5+) - most secure and short
# ECDSA (OpenSSH 5.7+)
# RSA
# DSA (No longer allowed by default in OpenSSH 7.0+)
#
# -c = comment, it can be your email or anything to
# distingush your key from other key
```

- Copy that key to each server

```bash
ssh-copy-id -i path/to/your/public/key.pub remote-user@ip-address-of-server

# -i = input file

# explaination
# ssh-copy-id copy your local public key and paste it on
# authorized_keys file in .ssh directory in your server
```

- Create and SSH key that is specific to Ansible

```bash
# generate dedicated ssh key for ansible
ssh-keygen -t ed25519 -C "ansible"
```

- Copy ansible key to each server

```bash
# copy ansible key to all server
ssh-copy-id -i path/to/your/public/ansible.pub user@ip-address
```

- Specify which key you want to use when ssh to a server

```bash
ssh -i ~/.ssh/ansible remote-user@server-ip

# -i = input file
```

- Cache the passphrase

```bash
# Start the ssh-agent in the background.
eval $(ssh-agent)

# Add your SSH private key to the ssh-agent.
ssh-add  ~/.ssh/private_key

# This will prevent you from add passphrase only in lifetime of that specific terminal instance. So if you close that termial you will have to run that commands again.

# Quick fix
# Create and add alias that will run that command for you to your .zshrc or .bashrc file

alias ssha='eval $(ssh-agent) && ssh-add'

# then you can use that alias next time you before you ssh to your server

ssha path/to/private/ssh-key
```

### Setting up the Git repository

- Add your default server private key to your github.
    - Create new ssh to github account under ```Account settings``` > ```SSH and GPG keys``` > ```New SSH key```

- This will help your server to pull and push changes from your github account.


## Installation and setup Ansible

### Install Ansible
```bash

# Ubuntu installation
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible

# Or Using pip install
# https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

# Other OS Platform
# Check it here:
# https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html
```

### Create inventory file 
- Create inventory file and add all your host IP addresses or domains


### Connect Ansible to hosts
- Run the following command to connect ansible with hosts 

```bash
ansible all --key-file /.ssh/ansible-private-key -i inventory -m ping

# all - all hosts
# --key-file - private ssh key with access to hosts
# -i - inventory
# -m - modules (all you to run module)
```

or Instead create ```ansible.cfg``` file inside your repo to take ansible arguments and attributes
```text
# ansible.cfg file
[defaults]
inventory = inventory
private_key_file = ~/.ssh/ansible-private-key
```
and this will make your above ansible command look like 

```bash
ansible all -m ping
```

### Usefull ansible command

```bash
# lists of all hosts
ansible all --list-hosts
```

```bash
# get hosts informations
ansible all -m gather_facts
``` 

```bash
# get informations for specific host
ansible all -m gather_facts --limit <IP-address>

<IP-address> - example 172.16.xxx.xx
``` 

## Running elevated ad-hoc Commands
Try to run apt moule to all server

```bash
ansible all -m apt -a update_cache=true

#-a - arguments 

# The above command is equivalent to 
apt update

# NOTE:
# - This will result to permission denied error 
```

```bash
# Solution
ansible all -m apt -a update_cache=true --became --ask-become-pass

# --become - elevate the privilege to sudo
# --ask-become-pass - this will ask for sudo password

# this command wil ask for sudo password 
# then run command to all hosts

# NOTE 
- Make sure all your host has same sudo password for this to work

# read more about apt module  in ansible docs:
# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html
```

### Install package to all hosts

```bash
# Install vim-nox package
ansible all -m apt -a name=vim-nox --become --ask-become-pass

# this command wil ask for sudo password 
# then install package to all hosts
```

### Update package for all hosts

```bash
# Update vim-nox package
ansible all -m apt -a "name=vim-nox state=latest" --become --ask-become-pass

#NOTE
# In order to add multiple arguments after -a use "" to add multiple agruments
```

### Update all packages with available update for all hosts

```bash
# Update all packages with updates from all hosts/servers
ansible all -m apt -a "upgrade=dist" --become --ask-become-pass
```

## Introduction to Playbook

Check [```install_apache.yml```](install_apache.yml) file

### Run playbook
```bash
ansible-playbook --ask-ecome-pass install_apache.yml
```

## The 'when' Conditional

This ansible playbook will work only if all hosts are debian based linux.

```yml
---

- hosts: all
  become: true
  tasks:
    
   - name: update repository index
     apt:
       update_cache: yes
        
   - name: install apache2 package
     apt:
        name: apache2
        state: latest
   
   - name: add php support for apache2
     apt:
        name: libapache2-mod-php
        state: latest
```

What if you add other distribution such as arch, fedora etc they don't have apt command.

Solution
Assume we have 3 ubuntu server and 1 CentOs 

```yml
---

- hosts: all
  become: true
  tasks:
    
   - name: update repository index for debian based distribution
     apt:
       update_cache: yes
     when: ansible_distribution == "Ubuntu"
        
   - name: install apache2 package for debian based distribution
     apt:
        name: apache2
        state: latest
     when: ansible_distribution == ["Ubuntu", "Debian"]
   
   - name: add php support for apache2 for debian based distribution
     apt:
        name: libapache2-mod-php
        state: latest
     when: ansible_distribution == "Ubuntu"

       
   - name: update repository index for CentOS
     dnf:
       update_cache: yes
     when: ansible_distribution == "CentOS"
        
   - name: install apache2 package for CentOS
     apt:
        name: httpd
        state: latest
     when: ansible_distribution == "CentOS"
   
   - name: add php support for apache2 for CentOS
     apt:
        name: php
        state: latest
     when: ansible_distribution == "CentOS"

```

> Read more about Condition [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#id2)


## Improving your Playbook

Refactor the above ansible playbook
```yml
---

- hosts: all
  become: true
  tasks:
   - name: install apache2 and php packages  for debian based distribution
     apt:
        name: 
          - apache2
          - libapache2-mod-php
        state: latest
        update_cache: yes
     when: ansible_distribution == ["Ubuntu", "Debian"]
        
   - name: install apache and php packages for CentOS
     dnf:
        name: 
          - httpd
          - php
        state: latest
        update_cache: yes
     when: ansible_distribution == "CentOS"
```

Further refactor to one play for all distribution
```yml
---

- hosts: all
  become: true
  tasks:
   - name: install apache and php packages
     package:
        name: 
          - "{{apache_package}}"
          - "{{php_package}}"
        state: latest
        update_cache: yes
```

Then, Update inventory file by adding variable name

```
172.16.250.123 apache_package=apache2 php_packag=libapache2-mod-php
172.16.250.124 apache_package=apache2 php_packag=libapache2-mod-php
172.16.250.125 apache_package=apache2 php_packag=libapache2-mod-php
172.16.250.128 apache_package=httpd php_packag=php
```

> This will work the same as the above playbook

## NB:

The package name is "package" and not apt or dnf, This is because package is Generic OS package manager.

Read more about ansible generic package manager [here](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html)


- php_package and apache_package are variables which are populated everytime ansible command run



## Targeting Specific Nodes
- Individualize our host by perform specific action on only specific hosts for example only one server is web server so we need to install apache in only web server host and not the rest.

- Group host into different categories

```
[web_servers]
172.16.250.123
172.16.250.124
172.16.250.127

[db_servers]
172.16.250.125
172.16.250.126

[file_servers]
172.16.250.128
```


create new playbook file named `site.yml`

```yml
---

- hosts: all
  become: true
  pre_tasks:
      
   - name: update repository index for CentOS
     dnf:
       update_only: yes
       update_cache: yes
     when: ansible_distribution == "CentOS"

   - name: update repository index for debian based distribution
     apt:
       update_only: yes
       update_cache: yes
     when: ansible_distribution == "Ubuntu"

- hosts: web_servers
  become: true
  tasks:

   - name: install apache2 and php packages  for debian based distribution
     apt:
        name: 
          - apache2
          - libapache2-mod-php
        state: latest
        update_cache: yes
     when: ansible_distribution == ["Ubuntu", "Debian"]
        
   - name: install apache and php packages for CentOS
     dnf:
        name: 
          - httpd
          - php
        state: latest
        update_cache: yes
     when: ansible_distribution == "CentOS"

- hosts: db_servers
  become: true
  tasks:
      .....

- hosts: file_servers
  become: true
  tasks:
       .....

```

> `pre-tasks` - run before other tasks


## Tags
- Target a specific playbook you want to test instead of run all


```yml
---

- hosts: all
  become: true
  pre_tasks:
      
   - name: update repository index for CentOS
     tags: always
     dnf:
       update_only: yes
       update_cache: yes
     when: ansible_distribution == "CentOS"

   - name: update repository index for debian based distribution
     tags: always
     apt:
       update_only: yes
       update_cache: yes
     when: ansible_distribution == "Ubuntu"

- hosts: web_servers
  become: true
  tasks:

   - name: install apache2 and php packages  for debian based distribution
     tags: apache,apache2,ubuntu
     apt:
        name: 
          - apache2
          - libapache2-mod-php
        state: latest
        update_cache: yes
     when: ansible_distribution == ["Ubuntu", "Debian"]
        
   - name: install apache and php packages for CentOS
     tags: apache,centos,httpd 
     dnf:
        name: 
          - httpd
          - php
        state: latest
        update_cache: yes
     when: ansible_distribution == "CentOS"

- hosts: db_servers
  become: true
  tasks:
      .....

- hosts: file_servers
  become: true
  tasks:
```

Review the list of tags from comand-line

```bash
ansible-playbook --list-tags site.yml
```

Target a specific tag in your playbook, for example centos

```bash
ansible-playbook --tags centos --ask-become-pass site.yml

# multiple tags
ansible-playbook --tags "centos, apache" --ask-become-pass site.yml
```

NB; `always` tag run everytime ansible run play even by specify other tags