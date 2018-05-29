---
title: "Automating Ansible SSH key setup"
date: 2017-10-25
draft: false
tags: ["ansible", "bash", "ssh"]
---
A small frustration with Ansible is the initial setup to procure ssh access to the remote hosts. 
Ansible is a great tool to automate all the things. However, it assumes you already have ssh access to the remote hosts.
Say your operations team member writes an Ansible deploy playbook to share with the team. 
It works for the operations team member locally, but other users won't be able to run the playbook until they also have ssh access to the remote hosts.
The goal of the following snippet is simple: install ssh keys into new hosts to rapidly enable Ansible playbook use.
If you're on a team, the benefits multiply because this script works for any Ansible inventory file and is easily distributed.
It uses a combination of Ansible and common ssh tools because they work well in tandem.

There are a few assumptions:

 - you don't have a better way to do this (i.e. AWS/GCP key management tools, prebaked keys in machine images, etc)
 - you have an Ansible inventory to parse
 - you already have access to all remote hosts with an ssh password (the sshpass tool may promote bad habits, but it's only used to install ssh keys)

```
#!/bin/bash

# Install your public key in remote hosts
# 
# Process:
#  - Parse an Ansible inventory file from stdin to obtain the host list
#  - Run ssh-keyscan on all hosts, adding output to $MY_KNOWN_HOSTS file
#  - Execute an Ansible playbook to install $MY_PUB_KEY in the remote host
#
# Requires Ansible, sshpass, and ssh-keyscan

# exit on error
set -e

##### VARS #####
MY_USERNAME=`whoami`
MY_PUB_KEY=`cat ~/.ssh/id_rsa.pub`
MY_KNOWN_HOSTS=~/.ssh/known_hosts
MY_VAULT_PW_FILE=~/.vault_password
################

if [ -z "$1" ]
then
    echo "Usage: $0 [path to Ansible inventory]"
    exit 99
fi

# Use Ansible to parse the inventory file to a string of hosts
HOSTS=`ansible all -i $1 --list-hosts --vault-password-file=$MY_VAULT_PW_FILE | sed '/hosts.*:/d; s/ //g'`

# Add each host key to known hosts
while read -r line; do
    ssh-keyscan -t ssh-rsa  $line >> $MY_KNOWN_HOSTS
done <<< "$HOSTS"

# Remove duplicates added from the keyscan
sort $MY_KNOWN_HOSTS | uniq > $MY_KNOWN_HOSTS.uniq
mv $MY_KNOWN_HOSTS{.uniq,}

# Kick off an Ansible playbook to copy your public key to the remote hosts
TMP_PLAYBOOK=$(mktemp) || { echo "Failed to create temp Ansible Playbook"; exit 1; }
cat <<EOF > $TMP_PLAYBOOK
---
- name: Install SSH key
  hosts: all
  tasks:
    - name: Add your authorized key on remote hosts
      authorized_key:
        user: "{{ my_username|trim }}"
        key: "{{ my_public_key|trim }}"
EOF

ansible-playbook -i $1 $TMP_PLAYBOOK --ask-pass --vault-password-file=$MY_VAULT_PW_FILE \
  -e "my_username=$MY_USERNAME" \
  -e "my_public_key=\"$MY_PUB_KEY\""

rm -f $TMP_PLAYBOOK
```

The script may be used as `install-ssh-keys.sh wordpress/production-inventory`

UPDATE: Since the original post date, I've since modified the parameters for `ssh-keyscan` above to include the `-t ssh-rsa` param.
By downloading all keys and sorting the known_hosts file, sometimes the wrong key was used first and would halt script executions.
I ended up clearing out my known_hosts file, applying the `-t ssh-rsa` limitation. Since the change this helper
script has been, well, helpful.