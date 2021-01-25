# Ansible Dynamic Assignments (Include) and Community Roles


We will introduce dynamic assignments using Ansible's `include` module

Now you may be wondering, what is the difference between static and dynamic assignments? And why do they matter?

Well, from [project12](https://professional-pbl.darey.io/en/latest/project12.html), you can tell already that static assignments makes use of the `import` Ansible module. The module that enables dynamic assignments is `include`

Hence,

```
import = Static
include = Dynamic
```


When the **import** module is used, all statements are pre-processed at the time playbooks are parsed. Meaning, when you execute `ansible-playbook site.yml`, Ansible will process all the playbooks referenced during the time it is parsing the statements. This also means that, during actual execution, if any statement changes, such statements will not be considered. Hence, it is static.

(***Parsing means analyzing the texts, syntax, strings in the file in order to understand what to do with it***). 

On the other hand, when **include** module is used, all statements are processed only during execution of the playbook. Meaning, after the statements are **parsed**, any changes to the statements encountered during execution will be used. 


Take note that it is always prefered to use static assignments for playbooks, because it is more reliable. With dynamic, it is hard to debug playbook problems due to its dynamic nature. However, you can use dynamic assignments for environment specific variables as we will be introducing in this project.

##### Introducing Dynamic Assignment Into Our structure

Create another folder and name it `dynamic-assignments`. Then inside that folder, create a new file and name it `env-vars.yml`. We will tell `site.yml` to `include` this playbook later. For now, lets keep building up the structure.


Your layout should now look like this.


```
├── dynamic-assignments
│   └── env-vars.yml
├── inventory
│   └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
├── static-assignments
│   └── common.yml
```


Since we will be using the same Ansible to configure multiple environments, and each of these environments will have certain unique attributes. such as **servername**, **ip-address** etc. we will need a way to set values to variables per specific environment.

For this reason, we will now create a folder to keep each environment's variables file. Therefore, create a new folder `env-vars`, then for each environment, create new **YAML** files which we will use to set variables.

Your layout should now look like this.

```
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
│   └── dev.yml
    └── stage.yml
    └── uat.yml
    └── prod.yml
├── inventory
│   └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
├── static-assignments
│   └── common.yml
    └── webservers.yml
```

Now paste the instruction below into the `env-vars.yml` file.

```
---
- name: collate variables from env specific file, if it exists
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ playbook_dir }}/../env_vars/{{ "{{ inventory_file }}.yml"
    - "{{ playbook_dir }}/../env_vars/default.yml"
  tags:
    - always
```

Notice 3 things here;

1. We used `include_vars` instead of `include`. That is because the developers of Ansible decided to separate the different possibilities on the module. From Ansible version 2.8, the `include` module will be deprecated and variants of `include` must be used. These are `include_tasks` `include_role` `include_vars`. In the same verin, **import** also has its variants such as the `import_playbook`, and `import_tasks` 
2. We made use of a [special variables](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html) `{{ playbook_dir }}` and `{{ inventory_file }}`. `{{ playbook_dir }}` will help Ansible to determine the location of the running playbook, and from there navigate to other path on the filesystem. `{{ inventory_file }}` on the other hand will dynamically resolve to the name of the inventory file being used, then append `.yml` so that it picks up the required file within the `env-vars` folder.
3. We are including the variables using a loop. `with_first_found` implies that, looping through the list of files, the first one found is used. This is good so that we can always set default values in case an environment specific env file does not exist.


### Update site.yml with dynamic assignments

Update the `site.yml` file to make use of the dynamic assignment. (*At this point, we cannot test it yet. We are just setting the stage for what is yet to come. So hang on to your hats*)


**site.yml** should now look like this.
```
---
- name: Include dynamic variables 
  hosts: all
  tasks:
    - import_playbook: ../static-assignments/common.yml 
    - include_playbook: ../dynamic-assignments/env-vars.yml
  tags:
    - always

- name: Webserver assignment
  hosts: webservers
    - import_playbook: ../static-assignments/webservers.yml

```

### Community Roles

It is time to develop a role for MySQL database. This role should install the database, and configure its users. But why should we re-invent the wheel? There are tons of roles that have already been developed by other open source engineers out there. These roles are actually production ready, and dynamic to accomodate most linux flavours or environment. With Ansible Galaxy again, we can simply download a ready to use ansible role, and keep going.

#### Downloading Mysql Ansible Role 

You can browse the available community roles [here](https://galaxy.ansible.com/home)

We will be using one developed by `geerlingguy`. Within your roles folder, run the command `ansible-galaxy install geerlingguy.mysql`. Once downloaded, rename the folder to `mysql`

Read the `README.md` file, and ensure that it configures the usernmae you require for the `tooling` website.


#### Other Roles

We need more roles for 

1. Nginx
2. Apache
3. Jenkins

With your experience on Ansible so far. 

- Decide if you want to develop your own role, or find an available one from the community
- Update both `static-assignment` and `site.yml` files to reflect all your work.
- Configure letsencrypt as part of nginx role. [Here is a guide you can take inspiration from](https://linuxbuz.com/linuxhowto/install-letsencrypt-ssl-ansible)

***MUST READ HINTS***:

- Ensure that you put condition to enable either **Nginx** or **Apache** load balancers. You cannot install both on the same machine. 
Follow the below guide to implement this use case

  - Declare a variable in `defaults/main.yml` file inside the Nginx and Apache roles. Name each variable whatever you like. Something like 
 `enable_nginx_lb` and `enable_apache_lb` respectively. 
  - Set both values to false like this `enable_nginx_lb: false`.
  - Declare another variable in both roles `load_balancer_is_required` and set its value to false as well
  - Update both assignment and site.yml files respectively

`loadbalancers.yml` file
```
- hosts: lb
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }

```

site.yml
```
     - name: Loadbalancers assignment
       hosts: lb
         - import_playbook: ../static-assignments/loadbalancers.yml
        when: load_balancer_is_required 
  ```

Now we will make use of env-vars to determine if we want to use loadbalancers in a certain environment. Assuming we only want to use nginx loadbalancer in `stage` and `prod` environments. While in `dev` and `uat`, we do not intend to use load balancers.

You will activate load balancer, and enable nginx by setting these in the respective environment's env-vars file. 

```
enable_nginx_lb: true
load_balancer_is_required: true
```

To test this, you will need to have another set of servers, update inventory for each environment, and run Ansible by specifying the respective environment. (If your laptop resources cannot accomodate more virtual servers, you can use AWS to create virtual servers in the cloud. [here is how to get AWS VMs](https://www.youtube.com/watch?v=xxKuB9kJoYM&list=PLtPuNR8I4TvkwU7Zu0l0G_uwtSUXLckvh&index=6)

# Lets get started with documenting Project13 Solution as Below:

### 1 - Prepare Remote Source Repository Gitlab or GitHub ###
1. Login to your Gitlab account.
2. Create a new repository and name it "pbl".

### 2 - Create VM  ###
Only on this server, Ansible will be installed.

1. Login into your GCP account.
2. Create Centos8 Control Machine
3. Login in to the control machine 
4. Create directoy to store all our Ansible file 
5. Installing Ansible with pip

Ansible can be installed with pip, the Python package manager. If pip isn’t already available on your system of Python, run the following commands to install it:

$ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
$ python get-pip.py --user

upgrading python to 2.9 to 2.10.14

If you have Ansible 2.9 or older installed, you need to use pip uninstall ansible first to remove older versions of Ansible before re-installing it.

first we check pip: which pip3
 pip3 uninstall ansible
 install: pip3 install ansible


To connect to Remote servers,
Gennerate ssh keys and copy it to remote servers
ssh-copy-id -i .ssh/id_rsa.pub root@54.147.121.140



[ Client Machines]
on nfs,jwebservers,db,and jenkins-master client machines:
* sudo su
* passwd
* sudo nano /etc/ssh/sshd_config
* open port22
* permitRootlogin yes
* password authention yes
* systemctl restart sshd
* systemctl status sshd

Ansible uses a configuration file to customize settings. The file is ansible.cfg file. It can be located anywhere. We just need to export an environmetal variable to let Ansible know where to find this file.

Create a new file and name it ansible.cfg . Update it with the below content, and specify the full path to your roles. If you do not do this, any Role you download through galaxy will be installed in the default settings which could either be in /etc/ansible/ansible.cfg or ~/.ansible.cfg

Run the export command on the terminal to let Ansible know where to find the configuration file. export ANSIBLE_CONFIG=<FULL PATH TO YOUR ansible.cfg File

exporting my config file only persists for one session, typically a shell script is used in a work environment.
Run the export command on the terminal to let Ansible know where to find the configuration file.

export ANSIBLE_CONFIG=<FULL PATH TO YOUR ansible.cfg File

export ANSIBLE_CONFIG=/home/sidahal2018/Ansible/ansible.cfg

 env | grep ANSIBLE    >> we can search environment
ANSIBLE_CONFIG=/home/siki/Ansible/ansible.cfg


## Indroducing Mysql Ansible Role 
Here I am going to implement mysql role downloaded ansibel galaxy
run the command `ansible-galaxy install geerlingguy.mysql`. Once downloaded, rename the folder to `mysql`
`sudo mv geerlingguy.mysql mysql`

Implementing mysql role for tooling website:
MySQL role is going to install and configure MySQL database on the Target host.

1. Install MySQL and other packages.

2. Start the MySQL service and enable it to start at boot.

3. Set the MySQL  password.
4. Create a database for tooling.
5 .Create a database user for tooling 
We are going to template the `tooling_db.sql file` for mysql becasue we need to use it to load the data initial data. However we dont need to put any variable inside the file since it will load directelcy as it is.
We are going to use the ansible mysql module for the create and insert module
#### Indroducing  Nginx Role 

### Indroducing  apache role Role 
Apache role to install and configure Apache on the Target host. This playbook will do the following things:

Install the Apache package.
Start the Apache service and enable it to start at boot.
Create an Apache web root directory.
Copy the Apache virtual host configuration template file from the Ansible control machine to the Ansible Target host.

# Indroducing Java and Jenkins Role 
Lets install Jenkins role from the Ansible community
Run the command below to install an Ansible Role for Jenkins
ansible-galaxy install geerlingguy.jenkins

Since Jenkins require JAVA to work, lets install Java Role first before we install Jenkins role
ansible-galaxy install geerlingguy.java
Lets rename to java >> sudo mv geerlingguy.java java
Similarly, we install jenkins role   ansible-galaxy install geerlingguy.jenkins
 
ansible-galaxy install geerlingguy.jenkins and rename it 


Now we have the Java and Jenkins roles are install, let import the jenkins.yml and java.yml file in the site.yml

Jenkins role:

deafault/main.yml

To install the list of plugins

jenkins_plugins:
  - git
  - maven-plugin
or

jenkins_plugins: [git, maven-plugin]

-----------------------------------------------------------------------------------------
To install a list of plugins, you may do this:

- name: Install Jenkins plugins
  jenkins_plugin:
    name: "{{ item }}"
    jenkins_home: "{{ jenkins_home_directory }}"
    url_username: "admin_username"
    url_password: "admin_password"
    state: present
    with_dependencies: yes
  with_items:
    - git
    - maven

### Indroducing Nginx as a load balancer


## Submitted the solution for review and feedback. Thank you ##
