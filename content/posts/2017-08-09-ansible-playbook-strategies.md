---
title: "Ansible Playbook Strategies"
date: 2017-08-09
draft: false
tags: ["ansible", "mysql", "vault"]
---
Below I've outline four useful but less common Ansible strategies. My primary use cases for Ansible are application configuration, application deployment, and local development environments. The patterns aren't overly technical, nor do they rely on hacks. But they do represent a few unusual problems Ansible can help solve.

## Skip Module Parameters With Omit Filter

Say you have a playbook task that needs to accept multiple module parameters which are themselves incompatible. The omit filter allows you to control which parameters are passed to the module. The task below has one responsibility: write out an ssh deploy key to a known location. This task accepts input either as a string (the content param) or as a file (the src param). The Ansible copy module can operate with _either_ the src or the content param, but not both at the same time. Instead of duplicating this task and adding a when clause to handle different conditions, use a dictionary and the omit filter.  The omit filter allows you to configure conditional module parameters. 
    
    
    - name: Deliver SSH deployment key
      copy:
        content: "{{ (item.key == 'content')|ternary(item.value, omit) }}"
        src: "{{ (item.key == 'src')|ternary(item.value, omit) }}"
        dest: "{{ deploy_base_dir }}/deploy_key"
        mode: 0400
      with_dict:
        src: "{{ deploy_key_path }}"
        content: "{{ deploy_key_content_as_string }}"
      when: item.value|trim != ""

(Beware of subtle behaviors: the last dictionary item prevails if all facts are set.) This strategy lets you process multiple input values with one task. It keeps playbook sprawl under control and avoids bugs arising from duplication of tasks to handle conditional inputs. 

## Run a Set of Tasks One Time Only

Certain system commands should typically be run only one time. A common example of a run-once situation is the  mysql_secure_installation command following an installation of MySQL. Specifically, resetting a root MySQL password if one exists may cause issues. This strategy utilizes the /root/ home folder to track state with a sentinel file. For example, if you store the root MySQL password at /root/.my.cnf Ansible can detect the presence of that file and only run a  mysql_secure_installation  routine if needed (i.e. the file doesn't exist). 
    
    
    - name: Check for root MariaDB password existence
      stat: path=/root/.my.cnf
      register: mysql_root_password_file
    
    - include: "mysql_secure_installation.yml"
      when: mysql_root_password_file.stat.exists != True

and the include: 
    
    
    ---
    # mysql_secure_installation.yml
    #
    # Adapted from
    # https://github.com/PCextreme/ansible-role-mariadb/blob/master/tasks/mysql_secure_installation.yml
    #  
    # Note: This task generates a random root password if you don't provide a preferred password
    
    - name: Generate a new mysql root password
      set_fact: vault_mysql_root_password="{{ lookup('pipe', 'openssl rand -base64 12') }}"
      when: vault_mysql_root_password is undefined
    
    - name: Set root Password
      mysql_user: 
        name:     "root"
        host:     "{{ item }}"
        password: "{{ vault_mysql_root_password }}"
        state:    "present"
      with_items:
        - 127.0.0.1
        - ::1
        - localhost 
    
    - name: Add root .my.cnf
      template: src=root.my.cnf.j2 dest=/root/.my.cnf mode=0600
    
    - name: Reload privilege tables
      command: 'mysql -ne "{{ item }}"'
      with_items:
        - FLUSH PRIVILEGES
      changed_when: False
    
    - name: Remove anonymous users
      command: 'mysql -ne "{{ item }}"'
      with_items:
        - DELETE FROM mysql.user WHERE User=''
      changed_when: False
    
    - name: Disallow root login remotely
      command: 'mysql -ne "{{ item }}"'
      with_items:
        - DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')
      changed_when: False
    
    - name: Remove test database and access to it
      command: 'mysql -ne "{{ item }}"'
      with_items:
        - DROP DATABASE test
        - DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%'
      changed_when: False
      ignore_errors: True
    
    - name: Reload privilege tables
      command: 'mysql -ne "{{ item }}"'
      with_items:
        - FLUSH PRIVILEGES
      changed_when: False
    

## Run Command Only When Files Change

Sometimes you need to run a command only after new content arrives. A common Ansible strategy is to copy the files to the remote host and register a fact when the files changed. 
    
    
    - name: Install InCommon CA certs
      copy: content="{{ incommon_ca_certs }}" dest=/etc/pki/ca-trust/source/anchors/incommon-sha2.pem
      register: installed_new_ca_cert
    
    - name: Update CA trusted cert bundle
      command: update-ca-trust
      when: installed_new_ca_cert.changed

The example above references the CentOS7 command update-ca-trust used to rebuild a cache of trusted CA certificates. It only needs to run when new CA certificate files arrive, not on every Ansible run. 

## Turn Folders Into Symlinks

It is possible to use Ansible to manipulate filesystems in this manner, but you might want to question why you need to change folders into symlinks in the first place. For me, it was a requirement of a web application which shipped with a folder where developers were to place custom code. We opted to place a symlink to a git repository located elsewhere in another code repo. 
    
    
    - name: Find folders to replace with symlinks
      stat: path={{ item }}
      register: folder_state
      with_items:
        - "{{ project_repo_dir }}/lib/local"
    
    - name: Remove folders to replace with symlinks
      file: path={{ item.stat.path }} state=absent
      with_items: "{{ folder_state.results }}"
      when: item.stat.isdir is defined and item.stat.isdir
    
    - name: Create new symlinks
      file: src={{ item.src }} dest={{ item.dest }} state="link"
      with_items:
        - { src: "/path/to/my_local_code", dest: "{{ project_repo_dir }}/lib/local" }

The tasks above apply if any folders need to be converted to symlinks. Otherwise, they're skipped in subsequent playbook runs.