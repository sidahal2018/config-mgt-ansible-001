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

Below steps shows project-13 solution documentation

### Prepare Remote Source Repository Gitlab or GitHub ###
1. Login to your Gitlab account.
2. Create a new repository and name it "pbl".

### Setting up Infrastructure ###

1. Login into your GCP account.
2. Create Centos8 Control Machine and other target hosts such as webserver, database,Load Balancer, Jenkins and so on.
3. SSH into the control machine 
4. Create directory named `Ansible` to store all our ansible work
5. Installing ansible on control machine.
```
   sudo yum epel-release
   sudo yum install ansible
```

6. Installing Ansible with pip

Ansible can be installed with pip, the Python package manager. If pip isn’t already available on your system of Python, run the following commands to install it:

```
   curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
   python get-pip.py --user

```


7. Upgrading python to 2.9 to 2.10.14

If you have Ansible 2.9 or older installed, you need to use pip uninstall ansible first to remove older versions of Ansible before re-installing it.

8. First checking pip and uninstalling ansible
 ```
   which pip
   pip3 uninstall ansible
```

9. Installing ansible with pip:

 ```
   pip3 install ansible
  
```
10. Clone the gitlab url

#### To connect to Remote servers ####
Generate ssh keys and copy it to remote servers
* `ssh-keygen`
* `ssh-copy-id -i .ssh/id_rsa.pub root@54.147.121.140`

#### Upon Permission denied issue, perform the follwoing on remote servers ####

* `sudo su`
* `passwd`
* `sudo nano /etc/ssh/sshd_config`
 ```
   open port22
   permitRootlogin yes
   password authention yes
   
```
* `systemctl restart sshd`
* `systemctl status sshd`

Ansible uses a configuration file to customize settings. The file is `ansible.cfg` file. It can be located anywhere. We just need to export an environmetal variable to let Ansible know where to find this file.  Create a new file and name it ansible.cfg . Update it with the below content
```
 [defaults]
timeout = 160
roles_path =/home/sidahal2018/Ansible/roles
callback_whitelist = profile_tasks
log_path=~/ansible.log
host_key_checking = False
gathering = smart

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ControlPath=/tmp/ansible-ssh-%h-%p-%r -o ServerAliveInterval=60 -o ServerAliveCountMax=60

```

we specify the full path to your roles. If you do not do this, any role you download through galaxy will be installed in the default settings which could either be in `/etc/ansible/ansible.cfg or ~/.ansible.cfg`

Run the export command on the terminal to let Ansible know where to find the configuration file. export ANSIBLE_CONFIG=<FULL PATH TO YOUR ansible.cfg File>

Exporting my config file only persists for one session, typically a shell script is used in a work environment.
Export environment variable everytime or we can put in the bash_rc `~/.bash_rc`

`export ANSIBLE_CONFIG=/home/sidahal2018/Ansible/ansible.cfg`

We can search environment using the below command 
 `env | grep ANSIBLE`   


## Introducing Mysql Ansible Role ##

Here I am going to implement mysql role downloaded ansible galaxy

run the command `ansible-galaxy install geerlingguy.mysql`. 
Once downloaded, rename the folder to `mysql`
`sudo mv geerlingguy.mysql mysql`

Implementing mysql role for tooling website:
MySQL role is going to install and configure MySQL database on the Target host.

1. Install MySQL and other packages.
2. Start the MySQL service and enable it to start at boot.
3. Set the MySQL  password.
4. Create a database for tooling.
5. Create a database user for tooling.

On mysql role, using `defaults/main.yml` to define the variable, add the following :
```
  tooling_db_username: "siki"
  tooling_db_password: "siki"
  tooling_db_name: "tooling_db"

```
We are going to template the `tooling_db.sql file` for mysql becasue we need to use it to load the data initial data. However we dont need to put any variable inside the file since it will load directly as it is on the script
We are going to use the ansible mysql module for the create and insert statements

```
   We create new directory called files.Inside the files directory we reate tooling_db.sql 
   file copy and paste the tooling_db.sql scripts

```

On our mysql roles, under tasks folder create `load-mysql.yml` file and add the following tasks:


```

  ---
- name: Creating MySQL user for toooling website
  mysql_user:
    name: "{{ tooling_db_username }}"
    password: "{{ tooling_db_password }}"
    priv: "{{ tooling_db_name }}.*:ALL"
    state: present

- name: Creatinng new database
  mysql_db:
    name: "{{ tooling_db_name }}"
    state: present

- name: Create target directory
  file: 
   path: /temp
   state: directory 
   mode: 0755

- name: copy the sql file onto the server
  copy:
   src: tooling-db.sql
   dest: /temp/tooling-db.sql

- name: Restoring DB
  mysql_db:
    name: "{{ tooling_db_name }}"
    login_user: "{{ tooling_db_username }}"
    login_password: "{{ tooling_db_password }}"
    state: import
    target: /temp/tooling-db.sql
  tags:
    - restore_db

```

Verifing by connecting to mysql Database
Now logining to the credentials we have created

`mysql -u siki -p`

Enter password: 
For remote login we use the following:

`mysql -h 35.232.163.181 -u siki -p`


![](./images/mysql-query.PNG)
We are successfuly able to loginto the user name and password we have created

## Introducing Nginx Ansible Role ##

Install the nginx packages.

Start the nginx service and enable it to start at boot.

Copy the  Nginx virtual host configuration template file from the Ansible control machine to the Ansible Target host.

Configure letsencrypt as part of nginx role

On nginx role, using `defaults/main.yml` to define the variable, add the following :
```
 
  defaults file for nginx
  certbot_site_names: "sikisharm.ml"
  server_name: "sikisharm.ml"
  tooling_root_dir: "/var/www/html/tooling/html"
  certbot_package: "python-certbot-nginx"
  certbot_plugin: "nginx"
  certbot_mail_address: sidahal@gmail.com
  enable_nginx_lb: false
  load_balancer_is_required: false

```
on the handlers main.yml 

```
   ---
   # handlers file for nginx
   - name: restart nginx
    service: name=nginx state=restarted

   - name: start nginx
    service: name=nginx state=started



````
on the tasks folder we have the follwing YAML files
 
auto-RenewalCron.yml
configure_nginx.yml
install-packages.yml 
setup-ssl.yml
main.yml
 
 `auto-RenewalCron.yml`
  ```
    `---
     - name: Set Letsencrypt Cronjob for Certificate Auto Renewal
      cron: name=letsencrypt_renewal special_time=monthly job="/usr/bin/certbot renew"
      when: ansible_facts['os_family'] == "RedHat"

  ```


  `configure_nginx.yml`
 ```
  ---
- name: clone tooling website from github
  git:
   repo: https://github.com/darey-io/tooling.git
   dest: /var/www/html/tooling
   clone: yes
   force: yes

- name: Creating sites-available directory on host for Nginx
  file:
    path: /etc/nginx/{{ item }}
    state: directory
    mode: '0755'
  with_items:
   - sites-available
   - sites-enabled


- name: Deploy nginx configuation file
  template:
     src: nginx-tooling.j2
     dest: "/etc/nginx/nginx.conf"
     force: yes
  notify:
   - restart nginx

- name: Deploy nginx configuation file
  template:
     src: nginx-configuration.j2
     dest: "/etc/nginx/sites-available/{{ server_name }}.conf"
     force: yes
  notify:
   - restart nginx

- name: Enable tooling website
  file:
    src: "/etc/nginx/sites-available/{{ server_name }}.conf"
    dest: "/etc/nginx/sites-enabled/{{ server_name }}.conf"
    state: link
    force: yes
  notify:
   - restart nginx

- name: de-activate default nginx 
  file:
    path: /usr/share/nginx/html/index.html
    mode: '0755'
    state: absent
  notify:
   - restart nginx


- name: Add enabled Nginx site to /etc/hosts
  lineinfile:
    dest: /etc/hosts
    regexp: "127.0.0.1"
    line: "18.234.60.170 {{ server_name }}"
  notify:
   - restart nginx

 ```


`install-packages.yml`
 
    ---
    - name: install necessary packages
      package: name={{item}} update_cache=yes state=present
      with_items:
      - epel-release
      - nginx
      - git
      - php
      - php-gd
      - php-mysqli
      notify:
      - start nginx 


`setup-ssl.yml`

    ---
    - name: Install Python Package
      yum: name=python3 update_cache=yes state=latest

     #   - name: Enable EPEL Repository on CentOS 8
     #   dnf: name=epel-release update_cache=yes state=latest

     # - name: install certbot
     #   yum: name=certbot update_cache=yes state=present

     - name : Install Let's Encrypt Package
     yum: name={{ certbot_package }} update_cache=yes state=latest
 
     # # free-form (string) arguments, some arguments on separate lines with the 'args' keyword
      # # 'args' is a task keyword, passed at the same level as the module
      # - name: Run command if /path/to/database does not exist (with 'args' keyword)
      #   command: /usr/bin/make_database.sh db_user db_name
      #   args:
      #     creates: /path/to/database


      - name: Create and Install Cert 
      command: "certbot --{{ certbot_plugin }} -d  {{ server_name }} -m {{ certbot_mail_address }} --agree-tos    --    noninteractive --redirect"
      #   # args:
      #   #   creates: /path/to/database


`main.yml`

    ---
    # tasks file for nginx
    - include_tasks: install-packages.yml
    - include_tasks: configure_nginx.yml
    - include_tasks: setup-ssl.yml
      when: ansible_os_family == 'RedHat'
    - include_tasks: auto-RenewalCron.yml


Nginx configuration file should like the below :


      server {
        server_name  sikisharm.ml;
        root         /var/www/html/tooling/html;
        index        login.php index.htm;
        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        location / {
        }
        error_page 404 /404.html;
            location = /40x.html {
        }
        error_page 500 502 503 504 /50x.html;
      listen 443 ssl; # managed by Certbot
      ssl_certificate /etc/letsencrypt/live/sikisharm.ml/fullchain.pem; # managed by Certbot
      ssl_certificate_key /etc/letsencrypt/live/sikisharm.ml/privkey.pem; # managed by Certbot
      include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
      ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
      }
      server {
      if ($host = sikisharm.ml) {
        return 301 https://$host$request_uri;
      } # managed by Certbot
        listen       80;
        server_name  sikisharm.ml;
      return 404; # managed by Certbot
      }
                                                        

Let's verify nginx rendering the tooling website page correctly or not

Create/Update DNS Record
----------------------------------
I have domain from the freenom, I am going to update the DNS record. 
create an A/CNAME record for my domain- sikisharm.ml

Login into your  Freenom account.
Naviate to services and click on mydomain as shown below:

![](./images/Manage-domain.PNG)
![](./images/Manage-Dns.PNG)
![](./images/update-records.PNG)

Wait for some time to let the record propagate.
Check the DNS propagation using Nslookup 
`yum install -y bind-utils utility`

Run the playbooks
-------------------------------
`sudo ansible-playbook -i ../inventory/dev site.yml`

![](./images/runtheplaybook.PNG)

Verify Let’s Encrypt Certificate
-------------------------------
Verify the Let’s Encrypt certificate by visiting the HTTPS version of your website.

https://sikisharm.ml

You should now get an HTTPS version of your site.


![](./images/SSL-certificate.PNG)

## Introducing  Ansible Apache Role ## 
This role is downloaded from ansible glaxy from geerlingguy 
Apache role will install and configure Apache on the Target host. 
Start the Apache service and enable it to start at boot.
Copy the Apache virtual host configuration template file from the Ansible control machine to the Ansible Target host.

 `ansible-galaxy install geerlingguy.apache`
 `sudo mv geerlingguy.apache apache`

`configure-apache.yml`



        ---
        - name: clone tooling website from github
        git:
        repo: https://github.com/darey-io/tooling.git
        dest: /var/www/html/tooling
        clone: yes
        force: yes

        - name: Creating sites-available directory on host for httpd
        file:
        path: /etc/httpd/{{ item }}
        state: directory
        mode: '0755'
        with_items:
          - sites-available
          - sites-enabled


        - name: update main httpd configuation file
        template:
        src: centos-config.j2
        dest: "/etc/httpd/conf/httpd.conf"
        force: yes
        notify:
        - restart apache

        - name: Set up Apache virtuahHost
        template:
        src: tooling-config.j2
        dest: /etc/httpd/sites-available/{{ http_host }}.conf
   
        - name: Enable tooling website
        file:
        src: "/etc/httpd/sites-available/{{ http_host }}.conf"
        dest: "/etc/httpd/sites-enabled/{{ http_host }}.conf"
        state: link
        force: yes
        notify:
        - restart apache

        - name: de-activate default httpd page
        file:
        path: /etc/httpd/conf.d/welcome.conf
        mode: '0755'
        state: absent
        notify:
        - restart apache

        # - name: Add enabled Nginx site to /etc/hosts
        #   lineinfile:
        #     dest: /etc/hosts
        #     regexp: "127.0.0.1"
        #     line: "35.222.190.156 {{ server_name }}"

Templating the configuration file 
`centos-config.j2`

    ServerRoot "/etc/httpd"
    Listen 80
    Include conf.modules.d/*.conf
    User apache
    Group apache
    ServerAdmin root@localhost
      <Directory />
      AllowOverride none
      Require all denied
      </Directory>
      DocumentRoot "/var/www/html"

      <Directory "/var/www">
      AllowOverride None
      Require all granted
      </Directory>
      <Directory "/var/www/html">
      Options Indexes FollowSymLinks
      AllowOverride None
      Require all granted
      </Directory>
      <IfModule dir_module>
      DirectoryIndex index.html 
      </IfModule>
      <Files ".ht*">
      Require all denied
      </Files>
      ErrorLog "logs/error_log"
      LogLevel warn
      <IfModule log_config_module>
      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
      LogFormat "%h %l %u %t \"%r\" %>s %b" common
      <IfModule logio_module>
      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio
      </IfModule>
      CustomLog "logs/access_log" combined
      </IfModule>
      <IfModule alias_module>
      ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"
      </IfModule>

    <Directory "/var/www/cgi-bin">
    AllowOverride None
    Options None
    Require all granted
    </Directory>
    <IfModule mime_module>
    TypesConfig /etc/mime.types
    AddType application/x-compress .Z
    AddType application/x-gzip .gz .tgz
    AddType text/html .shtml
    AddOutputFilter INCLUDES .shtml

    </IfModule>
    AddDefaultCharset UTF-8

    <IfModule mime_magic_module>
     MIMEMagicFile conf/magic
    </IfModule>
    EnableSendfile on
    <IfModule mod_http2.c>
    Protocols h2 h2c http/1.1
    </IfModule>
    IncludeOptional conf.d/*.conf
    IncludeOptional sites-enabled/*.conf


`tooling-config.j2`

    <VirtualHost *:{{ http_port }}>
      ServerAdmin webmaster@{{ http_host }}
      ServerName {{ http_host }}
      ServerAlias www.{{ http_host }}/tooling/html
      DocumentRoot /var/www/{{ http_host }}/tooling/html
   

      <Directory /var/www/{{ http_host }}/tooling/html>
       Options -Indexes
      </Directory>

      <IfModule mod_dir.c>
       DirectoryIndex index.php index.html index.cgi index.pl  index.xhtml index.htm
      </IfModule>

    </VirtualHost>


## Indroducing Java and Jenkins Role ## 
Lets install Jenkins role from the Ansible community
Since Jenkins require JAVA to work, lets install Java Role first before we install Jenkins role

 `ansible-galaxy install geerlingguy.jenkins`
  `sudo mv geerlingguy.java java`
Similarly, we install jenkins role, Run the command below to install an Ansible Role for Jenkins

`ansible-galaxy install geerlingguy.jenkins`
`sudo mv geerlingguy.jenkins jenkins`
Now we have the Java and Jenkins roles are install, let import the jenkins.yml and java.yml file in the site.yml
update "site.yml" file and import playbook "jenkins.yml" and "java.yml" files, which will call for "jenkins" role and java role respectively

  on our Jenkins role: `deafault/main.yml`

  To install the list of plugins

  jenkins_plugins:
  - git
  - maven-plugin

  Alternatively:

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

 Lets check the jenkins installation on browser "public ip:8080" and verify `blue ocean` plugin installed
  To check Jenkins is installed or not
  `sudo service jenkins status/start/restart`
  `chkconfig jenkins on` >> enabled on reboot, jenkins will start automatically

 ![](./images/images_plugin_blueocean.PNG)

 ![](./images/pipeline-jenkins.PNG)


#### Indroducing Nginx as a load balancer ####

HAProxy is free, open source, highly available, load balancer software written by Willy Tarreau in 2000. 
It supports both Layer 4 (TCP) and Layer 7 (HTTP) based application load balancing
The HAproxy role is going to Installing and Configuring HAProxy Server on CentOS 8
  - Setting Up HAProxy Logging
  - Configuring HAProxy Front-end and Back-ends


  Creating  HAProxy-nginx role 
     on `handlers/main.yml`
          ---
          - name: start haproxy service
            service: name=haproxy state=started

          - name: lb restart
            service: name=haproxy state=restarted

          - name: restart rsyslog
            service: name=rsyslog state=restarted

on the `tasks/main.yml`

      - name: upadte /etc/hosts file for load balancer
      lineinfile:
      dest: /etc/hosts
      regexp: "127.0.0.1"
      line: "18.234.60.170 {{ server_name }}"
      notify:
      - restart nginx

      - name: upadte /etc/hosts file webserver
      lineinfile:
      dest: /etc/hosts
      regexp: "127.0.0.1"
      line: "18.234.60.170 {{ server_name }}"
      notify:
      - restart nginx
---------------------------------------------------------------
Thank you !!!
