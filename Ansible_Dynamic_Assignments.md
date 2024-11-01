# Ansible Dynamic Assignments (Include) and Community Roles

In this project we will introduce [dynamic assignments](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse.html#includes-dynamic-re-use) by using `include module`.
We will continue configuring our `UAT servers`, learn and practice new Ansible concepts and modules.

### EC2 Instances for this project

Ansible server
![](./images/1.png)

UAT Web server 1
![](./images/2.png)

UAT Web server 2
![](./images/3.png)

Load balancer server
![](./images/4.png)

Load balancer security group inbound rule
![](./images/5.png)

Database server
![](./images/6.png)


From [previous project](https://steghub.com/lessons/ansible-refactoring-static-assignments-imports-and-roles-101/), we can already tell that static assignments use `import` Ansible module. The module that enables dynamic assignments is `include`.

Hence,
```bash
import = Static
include = Dynamic
```

When the `import` module is used, all statements are pre-processed at the time playbooks are parsed. Meaning, when you execute `site.yml` playbook, Ansible will process all the playbooks referenced during the time it is parsing the statements. This also means that, during actual execution, if any statement changes, such statements will not be considered. Hence, it is `static`.
On the other hand, when `include` module is used, all statements are processed only during execution of the playbook. Meaning, after the statements are parsed, any changes to the statements encountered during execution will be used.

Take note that in most cases it is recommended to use `static assignments` for playbooks, because it is more reliable. With `dynamic` ones, it is hard to debug playbook problems due to its dynamic nature. However, you can use dynamic assignments for environment specific variables as we will be introducing in this project.


## Introducing Dynamic Assignment Into Our structure

In your `https://github.com/<your-name>/ansible-config-mgt` GitHub repository start a new branch and call it `dynamic-assignments`.

```bash
git checkout -b dynamic-assignments
```

Create a new folder, name it `dynamic-assignments`.
Then inside this folder, create a new file and name it `env-vars.yml`. We will instruct `site.yml` to `include` this playbook later. For now, let us keep building up the structure.

```bash
mkdir dynamic-assignments
touch dynamic-assignments/env-vars.yml
```
![](./images/7.png)

Your GitHub shall have following structure by now.

__Note__: Depending on what method you used in the previous project you may have or not have `roles` folder in your GitHub repository - if you used `ansible-galaxy`, then `roles` directory was only created on your `Jenkins-Ansible` server locally. It is recommended to have all the codes managed and tracked in GitHub, so you might want to recreate this structure manually in this case - it is up to you.

```css
├── dynamic-assignments
│   └── env-vars.yml
├── inventory
│   └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
└── roles (optional folder)
    └──...(optional subfolders & files)
└── static-assignments
    └── common.yml
```

Since we will be using the same Ansible to configure multiple environments, and each of these environments will have certain unique attributes, such as `servername`, `ip-address` etc., we will need a way to set values to variables per specific environment.

For this reason, we will now create a folder to keep each environment's variables file. Therefore, create a new folder `env-vars`, then for each environment, create new `YAML` files which we will use to set variables.

```bash
mkdir env-vars

touch env-vars/dev.yml env-vars/stage.yml env-vars/uat.yml env-vars/prod.yml
```
![](./images/8.png)

Your layout should now look like this.

```css
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
    └── dev.yml
    └── stage.yml
    └── uat.yml
    └── prod.yml
├── inventory
    └── dev
    └── stage
    └── uat
    └── prod
├── playbooks
    └── site.yml
└── static-assignments
    └── common.yml
    └── webservers.yml
```

Now paste the instruction below into the `env-vars.yml` file.

```yaml
---
- name: looping through list of available files
  include_vars: "{{ item }}"
  with_first_found:
    - files:
        - dev.yml
        - stage.yml
        - prod.yml
        - uat.yml
      paths:
        - "{{ playbook_dir }}/../env-vars"
  tags:
    - always
```
![](./images/9.png)

Notice 3 things to notice here:

1. We used `include_vars` syntax instead of `include`, this is because Ansible developers decided to separate different features of the module. From Ansible version 2.8, the `include` module is deprecated and variants of `include_*` must be used. These are:

- [include_role](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_role_module.html#include-role-module)
- [include_tasks](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_tasks_module.html#include-tasks-module)
- [include_vars](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_vars_module.html#include-vars-module)

In the same version, variants of `import` were also introduces, such as:

- [import_role](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/import_role_module.html#import-role-module)
- [import_tasks](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/import_tasks_module.html#import-tasks-module)

2. We made use of a [special variables](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html) `{{ playbook_dir }}` and `{{ inventory_file }}`. `{{ playbook_dir }}` will help Ansible to determine the location of the running playbook, and from there navigate to other path on the filesystem. `{{ inventory_file }}` on the other hand will dynamically resolve to the name of the inventory file being used, then append `.yml` so that it picks up the required file within the `env-vars` folder.

3. We are including the variables using a loop. `with_first_found` implies that, looping through the list of files, the first one found is used. This is good so that we can always set default values in case an environment specific env file does not exist.


## Update `site.yml` with dynamic assignments

Update `site.yml` file to make use of the dynamic assignment. (At this point, we cannot test it yet. We are just setting the stage for what is yet to come. So hang on to your hats)

`site.yml` should now look like this.

```yaml
---
- hosts: all
  name: Include dynamic variables
  become: yes
  tasks:
    - include_tasks: ../dynamic-assignments/env-vars.yml
      tags:
        - always

- import_playbook: ../static-assignments/common.yml

- import_playbook: ../static-assignments/uat-webservers.yml

- import_playbook: ../static-assignments/loadbalancers.yml
```
![](./images/10.png)

## Community Roles

Now it is time to create a role for `MySQL` database - it should install the `MySQL` package, create a database and configure users. But why should we re-invent the wheel? There are tons of roles that have already been developed by other open source engineers out there. These roles are actually production ready, and dynamic to accomodate most of Linux flavours. With Ansible Galaxy again, we can simply download a ready to use ansible role, and keep going.

## Download Mysql Ansible Role

You can browse available community roles [here](https://galaxy.ansible.com/ui/)
We will be using a [MySQL role developed by geerlingguy](https://galaxy.ansible.com/ui/standalone/roles/geerlingguy/mysql/).

__Hint__: To preserve your your GitHub in actual state after you install a new role - make a commit and push to master your `ansible-config-mgt` directory. Of course you must have `git` installed and configured on `Jenkins-Ansible` server and, for more convenient work with codes, you can configure `Visual Studio Code to work with this directory`. In this case, you will no longer need webhook and Jenkins jobs to update your codes on `Jenkins-Ansible` server, so you can disable it - we will be using Jenkins later for a better purpose.

### Configure vscode to work with the directory (`ansible-config-mgt`)

__Configure SSH for vscode__

HostName = Jenkins-Ansible Public IP Address

Click on `remote explorer` and Select `ansible-config-mgt`

![](./images/9-1.png)
![](./images/9-2.png)
![](./images/9-3.png)

On `Jenkins-Ansible` server make sure that `git` is installed with `git --version`, then go to `ansible-config-mgt` directory and run

```bash
git --version
git init
git pull https://github.com/<your-name>/ansible-config-mgt.git
git remote add origin https://github.com/<your-name>/ansible-config-mgt.git
git branch roles-feature
git switch roles-feature
```

![](./images/12.png)

![](./images/13.png)

### Inside `roles` directory create your new `MySQL role` with `ansible-galaxy` install `geerlingguy.mysql`

```bash
ansible-galaxy role install geerlingguy.mysql
```
![](./images/14.png)

![](./images/15.png)

__Rename the folder to `mysql`__

```bash
mv geerlingguy.mysql/ mysql
```
```bash
mv /home/ubuntu/.ansible/roles/mysql/ /home/ubuntu/ansible-config-mgt/roles #its depend on downloaded location for move command.
```
![](./images/16.png)
![](./images/17.png)

Read README.md file, and edit roles configuration to use correct credentials for MySQL required for the tooling website.

### Create Database and mysql user (`roles/mysql/vars/main.yml`)

```yaml
mysql_root_password: ""
mysql_databases:
  - name: tooling
    encoding: utf8
    collation: utf8_general_ci
mysql_users:
  - name: webaccess
    host: "172.31.32.0/20" # Webserver subnet cidr
    password: ZakirCmt$
    priv: "tooling.*:ALL"
```
![](./images/18.png)

### Create a new playbook inside `static-assignments` folder and name it `db-servers.yml` , update it with `mysql` roles.

```yaml
- hosts: db
  become: yes
  vars_files:
    - vars/main.yml
  roles:
    - { role: mysql }
```
![](./images/19.png)

### Now it is time to upload the changes into your GitHub:

```bash
git add .
git commit -m "Commit new role files into GitHub"
git push --set-upstream origin roles-feature
```

![](./images/20-1.png)
![](./images/20-2.png)

# Load Balancer roles

We want to be able to choose which Load Balancer to use, `Nginx` or `Apache`, so we need to have two roles respectively:

1. Nginx
2. Apache

With your experience on Ansible so far you can:

- Decide if you want to develop your own roles, or find available ones from the community

### Using the Community

![](./images/22.png)


```bash
ansible-galaxy role install geerlingguy.nginx

ansible-galaxy role install geerlingguy.apache
```
![](./images/23.png)

### Rename the installed Nginx and Apache roles

```bash
mv geerlingguy.nginx nginx

mv geerlingguy.apache apache
```
![](./images/24.png)

### The folder structure now looks like this

![](./images/25.png)

- ### Update both static-assignment and site.yml files to refer the roles

__Important Hints:__

- Since you cannot use both `Nginx` and `Apache` load balancer, you need to add a condition to enable either one - this is where you can make use of variables.
- Declare a variable in `defaults/main.yml` file inside the `Nginx` and `Apache` roles. Name each variables `enable_nginx_lb` and `enable_apache_lb` respectively.
- Set both values to `false` like this `enable_nginx_lb: false` and `enable_apache_lb: false`.
- Declare another variable in both roles `load_balancer_is_required` and set its value to `false` as well

### For nginx

```yaml
# roles/nginx/defaults/main.yml
enable_nginx_lb: false
load_balancer_is_required: false
```
![](./images/26.png)

### For apache

```yaml
# roles/apache/defaults/main.yml
enable_apache_lb: false
load_balancer_is_required: false
```
![](./images/27.png)

### Update assignment

`loadbalancers.yml` file

```yaml
---
- hosts: lb
  become: yes
  roles:
    - role: nginx
      when: enable_nginx_lb | bool and load_balancer_is_required | bool
    - role: apache
      when: enable_apache_lb | bool and load_balancer_is_required | bool
```

![](./images/29.png)

- ### Update `site.yml` files respectively

```yaml
---
- hosts: all
  name: Include dynamic variables
  become: yes
  tasks:
    - include_tasks: ../dynamic-assignments/env-vars.yml
      tags:
        - always

- import_playbook: ../static-assignments/common.yml

- import_playbook: ../static-assignments/uat-webservers.yml

- import_playbook: ../static-assignments/loadbalancers.yml

- import_playbook: ../static-assignments/db-servers.yml
```

![](./images/30.png)


Now you can make use of `env-vars\uat.yml` file to define which `loadbalancer` to use in UAT environment by setting respective environmental variable to `true`.

You will activate `load balancer`, and enable `nginx` by setting these in the respective environment's `env-vars` file.

### Enable Nginx

```yaml
enable_nginx_lb: true
load_balancer_is_required: true
```
![](./images/31.png)


# Set up for Nginx Load Balancer

### Update `roles/nginx/defaults/main.yml`

__Configure Nginx virtual host__

```yaml
---
nginx_vhosts:
  - listen: "80"
    server_name: "example.com"
    root: "/var/www/html"
    index: "index.php index.html index.htm"
    locations:
              - path: "/"
                proxy_pass: "http://myapp1"

    # Properties that are only added if defined:
    server_name_redirect: "www.example.com"
    error_page: ""
    access_log: ""
    error_log: ""
    extra_parameters: ""
    template: "{{ nginx_vhost_template }}"
    state: "present"

nginx_upstreams:
- name: myapp1
  strategy: "ip_hash"
  keepalive: 16
  servers:
    - "172.31.40.134 weight=5"
    - "172.31.42.151 weight=5"

nginx_log_format: |-
  '$remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"'
become: yes
```
![](./images/32.png)


### Update `roles/nginx/templates/nginx.conf.j2`

Comment the line `include {{ nginx_vhost_path }}/*;`



This line renders the `/etc/nginx/sites-enabled/` to the `http` configuration of Nginx.

#### Create a `server block` template in `Nginx.conf.j2` for nginx configuration file to override the default in nginx role.

```jinja
{% for vhost in nginx_vhosts %}
    server {
        listen {{ vhost.listen }};
        server_name {{ vhost.server_name }};
        root {{ vhost.root }};
        index {{ vhost.index }};

    {% for location in vhost.locations %}
        location {{ location.path }} {
            proxy_pass {{ location.proxy_pass }};
        }
    {% endfor %}
  }
{% endfor %}
```
![](./images/33.png)


### Update `inventory/uat`

```yaml
[lb]
load_balancer ansible_host=172.31.6.105 ansible_ssh_user='ubuntu'

[uat_webservers]
Web1 ansible_host=172.31.35.223 ansible_ssh_user='ec2-user'
Web2 ansible_host=172.31.34.101 ansible_ssh_user='ec2-user'

[db_servers]
db ansible_host=172.31.2.161 ansible_ssh_user='ubuntu'
```
![](./images/34.png)

### Update Webservers Role in `roles/webservers/tasks/main.yml` to install Epel, Remi's repoeitory, Apache, PHP and clone the tooling website from your GitHub repository

```yaml
---
- name: install apache
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "httpd"
    state: present

- name: Enable apache
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo systemctl enable httpd

- name: Install EPEL release
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y

- name: Install dnf-utils and Remi repository
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm -y

- name: Reset PHP module
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf module reset php -y

- name: Enable PHP 8.2 module
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf module enable php:remi-8.2 -y

- name: Install PHP and extensions
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name:
      - php
      - php-opcache
      - php-gd
      - php-curl
      - php-mysqlnd
    state: present

- name: Install MySQL client
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "mysql"
    state: present

- name: Start PHP-FPM service
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.service:
    name: php-fpm
    state: started

- name: Enable PHP-FPM service
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.service:
    name: php-fpm
    enabled: true

- name: Set SELinux policies for web servers
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo setsebool -P httpd_execmem 1
    cmd: sudo setsebool -P httpd_can_network_connect=1
    cmd: sudo setsebool -P httpd_can_network_connect_db=1

- name: install git
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "git"
    state: present

- name: clone a repo
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.git:
    repo: https://github.com/zakircmt/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  remote_user: ec2-user
  become: true
  become_user: root
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
```
![](./images/35-1.png)
![](./images/35-2.png)
![](./images/35-3.png)
![](./images/35-4.png)


### Update `roles/nginx/tasks/main.yml` with the code below to create a task that check and stop apache if it is running

```yaml
---
- name: Check if Apache is running
  ansible.builtin.service_facts:

- name: Stop and disable Apache if it is running
  ansible.builtin.service:
    name: apache2
    state: stopped
    enabled: no
  when: "'apache2' in services and services['apache2'].state == 'running'"
  become: yes
```
![](./images/36.png)

### Now run the playbook against your uat inventory

```bash
export ANSIBLE_CONFIG=/home/ubuntu/ansible-config-mgt/static-assignments/ansible.cfg
ansible-playbook -i inventory/uat playbooks/site.yml --extra-vars "@env-vars/uat.yml"
```
![](./images/37-1.png)
![](./images/37-2.png)
![](./images/37-3.png)
![](./images/37-4.png)
![](./images/37-5.png)
![](./images/37-6.png)
![](./images/37-7.png)


### Confirm that Nginx is enabled and running and Apache is disabled

__Access load balancer server__

```bash
ssh -A ubuntu@172.31.44.45
```
![](./images/39.png)

### Check Nginx and Apache status

![](./images/40.png)

### Check ansible configuration for Nginx

```bash
sudo vi /etc/nginx/nginx.conf
```
![](./images/41.png)


### Update the website's configuration with the database and user credentials to connect to the database (in /var/www/html/function.php file)

```bash
sudo vi /var/www/html/functions.php
```
![](./images/42.png)

### Apply tooling-db.sql command on the webservers

```bash

sudo mysql -h 172.31.46.99 -u webaccess -p tooling < tooling-db.sql
```

### Access the database server from Web Server

```bash
sudo mysql -h 172.31.46.99 -u webaccess -p
```

### Create in MyQSL a new admin user with username: myuser and password: password

```SQL
INSERT INTO users(id, username, password, email, user_type, status) VALUES (2, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
```

### Access the tooling website using the LB's Public IP address on a browser

![](./images/43.png)
![](./images/44.png)
![](./images/45.png)

The same must work with apache LB, so you can switch it by setting respective environmental variable to true and other to false.

To test this, you can update inventory for each environment and run Ansible against each environment.

## Git Push for nginx LB update the changes into your GitHub:
```
git add .
git commit -m "full deploy with nginx loadbalancer"
git push --set-upstream origin roles-feature
```
![](./images/git-push-nginx.png)


# Set up for Apache Load Balancer

__Update `roles/apache/tasks/configure-Dedian.yml`__

Configure Apache virtual host

```yaml
---
- name: Add apache vhosts configuration.
  template:
    src: "{{ apache_vhosts_template }}"
    dest: "{{ apache_conf_path }}/sites-available/{{ apache_vhosts_filename }}"
    owner: root
    group: root
    mode: 0644
  notify: restart apache
  when: apache_create_vhosts | bool
  become: yes

- name: Enable Apache modules
  ansible.builtin.shell:
    cmd: "a2enmod {{ item }}"
  loop:
    - rewrite
    - proxy
    - proxy_balancer
    - proxy_http
    - headers
    - lbmethod_bytraffic
    - lbmethod_byrequests
  notify: restart apache
  become: yes

- name: Insert load balancer configuration into Apache virtual host
  ansible.builtin.blockinfile:
    path: /etc/apache2/sites-available/000-default.conf
    block: |
      <Proxy "balancer://mycluster">
        BalancerMember http://172.31.40.134:80
        BalancerMember http://172.31.42.151:80
        ProxySet lbmethod=byrequests
      </Proxy>
      ProxyPass "/" "balancer://mycluster/"
      ProxyPassReverse "/" "balancer://mycluster/"
    marker: "# {mark} ANSIBLE MANAGED BLOCK"
    insertbefore: "</VirtualHost>"
  notify: restart apache
  become: yes
```
![](./images/47.png)


### Enable Apache (in `env-vars/uat.yml`)

#### Switch Apache to true and nginx to false

![](./images/47-1.png)

__Update `roles/apache/tasks/main.yml` to create a task that check and stop nginx if it is running__

```yaml
---
- name: Check if nginx is running
  ansible.builtin.service_facts:

- name: Stop and disable nginx if it is running
  ansible.builtin.service:
    name: nginx
    state: stopped
    enabled: no
  when: "'nginx' in services and services['nginx'].state == 'running'"
  become: yes
```
![](./images/48.png)

### Comment out the references for the db-servers and uat-webservers playbook in site.yml file in order not to rerun the tasks. Only the reference to import the load balancer playbook should be left.

### Now run the playbook against the uat inventory

```bash
ansible-playbook -i inventory/uat playbooks/site.yml --extra-vars "@env-vars/uat.yml"
```
![](./images/49-1.png)
![](./images/49-2.png)
![](./images/49-3.png)
![](./images/49-4.png)
![](./images/49-5.png)
![](./images/49-6.png)
![](./images/49-7.png)
![](./images/49-8.png)


### Confirm that Apache is running and enabled and Nginx is disabled

Access the Load balancer

```bash
ssh -A ubuntu@172.31.44.45
```
![](./images/50.png)

__Check Apache and Nginx status__

![](./images/51.png)

### Check ansible configuration for Apache

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```
![](./images/52.png)

### Access the tooling website using the LB's Public IP address

![](./images/43.png)
![](./images/53-1.png)
![](./images/53-2.png)
![](./images/53-3.png)
![](./images/53-4.png)

## Git Push for apache LB update the changes into your GitHub:
```
git add .
git commit -m "full deploy with apache loadbalancer"
git push --set-upstream origin roles-feature
```
![](./images/git-push-apache.png)
### Conclusion

We have learned and practiced how to use Ansible configuration management tool to prepare UAT environment for Tooling web solution.

# Md. Zakir Hossen
## Linkedin: https://www.linkedin.com/in/zakircmt/
### Emal:- zakircmt1@gmail.com