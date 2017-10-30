# Intro

[Ansible](https://www.ansible.com/) is a great tool for automation of configuration and orchestration. It can be used for everything from simply install a piece of software to completely setting up an environment to the granular level. We will be using Ansible to create a web server using Linux, Nginx, MariaDB, and PHP (LEMP) stack.

* We will use [CentOS](https://www.centos.org/) as out Linux distro because it is the closest to RHEL we can get on our local machine. We will run using Vagrant to make our lives easier. You are free to use whichever distro you link, I just prefer RHEL for my servers.

* We will setup Nginx instead of Apache because Nginx is much better with performance.

* We will be using [MariaDB](https://mariadb.org/) in place of MySQL because it is the drop in replacement provided by the distro. The DB will be initialized using an SQL script and we will limit a user's access to certain DBs.

* We will install the latest [PHP](https://secure.php.net/) from external repositories and install phpMyAdmin as well.

## Getting Started

First, a basic explanation of the terminology used by Ansible to refer to different files.

1. A **playbook** is the main file Ansible runs during execution

1. An **inventory** is a file that informs Ansible about the servers it needs to run on and method it will use to run

1. A **role** is a folder containing **tasks**, **defaults**, **vars**, **handlers**, **templates**, and **meta**. Everything, except **templates**, uses YAML files within

    * **tasks** contains files for the tasks that this **role** will run

    * **defaults** contains the default values for any low priority variables the role defines

    * **vars** contains higher priority variables that **role** defines (Variable priorities are out of scope for this post)

    * **handlers** are like tasks that can be reused in other scenarios. For example, restarting a web server can be a handler to be reused whenever config or code changes are uploaded

    * **templates** contains files that use the [Jinja2](http://jinja.pocoo.org/docs/2.9/#) template engine. Ansible can parse and swap variables within these **templates**

    * **meta** simply contains any meta information about the **role** like author, description, dependencies, etc

## Directory Structure

For the directory structure we will keep it simple and easy to understand. There will be a single playbook file and a single Vagrantfile at the root. The directory names are self explanatory, they will contain our inventory, roles, and variable files respectively. All subsequent commands will be executed in this project folder.

```shell
.
|-- inventory
|   |-- host_vars
|   |   |-- 192.168.33.10.yml
|   `-- inventory.ini
|-- roles
|-- vars
|   |-- local.yml
|-- Vagrantfile
|-- playbook.local.yml
```

For Ansible recommended directory structure please refer to the [Ansible best practices](http://docs.ansible.com/ansible/latest/playbooks_best_practices.html) page.

## The L in LEMP

For our Linux server we will be using Centos 7 and its official Vagrant image. [Vagrant](https://www.vagrantup.com/) is great for managing virtual machines and quickly provisioning local environments. Vagrant needs to be installed to use the Vangrantfile. Here is what our Vagrantfile looks like:

```Ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
    # The most common configuration options are documented and commented below.
    # For a complete reference, please see the online documentation at
    # https://docs.vagrantup.com.

    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://vagrantcloud.com/search.
    config.vm.box = "centos/7"

    # Disable automatic box update checking. If you disable this, then
    # boxes will only be checked for updates when the user runs
    # `vagrant box outdated`. This is not recommended.
    # config.vm.box_check_update = false

    # Create a forwarded port mapping which allows access to a specific port
    # within the machine from a port on the host machine. In the example below,
    # accessing "localhost:8080" will access port 80 on the guest machine.
    # NOTE: This will enable public access to the opened port
    # config.vm.network "forwarded_port", guest: 80, host: 8080

    # Create a forwarded port mapping which allows access to a specific port
    # within the machine from a port on the host machine and only allow access
    # via 127.0.0.1 to disable public access
    # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

    # Create a private network, which allows host-only access to the machine
    # using a specific IP.
    config.vm.network "private_network", ip: "192.168.33.10"

    # Create a public network, which generally matched to bridged network.
    # Bridged networks make the machine appear as another physical device on
    # your network.
    # config.vm.network "public_network"

    # Share an additional folder to the guest VM. The first argument is
    # the path on the host to the actual folder. The second argument is
    # the path on the guest to mount the folder. And the optional third
    # argument is a set of non-required options.
    # config.vm.synced_folder "../data", "/vagrant_data"

    # Provider-specific configuration so you can fine-tune various
    # backing providers for Vagrant. These expose provider-specific options.
    # Example for VirtualBox:
    #
    # config.vm.provider "virtualbox" do |vb|
    #   # Display the VirtualBox GUI when booting the machine
    #   vb.gui = true
    #
    #   # Customize the amount of memory on the VM:
    #   vb.memory = "1024"
    # end
    #
    # View the documentation for the provider you are using for more
    # information on available options.

    # Enable provisioning with a shell script. Additional provisioners such as
    # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
    # documentation for more information about their specific syntax and use.
    # config.vm.provision "shell" do |s|
    #   s.inline = "sudo apt update"
    #   s.inline = "sudo apt upgrade -y"
    #   s.inline = "sudo apt install python -y"
    # end

    config.vm.provision "ansible" do |ansible|
        ansible.inventory_path = "./inventory/inventory.ini"
        ansible.limit = "testservers"
        ansible.playbook = "playbook.local.yml"
    end
end
```

Most of the Vagrantfile is boilerplate that is generated by Vagrant when the file is initialized. This file can be generate with the following command:

```shell
vagrant init centos/7
```

The generated file will need to be modified too look like the above. Namely, the **private_network** line will need to uncommented, and the **config.vm.provision** section will need to be added. We need an IP assigned to the machine to access it from the our host using the Ansible CLI. The Ansible provisioning section will be executed when we first run `vagrant up`, or subsequently after the VM is started and we run `vagrant provision`.

The `ansible.inventory_path` line lets the Ansible CLI know which inventory file to use. The next line, `ansible.limit` tells Ansible which servers to use from within the inventory file (more on this later). Lastly, the `ansible.playbook` line defines which playbook file should be executed on provision.

Let's modify the inventory file to include the necessary config:

```text
[testservers]
192.168.33.10
```

Now let's add necessary connection info into the servers **host_vars** file:

```python
---
ansible_connection: ssh
ansible_user: vagrant
ansible_ssh_private_key_file: ../../.vagrant/machines/default/virtualbox/private_key
```

The private key path should point to our vagrant machines private key. If we keep the folder structure shown above, the private key path above should work.

## E is for Nginx

Now we get into using **ansible-galaxy**. [Ansible galaxy](https://galaxy.ansible.com/) is a marketplace of sorts for already written, configurable roles. We will interact with it by using the **ansible-galaxy** CLI utility. It comes packaged with our Ansible installation.

We will be making use of the **EPEL** repository and the **Remi** repository for PHP. So we need to install those onto our vagrant box prior to any other package. Use the following command to install *geerlingguy.repo-epel* and *geerlingguy.repo-remi* using **ansible-galaxy**:

```shell
ansible-galaxy install geerlingguy.repo-epel geerlingguy.repo-remi
```

We will use the following command to install the [*geerlingguy.nginx*](https://github.com/geerlingguy/ansible-role-nginx) role:

```shell
ansible-galaxy install geerlingguy.nginx
```

Now let's modify our **playbook.local.yml** file to make use of these role.

```python
---
- hosts: testservers
  become: true
  become_method: sudo
  vars_files:
    - vars/local.yml
  roles:
    - geerlingguy.repo-epel
    - geerlingguy.repo-remi
    - geerlingguy.nginx
```

Here we are specifying which host group to use from the inventory file. We also specify to use `become: true` and `become_method: sudo` so we can execute all commands as root. Then we point to a vars file (*dev.yml*) that will contain our variables for the various roles we will be using. Let's modify that file to add a few variables for configuring Nginx:

```python
nginx_yum_repo_enabled: true
nginx_vhosts:
  - listen: "80"
    server_name: "app.local"
    root: "{{ web_root }}"
    index: "index.php index.html index.htm"
    state: "present"
    filename: "app.local.conf"
    extra_parameters: |
        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index app.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            try_files $uri =404;
        }
        location /phpMyAdmin {
            root {{ phpMyAdmin_dest }};
            index index.php index.html index.htm;
            location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
            }
        }
```

We will look at how we can define *{{ phpMyAdmin_dest }}* and *{{ web_root }}* later. *app.local* is a directory which we will be creating using a role that we define ourselves.

That completes the setup for Nginx.

## M for MariaDB

[MariaDB](https://mariadb.org/) is a drop in replacement for MySQL. It is a popular database solution ever since MySQL was brought under Orcale's domain. We will use another role and variables to setup our database with a root and DB user.

We'll use the following command to install [*geerlingguy.mysql*](https://github.com/geerlingguy/ansible-role-mysql). It says mysql but it actually installs MariaDB. It has the option to install MySQL if needed.

```shell
ansible-galaxy install geerlingguy.mysql
```

Now let's modify our playbook file from above:

```python
---
- hosts: testservers
  become: true
  become_method: sudo
  vars_files:
    - vars/local.yml
  roles:
    - geerlingguy.repo-epel
    - geerlingguy.repo-remi
    - geerlingguy.nginx
    - geerlingguy.mysql
```

To configure the DB we need to define some variables in **vars/local.yml** file again. Our vars file should now look like the following:

```python
mysql_root_password: "super secret root password"
mysql_databases:
  - name: "appDbName"
mysql_users:
  - name: "appDbUser"
    password: "super secet user password"
    priv: "appDbName.*:ALL"

nginx_yum_repo_enabled: true
nginx_vhosts:
  - listen: "80"
    server_name: "app.local"
    root: "{{ web_root }}"
    index: "index.php index.html index.htm"
    state: "present"
    filename: "app.local.conf"
    extra_parameters: |
        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index app.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            try_files $uri =404;
        }
        location /phpMyAdmin {
            root {{ phpMyAdmin_dest }};
            index index.php index.html index.htm;
            location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
            }
        }
```

The above variables will configure our root password, database name, and database user.

## P is PHP

To install PHP we are going to be using [geerlingguy.php](https://github.com/geerlingguy/ansible-role-php). To install PHP 7.1 on our Vagrant box we will be making use of **geerlingguy.repo-remi** role that we added earlier. We can use it simply by adding to our vars file again.

```python
mysql_root_password: "super secret root password"
mysql_databases:
  - name: "appDbName"
mysql_users:
  - name: "appDbUser"
    password: "super secet user password"
    priv: "appDbName.*:ALL"

nginx_yum_repo_enabled: true
nginx_vhosts:
  - listen: "80"
    server_name: "app.local"
    root: "{{ web_root }}"
    index: "index.php index.html index.htm"
    state: "present"
    filename: "app.local.conf"
    extra_parameters: |
        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index app.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            try_files $uri =404;
        }
        location /phpMyAdmin {
            root {{ phpMyAdmin_dest }};
            index index.php index.html index.htm;
            location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
            }
        }

php_webserver_daemon: "nginx"
php_enablerepo: "remi-php71,epel"
php_packages_state: "latest"
php_enable_php_fpm: true
php_packages:
  - php
  - php-common
  - php-fpm
  - php-opcache
  - php-cli
  - php-mysql
  - php-gd
  - php-mbstring
  - php-mcrypt
  - php-pdo
  - php-pear
  - php-xml
  - php-apcu
  - php-pecl-apcu
  - phpmyadmin
```

The PHP varialbes that we added tell role which webserver we are using, which repos to enable, which version of the packages should be installed, whether to use PHP FPM, and which packages to install specifically. Notice that we are also installing **phpmyadmin**.

Our playbook now looks like this:

```python
---
- hosts: testservers
  become: true
  become_method: sudo
  vars_files:
    - vars/local.yml
  roles:
    - geerlingguy.repo-epel
    - geerlingguy.repo-remi
    - geerlingguy.nginx
    - geerlingguy.mysql
    - geerlingguy.php
```

## Custom Role

We will now create a custom role that will copy our source code from our speficied directory and place in the right directory on our web server. We can use the following command to create a role skeleton:

```shell
ansible-galaxy init --init-path roles/ app.source
```

Ansible galaxy will then create a directory structure for a role.

```shell
.
|-- README.md
|-- defaults
|   `-- main.yml
|-- files
|-- handlers
|   `-- main.yml
|-- meta
|   `-- main.yml
|-- tasks
|   `-- main.yml
|-- templates
|-- tests
|   |-- inventory
|   `-- test.yml
`-- vars
    `-- main.yml
```

We will mainly be working tasks, defaults, and the files directories.

Let's put a simple file called *app.php* in the *files* directory.

```php
<?php

phpinfo();
```

We will now add to the *main.yml* file in the *defaults* directory. This is where *web_root* from the previous section is defined.

```python
web_root: /var/www/app
```

